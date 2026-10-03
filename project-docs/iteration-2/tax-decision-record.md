# Tax Decision Record — DR-1 to DR-3

**Owner:** Tharun Swaminathan
**Iteration:** 2 — Backend Development
**Date:** October 2, 2026
**Issue:** #45 — Close DR-1–3 and create Scrum notes template
**Status:** Approved for implementation planning
**Scope:** Tax behavior decisions only; no backend code in this document

## Purpose

Iteration 1 left several tax behaviors that had to be made explicit before backend implementation could begin. This record closes DR-1 through DR-3:

- DR-1 — net-dividend nullability
- DR-2 — currency handling
- DR-3 — same-date ordering tie-breaker

The decisions below are based on the completed Iteration 1 tax specifications and the current Ghostfolio activity model.

Tax coding remains gated until DR-4 through DR-6 are also closed on Oct 5.

## DR-1 — Net-Dividend Nullability

### Question

What should `netDividend` contain when the source dividend has no known withholding-tax value?

Relevant source states:

```text
withholdingTax = null
withholdingTax = 0
withholdingTax > 0
```

### Decision

`netDividend` is nullable.

```text
grossDividend = quantity × unitPrice
```

If:

```text
withholdingTax = null
```

then:

```text
netDividend = null
withholdingTaxRate = null
```

If:

```text
withholdingTax = 0
```

then:

```text
netDividend = grossDividend
withholdingTaxRate = 0
```

If:

```text
withholdingTax > 0
```

then:

```text
netDividend = grossDividend - withholdingTax
```

and, when `grossDividend > 0`:

```text
withholdingTaxRate = withholdingTax / grossDividend
```

### Meaning

The design distinguishes:

```text
null = unknown
0 = known zero
```

A historical or imported dividend with no withholding information must not be reported as though the system knows that no tax was withheld.

### Rationale

Treating unknown withholding as zero would create a false tax result.

Example:

```text
grossDividend = 100 USD
withholdingTax = null
```

Correct tax-reporting result:

```text
grossDividend = 100 USD
withholdingTax = null
netDividend = null
```

The gross dividend remains known even when the net value is not.

### Consequences

- `netDividend` must be nullable in tax-domain result interfaces.
- yearly net-dividend totals can become unavailable/incomplete when source withholding is unknown;
- exports must preserve the difference between unknown and zero;
- frontend code must not render `null` as `0`.

### Implementation Impact

The future `DividendRecord` keeps:

```text
grossDividend
withholdingTax
withholdingTaxRate
netDividend
```

as separate values.

Only the withholding amount is persisted. `netDividend` and `withholdingTaxRate` are derived.

The existing activity `fee` remains separate.

### Test Impact

At minimum, cover:

```text
gross = 100, withholding = 15 -> net = 85
gross = 100, withholding = 0  -> net = 100
gross = 100, withholding = null -> net = null, rate = null
```

Also verify that the rate calculation never divides by zero.

## DR-2 — Currency Handling

### Question

Which currency owns the stored withholding value, and how should tax-domain amounts behave when source, asset, and base currencies differ?

### Decision

The persisted `withholdingTax` amount uses the same transaction currency as the source activity.

Example:

```text
activity.currency = USD
withholdingTax = 15
```

means:

```text
15 USD withheld
```

The original stored amount is never overwritten by an asset-currency or base-currency conversion.

Tax calculations may additionally derive:

```text
transaction-currency value
asset-profile-currency value
base-currency value
```

when those conversions are available.

### Historical Conversion Rule

When a converted value is required, reuse Ghostfolio's existing historical currency-conversion path for the activity date.

Conceptually:

```text
source amount
+ source currency
+ target currency
+ activity date
-> converted amount
```

The tax feature does not maintain a separate exchange-rate database.

### Missing FX Rule

Missing historical FX data must remain explicit.

Do not convert missing FX to:

```text
0
```

and do not assume:

```text
1 source unit = 1 target unit
```

unless source and target currencies are actually the same.

If conversion is unavailable:

```text
converted value = null / unavailable
```

and the tax result carries an appropriate diagnostic, for example:

```text
MISSING_BASE_CURRENCY_CONVERSION
```

The original transaction-currency amount remains valid.

### Same-Currency Rule

If source and target currency are identical, pass the amount through unchanged. No exchange-rate lookup is required.

### Precision Rule

This decision does not close the rounding question.

Internal tax arithmetic remains high precision. Final rounding and serialization behavior is closed by DR-6 on Oct 5.

### Shared API Note

This decision defines tax currency semantics, not a tax-specific JSON number format.

The implementation must follow the team's shared API conventions rather than inventing a separate serialization rule.

### Consequences

- persisted withholding remains traceable to the source transaction;
- derived conversions never mutate source data;
- yearly portfolio totals can use the selected base currency;
- missing FX can make a base-currency result incomplete without destroying the known transaction-currency result.

### Implementation Impact

Normalized tax inputs should carry enough context to identify:

```text
transactionCurrency
assetCurrency
baseCurrency
activityDate
```

Conversion belongs in the shared tax normalization/calculation pipeline rather than being repeated independently by FIFO, average cost, yearly summary, CSV export, and PDF export.

### Test Impact

Cover:

- same-currency passthrough;
- transaction currency different from base currency;
- historical conversion using the activity date;
- missing historical conversion;
- multiple currencies;
- no silent zero fallback;
- no silent 1:1 fallback.

## DR-3 — Same-Date Ordering Tie-Breaker

### Question

If two tax-relevant activities have the same timestamp/date, what deterministic order should FIFO and average cost use?

### Decision

Tax activities are ordered by:

```text
1. activity.date ascending
2. activity.id ascending
```

The full stored activity timestamp is used first.

If two activities have the exact same timestamp, the activity ID is the deterministic secondary key.

### Rationale

Cost-basis calculations are sequence-sensitive.

A stable second key prevents results from changing because of database return order, runtime behavior, or fixture ordering.

This also aligns with the activity ordering pattern identified in the current Ghostfolio implementation during Iteration 1.

### Limitation

The ID tie-breaker provides deterministic software behavior. It is not a jurisdiction-specific legal ordering rule.

If the project later requires an explicit user-defined execution sequence, that would be a separate feature.

### Consequences

FIFO and average cost must consume the same normalized ordering.

Individual calculators must not implement different sorting rules.

Yearly summaries and exports consume calculated results instead of re-sorting source transactions.

### Implementation Impact

The common tax normalization step should expose activities already ordered by:

```text
date ASC
id ASC
```

### Test Impact

Cover:

- different dates;
- same calendar date with different timestamps;
- identical timestamps with different IDs;
- multiple BUYs at the same timestamp;
- BUY and SELL at the same timestamp;
- repeated execution producing identical results.

## Decision Summary

| Decision | Final Rule |
| --- | --- |
| DR-1 | Unknown withholding means `netDividend = null`; known zero withholding means `netDividend = grossDividend`. |
| DR-2 | Persist withholding in transaction currency; derive other currencies using historical conversion; missing FX stays unavailable with a diagnostic. |
| DR-3 | Deterministic activity order is `date ASC`, then `id ASC`. |

## Implementation Gate

The following decisions remain scheduled for Oct 5:

```text
DR-4 — fee treatment
DR-5 — average-cost pool scope
DR-6 — rounding / decimal-safe output rule
```

Do not implement tax behavior that depends on DR-4 through DR-6 before they are closed.

The first planned tax coding PR remains Oct 6.

## Related Iteration 1 Design Sources

```text
project-docs/iteration-1/tax/withholding-tax-schema.md
project-docs/iteration-1/tax/fifo-capital-gains-spec.md
project-docs/iteration-1/tax/average-cost-capital-gains-spec.md
project-docs/iteration-1/tax/tax-lot-data-model.md
project-docs/iteration-1/tax/yearly-tax-summary-data-structure.md
project-docs/iteration-1/tax/tax-design-architecture-report-section.md
```

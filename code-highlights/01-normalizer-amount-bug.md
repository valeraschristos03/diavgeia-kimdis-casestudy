# 01 — The amount-calculation bug in the normalizer

## Context

In Diavgeia the amount of a decision sits in a different place per decision type. The normalizer has an `if / else if` chain that tries the paths in priority order:

```
sponsor -> person -> amountWithKae -> amountWithVAT
```

The relevant raw field is `extraFieldValues` (abbreviated `efv`).

## Looking at the data first

Before any change, the raw JSON (9,349 decisions) was inspected. A sample `amountWithKae` line:

```json
{
  "kae": "0.64.98.95.0001",
  "amountWithVAT": 3542.45,
  "kaeCreditRemainder": 4071.41,
  "kaeBudgetRemainder": 4071.41
}
```

Here `amountWithVAT` is a number. By contrast the top-level `extraFieldValues.amountWithVAT` is an object:

```json
{ "amount": 3542.45, "currency": "EUR" }
```

Two fields with the same name and different shapes. That is where the problem was.

## Before

```js
} else if (efv && efv.amountWithKae && efv.amountWithKae.length > 0) {
    sponsor    = efv.amountWithKae[0].sponsorAFMName?.name ?? "unknown";
    sponsorAfm = efv.amountWithKae[0].sponsorAFMName?.afm  ?? "-";
    amount     = efv.amountWithKae[0].amountWithVAT?.amount ?? 0;
    //                                ^ reading .amount on a number -> undefined -> 0
}
```

Reading `.amount` on a number returns `undefined`, so the amount fell back to 0 with no error or warning.

## Extent (measured on the sample)

Which branch actually handled each of the 9,349 decisions:

| Branch | Count | Notes |
|---|---:|---|
| `sponsor` | 1,449 | worked correctly (`expenseAmount.amount`) |
| `person` | 6 | `awardAmount.amount`, usually null |
| `amountWithKae` | 1,646 | **all zeroed by the old code** |
| `amountWithVAT` | 805 | worked correctly |
| none | 5,443 | genuinely no amount |

```
amountWithKae decisions:            1,646
zeroed by the old code:             1,646 (100%)
real sum of those decisions:        15,678,275.72 EUR
```

## After

The logic was extracted into a pure function, shared by the normalizer and the aggregation pipeline (single source of truth):

```js
export function extractCounterparty(praksi) {
    let sponsor = "unknown", sponsorAfm = "-", amount = 0;
    const efv = praksi.extraFieldValues;
    if (!efv) return { sponsor, sponsorAfm, amount };

    if (efv.sponsor && efv.sponsor.length > 0) {
        sponsor    = efv.sponsor[0].sponsorAFMName?.name ?? "unknown";
        sponsorAfm = efv.sponsor[0].sponsorAFMName?.afm  ?? "-";
        amount     = efv.sponsor[0].expenseAmount?.amount ?? 0;

    } else if (efv.person && efv.person.length > 0) {
        sponsor    = efv.person[0]?.name ?? "unknown";
        sponsorAfm = efv.person[0]?.afm  ?? "-";
        amount     = efv.awardAmount?.amount ?? 0;

    } else if (efv.amountWithKae && efv.amountWithKae.length > 0) {
        // amountWithKae[i].amountWithVAT is a NUMBER, not { amount }.
        // The total is on the top-level efv.amountWithVAT.amount.
        sponsor    = efv.amountWithKae[0].sponsorAFMName?.name ?? "unknown";
        sponsorAfm = efv.amountWithKae[0].sponsorAFMName?.afm  ?? "-";
        amount =
            efv.amountWithVAT?.amount ??
            efv.amountWithKae.reduce((s, line) => s + (Number(line.amountWithVAT) || 0), 0);

    } else if (efv.amountWithVAT) {
        sponsor    = efv.donationReceiver?.[0]?.name ?? "unknown";
        sponsorAfm = efv.donationReceiver?.[0]?.afm  ?? "-";
        amount     = efv.amountWithVAT?.amount ?? 0;
    }

    return { sponsor, sponsorAfm, amount: Number(amount) || 0 };
}
```

Note the `Number(amount) || 0` at the exit: it guarantees a numeric result even if a source value is a numeric string or null.

## Verification

```
Total before:  6,383,282.20 EUR
Total after:   22,061,557.92 EUR
Difference:    15,678,275.72 EUR
```

These `amountWithKae` decisions carry no public counterparty AFM, so the unidentified counterparty (`sponsorAfm === "-"`) was excluded from the top-sponsors ranking while remaining in the totals — otherwise it would dominate the ranking with roughly 19.6M EUR of otherwise-legitimate but unattributed spend.

## Reproduce it yourself

With the dataset present, a few lines confirm the effect end to end:

```bash
cd backend
node --input-type=module -e "
  import { readFileSync } from 'fs';
  import { normalize } from './src/normalizer.mjs';
  import { calculateStats } from './src/statistics.mjs';
  const d = JSON.parse(readFileSync('./praxis_54446.json'));
  const s = calculateStats(d.map(normalize), '54446');
  console.log('sum:', s.sum.toFixed(2), 'withAmount:', s.withAmountLength);
"
# sum: 22061557.92 withAmount: 3363
```

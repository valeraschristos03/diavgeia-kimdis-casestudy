# 02 — Statistics via MongoDB aggregation

## Problem

`GET /stats/:afm` fetched every decision of a body into Node memory (up to ~45,000 for large bodies, an estimate), normalised, filtered and computed, on every call.

## Solution

The computation was moved into MongoDB with a `$facet` pipeline. The amount is computed inline with `$switch`, using the same logic as the normalizer's `extractCounterparty`.

### Amount computation

```js
const AMOUNT_EXPR = {
  $switch: {
    branches: [
      { case: hasItems("$extraFieldValues.sponsor"),
        then: { $ifNull: [{ $arrayElemAt: ["$extraFieldValues.sponsor.expenseAmount.amount", 0] }, 0] } },
      { case: hasItems("$extraFieldValues.person"),
        then: { $ifNull: ["$extraFieldValues.awardAmount.amount", 0] } },
      { case: hasItems("$extraFieldValues.amountWithKae"),
        then: { $ifNull: ["$extraFieldValues.amountWithVAT.amount", 0] } },
      { case: hasField("$extraFieldValues.amountWithVAT"),
        then: { $ifNull: ["$extraFieldValues.amountWithVAT.amount", 0] } },
    ],
    default: 0,
  },
};
```

The dotted array path `"$extraFieldValues.sponsor.expenseAmount.amount"` uses Mongo's implicit array traversal: because `sponsor` is an array, the path maps over it and yields an array, and `$arrayElemAt [ …, 0 ]` picks the first element.

### Handling non-uniform shapes

The first run failed:

```
MongoServerError: The argument to $size must be an array, but was of type: object
```

The raw fields are not always arrays. Measured on the sample:

```
sponsor       — array 1,450 | null 4,069
person        — array    33 | null   11 | bare object 15
amountWithKae — array 2,452 | null    3
```

`person` is sometimes a bare `{ afm, name }`. In JavaScript `efv.person.length` comes out `undefined` and the branch is skipped, but Mongo's `$size` throws on a non-array. Every `$size` is guarded with `$isArray`, so a non-array maps to size 0 (the same behaviour as `x && x.length > 0`):

```js
const arraySize = (path) => ({ $cond: [{ $isArray: path }, { $size: path }, 0] });
const hasItems  = (path) => ({ $gt: [arraySize(path), 0] });
const hasField  = (path) => ({ $ne: [{ $ifNull: [path, null] }, null] });
```

The same `$switch`/guard pattern computes the counterparty AFM and name (`CP_AFM_EXPR`, `CP_NAME_EXPR`) used for the top-sponsors grouping.

### Statistics with `$facet`

```js
{ $facet: {
    totalCount: [{ $count: "n" }],
    withAmount: [
      { $match: { _amt: { $ne: 0 } } },
      { $sort: { _amt: 1 } },
      { $group: { _id: null, sum: { $sum: "$_amt" }, count: { $sum: 1 },
                  min: { $min: "$_amt" }, max: { $max: "$_amt" }, arr: { $push: "$_amt" } } },
      { $project: { sum:1, count:1, min:1, max:1,
          average: { $cond: [{ $gt: ["$count", 0] }, { $divide: ["$sum", "$count"] }, 0] },
          median:  /* $let over count: middle element, or average of the two middle ones */ }},
    ],
    topSponsors: [
      { $match: { _amt: { $ne: 0 }, _cpAfm: { $nin: ["-", ownAfm] } } },
      { $group: { _id: "$_cpAfm", total: { $sum: "$_amt" }, name: { $first: "$_cpName" } } },
      { $sort: { total: -1 } }, { $limit: 10 },
    ],
}}
```

Pipeline shape: `$match (pre-filters) -> $addFields (_amt, _cpAfm, _cpName) -> $match (amount range / counterparty) -> $facet`. Filters map to stages as follows:

| Filter | Stage | Expression |
|---|---|---|
| `type` | pre `$match` | `decisionTypeId: type` |
| `keyword` | pre `$match` | `subject: { $regex: escapeRegex(keyword), $options: "i" }` |
| `startDate`/`endDate` | pre `$match` | `issueDate: { $gte, $lte }` (epoch ms) |
| `minAmount`/`maxAmount` | post `$match` | `_amt: { $gte, $lte }` (needs the computed amount) |
| `counterparty` | post `$match` | `$or: [{ _cpAfm }, { _cpName }]` |

The whole pipeline runs with `{ allowDiskUse: true }`.

## Result

Identical results to the in-memory path (see [05](05-testing-offline.md)), with no decisions transferred from the database into Node.

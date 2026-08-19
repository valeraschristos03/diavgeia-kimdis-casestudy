# 02 · Στατιστικά μέσω Mongo aggregation

## Το πρόβλημα

`GET /stats/:afm` έφερνε **όλες** τις πράξεις του φορέα στη μνήμη του Node (~45.000 για μεγάλους φορείς), τις `normalize`-άριζε, φιλτράριζε και υπολόγιζε — σε **κάθε** κλήση. Ακριβό και μη κλιμακώσιμο.

## Η λύση

Υπολογισμός **μέσα στη MongoDB** με ένα `$facet` pipeline. Το ποσό υπολογίζεται inline με `$switch`, με **ίδια ακριβώς λογική** με τη `extractCounterparty` του normalizer.

### 1. Το ποσό inline (η καρδιά)

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

### 2. Η παγίδα που έπιασε μόνο η πραγματική Mongo

Πρώτη εκτέλεση → crash:

```
MongoServerError: The argument to $size must be an array, but was of type: object
```

Γιατί; Τα raw πεδία **δεν είναι πάντα πίνακες**:

```
sponsor        →  array 1.450 |  null 4.069
person         →  array    33 |  null   11  |  ΣΚΕΤΟ object 15   ← ο ένοχος
amountWithKae   →  array 2.452 |  null    3
```

Το `person` μερικές φορές είναι σκέτο `{ afm, name }`. Στο JS το `efv.person.length` απλώς βγαίνει `undefined` (falsy) και ο κλάδος προσπερνιέται — αλλά το Mongo `$size` **σκάει**.

**Fix:** κάθε `$size` προστατεύεται με `$isArray`, ώστε non-array → μέγεθος 0 (ίδια συμπεριφορά με το `x && x.length > 0`):

```js
const arraySize = (path) => ({ $cond: [{ $isArray: path }, { $size: path }, 0] });
const hasItems  = (path) => ({ $gt: [arraySize(path), 0] });
const hasField  = (path) => ({ $ne: [{ $ifNull: [path, null] }, null] });
```

### 3. Στατιστικά με `$facet`

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
          median:  /* $let με βάση το count: μεσαίο ή μέσος δύο μεσαίων */ }},
    ],
    topSponsors: [
      { $match: { _amt: { $ne: 0 }, _cpAfm: { $nin: ["-", ownAfm] } } },
      { $group: { _id: "$_cpAfm", total: { $sum: "$_amt" }, name: { $first: "$_cpName" } } },
      { $sort: { total: -1 } }, { $limit: 10 },
    ],
}}
```

Τα φίλτρα (τύπος, ημερομηνία, ποσό, λέξη-κλειδί, αντισυμβαλλόμενος) μπαίνουν σε `$match` **πριν/μετά** τον υπολογισμό του `_amt` αναλόγως.

## Αποτέλεσμα

Ταυτόσημα νούμερα με την in-memory διαδρομή (δες [05](05-testing-offline.md)), χωρίς να φεύγει καμία πράξη από τη βάση προς το Node.

# 02 — Στατιστικά μέσω Mongo aggregation

## Πρόβλημα

Το `GET /stats/:afm` έφερνε όλες τις πράξεις του φορέα στη μνήμη του Node (έως περίπου 45.000 για μεγάλους φορείς), τις κανονικοποιούσε, φιλτράριζε και υπολόγιζε σε κάθε κλήση.

## Λύση

Ο υπολογισμός μεταφέρθηκε στη MongoDB με `$facet` pipeline. Το ποσό υπολογίζεται inline με `$switch`, με την ίδια λογική που εφαρμόζει η `extractCounterparty` του normalizer.

### Υπολογισμός ποσού

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

### Χειρισμός μη ομοιόμορφων σχημάτων

Η πρώτη εκτέλεση απέτυχε:

```
MongoServerError: The argument to $size must be an array, but was of type: object
```

Τα raw πεδία δεν είναι πάντοτε πίνακες:

```
sponsor       — array 1.450 | null 4.069
person        — array    33 | null   11 | object 15
amountWithKae — array 2.452 | null    3
```

Το `person` εμφανίζεται ενίοτε ως μεμονωμένο `{ afm, name }`. Στο JavaScript το `efv.person.length` προκύπτει `undefined` και ο κλάδος προσπερνιέται, αλλά ο τελεστής `$size` απαιτεί πίνακα. Κάθε `$size` προστατεύεται με `$isArray`, ώστε non-array να αντιστοιχεί σε μέγεθος 0 (ίδια συμπεριφορά με τον έλεγχο `x && x.length > 0`):

```js
const arraySize = (path) => ({ $cond: [{ $isArray: path }, { $size: path }, 0] });
const hasItems  = (path) => ({ $gt: [arraySize(path), 0] });
const hasField  = (path) => ({ $ne: [{ $ifNull: [path, null] }, null] });
```

### Στατιστικά με `$facet`

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
          median:  /* $let με βάση το count: μεσαίο στοιχείο ή μέσος των δύο μεσαίων */ }},
    ],
    topSponsors: [
      { $match: { _amt: { $ne: 0 }, _cpAfm: { $nin: ["-", ownAfm] } } },
      { $group: { _id: "$_cpAfm", total: { $sum: "$_amt" }, name: { $first: "$_cpName" } } },
      { $sort: { total: -1 } }, { $limit: 10 },
    ],
}}
```

Τα φίλτρα (τύπος, ημερομηνία, ποσό, λέξη-κλειδί, αντισυμβαλλόμενος) εφαρμόζονται με `$match` πριν ή μετά τον υπολογισμό του `_amt`, ανάλογα με το πεδίο.

## Αποτέλεσμα

Ταυτόσημα αποτελέσματα με την in-memory διαδρομή (βλ. [05](05-testing-offline.md)), χωρίς μεταφορά πράξεων από τη βάση στο Node.

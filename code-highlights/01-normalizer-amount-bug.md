# 01 — Σφάλμα υπολογισμού ποσού στον normalizer

## Πλαίσιο

Στη Διαύγεια το ποσό μιας πράξης βρίσκεται σε διαφορετικό σημείο ανά τύπο απόφασης. Ο normalizer έχει αλυσίδα `if/else if` που δοκιμάζει τα paths με σειρά προτεραιότητας:

```
sponsor → person → amountWithKae → amountWithVAT
```

## Ανάλυση των δεδομένων

Πριν από οποιαδήποτε αλλαγή, εξετάστηκε το raw JSON (9.349 πράξεις). Δείγμα μιας γραμμής `amountWithKae`:

```json
{
  "kae": "0.64.98.95.0001",
  "amountWithVAT": 3542.45,
  "kaeCreditRemainder": 4071.41,
  "kaeBudgetRemainder": 4071.41
}
```

Εδώ το `amountWithVAT` είναι αριθμός. Αντίθετα, το top-level `extraFieldValues.amountWithVAT` είναι αντικείμενο:

```json
{ "amount": 3542.45, "currency": "EUR" }
```

Δύο πεδία με το ίδιο όνομα (`amountWithVAT`) και διαφορετικό σχήμα. Εκεί βρισκόταν το πρόβλημα.

## Πριν

```js
} else if (efv && efv.amountWithKae && efv.amountWithKae.length > 0) {
    sponsor    = efv.amountWithKae[0].sponsorAFMName?.name ?? "άγνωστος";
    sponsorAfm = efv.amountWithKae[0].sponsorAFMName?.afm  ?? "-";
    amount     = efv.amountWithKae[0].amountWithVAT?.amount ?? 0;
}
```

Η ανάγνωση `.amount` επί αριθμού επιστρέφει `undefined`, οπότε το ποσό μηδενιζόταν χωρίς σφάλμα ή προειδοποίηση.

## Έκταση

```
Πράξεις τύπου amountWithKae:      1.646
Μηδενίζονταν με τον παλιό κώδικα: 1.646 (100%)
Πραγματικό άθροισμα:             15.678.275,72 ευρώ
```

## Μετά

Η λογική εξήχθη σε καθαρή συνάρτηση, κοινή για τον normalizer και το aggregation pipeline:

```js
export function extractCounterparty(praksi) {
    let sponsor = "άγνωστος", sponsorAfm = "-", amount = 0;
    const efv = praksi.extraFieldValues;
    if (!efv) return { sponsor, sponsorAfm, amount };

    if (efv.sponsor && efv.sponsor.length > 0) {
        sponsor    = efv.sponsor[0].sponsorAFMName?.name ?? "άγνωστος";
        sponsorAfm = efv.sponsor[0].sponsorAFMName?.afm  ?? "-";
        amount     = efv.sponsor[0].expenseAmount?.amount ?? 0;

    } else if (efv.person && efv.person.length > 0) {
        sponsor    = efv.person[0]?.name ?? "άγνωστος";
        sponsorAfm = efv.person[0]?.afm  ?? "-";
        amount     = efv.awardAmount?.amount ?? 0;

    } else if (efv.amountWithKae && efv.amountWithKae.length > 0) {
        // Το amountWithKae[i].amountWithVAT είναι αριθμός, όχι { amount }.
        // Το σύνολο βρίσκεται στο top-level efv.amountWithVAT.amount.
        sponsor    = efv.amountWithKae[0].sponsorAFMName?.name ?? "άγνωστος";
        sponsorAfm = efv.amountWithKae[0].sponsorAFMName?.afm  ?? "-";
        amount =
            efv.amountWithVAT?.amount ??
            efv.amountWithKae.reduce((s, line) => s + (Number(line.amountWithVAT) || 0), 0);

    } else if (efv.amountWithVAT) {
        sponsor    = efv.donationReceiver?.[0]?.name ?? "άγνωστος";
        sponsorAfm = efv.donationReceiver?.[0]?.afm  ?? "-";
        amount     = efv.amountWithVAT?.amount ?? 0;
    }

    return { sponsor, sponsorAfm, amount: Number(amount) || 0 };
}
```

## Επαλήθευση

```
Σύνολο πριν:  6.383.282,20 ευρώ
Σύνολο μετά:  22.061.557,92 ευρώ
Διαφορά:      15.678.275,72 ευρώ
```

Οι πράξεις αυτού του τύπου δεν φέρουν δημόσιο ΑΦΜ αναδόχου, οπότε ο μη ταυτοποιημένος αντισυμβαλλόμενος (`sponsorAfm === "-"`) εξαιρέθηκε από τους κορυφαίους αναδόχους, παραμένοντας στα συνολικά ποσά.

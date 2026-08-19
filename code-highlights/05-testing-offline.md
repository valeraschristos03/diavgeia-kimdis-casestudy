# 05 · Επαλήθευση χωρίς πρόσβαση στη βάση

## Ο περιορισμός

Το production MongoDB Atlas **δεν ήταν προσβάσιμο** από το περιβάλλον (TLS alert — IP allowlist / network):

```
❌ Απέτυχε η σύνδεση: ...SSL alert number 80
```

Δύο επιλογές: (α) να αλλάξω κώδικα «στα τυφλά», ή (β) να στήσω αξιόπιστο offline testing. Επιλέχθηκε το (β).

## Η προσέγγιση

```
mongodb-memory-server  →  πραγματική μηχανή MongoDB, στη μνήμη
        +
praxis_54446.json      →  το ΠΡΑΓΜΑΤΙΚΟ dataset (9.349 πράξεις)
```

Έτσι το aggregation pipeline δοκιμάστηκε σε **αληθινή** MongoDB με **αληθινά** δεδομένα — γι' αυτό και βρέθηκε το `$size`/`$isArray` bug (δες [02](02-stats-aggregation.md)): ένα in-memory mock δεν θα το έπιανε ποτέ.

## 1. Parity test — νέα διαδρομή vs παλιά

Συγκρίνεται κάθε μετρικό μεταξύ (α) in-memory `normalize + calculateStats` και (β) `statsAggregate`:

```
=== NO FILTERS ===
✅ decisionsLength    agg=9349        ref=9349
✅ withAmountLength   agg=3363        ref=3363
✅ withoutAmountCount agg=5986        ref=5986
✅ sum                agg=22061557.92 ref=22061557.92
✅ average            agg=6560.08     ref=6560.08
✅ median             agg=932         ref=932
✅ max                agg=1212323.02  ref=1212323.02
✅ min                agg=0.01        ref=0.01
✅ topSponsors        (10 εγγραφές, ταυτόσημες)

=== FILTER type=Β.1.3 minAmount=1000 maxAmount=50000 ===
✅ decisionsLength    agg=1114        ref=1114
✅ sum                agg=8988430.62  ref=8988430.62
✅ withAmountLength   agg=1114        ref=1114

ALL GOOD ✅
```

## 2. HTTP integration test — πραγματικός server

Ο Express server χρησιμοποιεί lazy, cached `getDb()`. Θέτοντας `MONGO_URI` στην in-memory instance **πριν** το `import`, χτυπιούνται τα endpoints σε πραγματικές HTTP κλήσεις:

```
Ο server ζει στο localhost:3000
stats 54446: 39.69ms
/stats           200  sum 22061557.92  len 9349  topSponsors 10
/stats filtered  200  len 1163  sum 17952995
/decisions       200  total 9349  rows 3
/decisions kw    200  total 15
/export csv      200  text/csv; charset=utf-8  lines 10109
/types           200  count 19
/organizations   200  count 1
/stats missing   404  (σωστό)
```

## 3. Full-pipeline test

Τρέξιμο του standalone `analyze.mjs` πάνω στο dataset → παρήχθη CSV όπου μια `Β.1.3` πράξη που **πριν** έβγαινε `0€` τώρα βγαίνει `19945€`. Ζωντανή απόδειξη του fix, end-to-end (normalize → filter → CSV/PDF).

## Καθαρισμός

Το `mongodb-memory-server` εγκαταστάθηκε με `--no-save` (δεν μπήκε στο `package.json`) και αφαιρέθηκε μετά τα tests — μηδενικό αποτύπωμα στις εξαρτήσεις του project.

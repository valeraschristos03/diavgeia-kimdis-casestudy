# 05 — Επαλήθευση χωρίς πρόσβαση στη βάση

## Περιορισμός

Το production MongoDB Atlas δεν ήταν προσβάσιμο από το περιβάλλον εργασίας, λόγω αστοχίας TLS (IP allowlist):

```
Απέτυχε η σύνδεση: ...SSL alert number 80
```

Η επαλήθευση έγινε offline αντί για αλλαγές χωρίς δοκιμή.

## Προσέγγιση

```
mongodb-memory-server  —  πραγματική μηχανή MongoDB στη μνήμη
praxis_54446.json      —  το πραγματικό dataset (9.349 πράξεις)
```

Το aggregation pipeline δοκιμάστηκε σε πραγματική MongoDB με πραγματικά δεδομένα. Γι' αυτό εντοπίστηκε το bug των `$size` / `$isArray` (βλ. [02](02-stats-aggregation.md)): ένα mock δεν θα το ανέδειχνε.

## Έλεγχος ισοδυναμίας

Σύγκριση κάθε μετρικής μεταξύ της in-memory διαδρομής (`normalize` + `calculateStats`) και του `statsAggregate`:

```
Χωρίς φίλτρα
  decisionsLength    agg=9349        ref=9349
  withAmountLength   agg=3363        ref=3363
  withoutAmountCount agg=5986        ref=5986
  sum                agg=22061557.92 ref=22061557.92
  average            agg=6560.08     ref=6560.08
  median             agg=932         ref=932
  max                agg=1212323.02  ref=1212323.02
  min                agg=0.01        ref=0.01
  topSponsors        10 εγγραφές, ταυτόσημες

Φίλτρο type=Β.1.3 minAmount=1000 maxAmount=50000
  decisionsLength    agg=1114        ref=1114
  sum                agg=8988430.62  ref=8988430.62
  withAmountLength   agg=1114        ref=1114
```

## Έλεγχος ενοποίησης HTTP

Ο Express server χρησιμοποιεί lazy, cached `getDb()`. Θέτοντας το `MONGO_URI` στην in-memory instance πριν το `import`, χτυπήθηκαν τα endpoints με πραγματικές HTTP κλήσεις:

```
stats 54446: 39.69ms
/stats           200  sum 22061557.92  len 9349  topSponsors 10
/stats filtered  200  len 1163  sum 17952995
/decisions       200  total 9349  rows 3
/decisions kw    200  total 15
/export csv      200  text/csv; charset=utf-8  lines 10109
/types           200  count 19
/organizations   200  count 1
/stats missing   404
```

## Έλεγχος πλήρους διαδρομής

Το standalone `analyze.mjs` τρέχτηκε πάνω στο dataset και παρήγαγε CSV όπου μια πράξη τύπου `Β.1.3`, που προηγουμένως εμφάνιζε ποσό 0, εμφανίζει πλέον 19945 (normalize, filter, CSV/PDF).

## Καθαρισμός

Το `mongodb-memory-server` εγκαταστάθηκε με `--no-save` και αφαιρέθηκε μετά τις δοκιμές. Δεν προστέθηκε στις εξαρτήσεις του project.

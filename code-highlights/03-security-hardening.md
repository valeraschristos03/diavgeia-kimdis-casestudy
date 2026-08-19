# 03 — Ασφάλεια και ανθεκτικότητα

## Regex injection / ReDoS

Πριν, η αναζήτηση φορέα τοποθετούσε μη ελεγμένες λέξεις χρήστη σε Mongo `$regex`:

```js
const conditions = words.map((w) => ({
  label: { $regex: w, $options: "i" },
}));
```

Το reverse lookup ΑΔΑΜ ένωνε επίσης μη ελεγμένα strings, ορισμένα από εξωτερικό API:

```js
.find({ subject: { $regex: adams.join("|") } })
```

Είσοδος όπως `(a+)+` ή `(.*a){20}` μπορεί να προκαλέσει catastrophic backtracking (ReDoS). Επιπλέον, χαρακτήρες όπως `.`, `*`, `(` αλλάζουν τη σημασία της αναζήτησης (regex injection).

Μετά, με escaping σε κάθε σημείο:

```js
function escapeRegex(text) {
  return String(text).replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}

// αναζήτηση φορέα
label: { $regex: escapeRegex(w), $options: "i" }

// reverse lookup ΑΔΑΜ
const pattern = adams.map(escapeRegex).join("|");
.find({ subject: { $regex: pattern } })
```

## Χειρισμός σφαλμάτων

Πριν, ένα γενικό `catch` μετέτρεπε κάθε αποτυχία σε 404:

```js
} catch (error) {
  response.status(404).json({ error: `Δεν βρέθηκαν δεδομένα για το ΑΦΜ ${afm}` });
}
```

Σε πτώση της βάσης, ο χρήστης έβλεπε κενό αποτέλεσμα και δεν υπήρχε ίχνος για διάγνωση.

Μετά, με διαχωρισμό των περιπτώσεων:

```js
try {
  const stats = await statsAggregate(afm, options, afm);
  if (stats === null) {
    return response.status(404).json({ error: `Δεν βρέθηκαν δεδομένα για το ΑΦΜ ${afm}` });
  }
  response.json(stats);
} catch (error) {
  console.error(`stats ${afm} ΣΦΑΛΜΑ:`, error);
  response.status(500).json({ error: "Σφάλμα κατά τον υπολογισμό στατιστικών" });
}
```

- 404: κενό αποτέλεσμα.
- 500: πραγματικό σφάλμα, με καταγραφή.

## Μυστικά εκτός version control

Το `backend/.env` περιέχει το `MONGO_URI` και αποκλείστηκε:

```gitignore
.env
.env.*
!.env.example
backend/.env
```

Προστέθηκε `.env.example` με placeholder. Πριν το πρώτο commit επιβεβαιώθηκε ρητά ότι κανένα `.env`, dataset ή `node_modules` δεν συμπεριλήφθηκε στο index.

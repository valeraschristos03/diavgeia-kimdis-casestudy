# 03 · Ασφάλεια & robustness

## 1. Regex-injection / ReDoS

**Πριν** — η αναζήτηση φορέα έβαζε **raw λέξεις χρήστη** σε Mongo `$regex`:

```js
const conditions = words.map((w) => ({
  label: { $regex: w, $options: "i" },   // ← w = ό,τι πληκτρολογήσει ο χρήστης
}));
```

Και το reverse-lookup ΑΔΑΜ ένωνε **μη ελεγμένα** strings (κάποια από εξωτερικό API):

```js
.find({ subject: { $regex: adams.join("|") } })
```

**Κίνδυνος:** ένα input όπως `(a+)+` ή `(.*a){20}` προκαλεί *catastrophic backtracking* → η βάση καίει CPU (ReDoS). Επίσης χαρακτήρες όπως `.` `*` `(` αλλάζουν το νόημα της αναζήτησης (regex-injection).

**Μετά** — escaping σε κάθε σημείο:

```js
function escapeRegex(text) {
  return String(text).replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}

// αναζήτηση φορέα
label: { $regex: escapeRegex(w), $options: "i" }

// reverse-lookup ΑΔΑΜ
const pattern = adams.map(escapeRegex).join("|");
.find({ subject: { $regex: pattern } })
```

## 2. Τα σφάλματα δεν κρύβονται πια πίσω από 404

**Πριν** — ένα γενικό `catch` μετέτρεπε **κάθε** αποτυχία σε «δεν βρέθηκαν δεδομένα»:

```js
} catch (error) {
  response.status(404).json({ error: `Δεν βρέθηκαν δεδομένα για το ΑΦΜ ${afm}` });
}
```

Αν έπεφτε η βάση, ο χρήστης έβλεπε «κενό» — και ο developer δεν είχε ίχνος.

**Μετά** — καθαρός διαχωρισμός:

```js
try {
  const stats = await statsAggregate(afm, options, afm);
  if (stats === null) {
    return response.status(404).json({ error: `Δεν βρέθηκαν δεδομένα για το ΑΦΜ ${afm}` });
  }
  response.json(stats);
} catch (error) {
  console.error(`stats ${afm} ΣΦΑΛΜΑ:`, error);     // ίχνος για διάγνωση
  response.status(500).json({ error: "Σφάλμα κατά τον υπολογισμό στατιστικών" });
}
```

- `404` → όντως κενό
- `500` → πραγματικό σφάλμα (με log)

## 3. Μυστικά εκτός git

Το `backend/.env` περιέχει το ζωντανό `MONGO_URI`. Αποκλείστηκε ρητά:

```gitignore
# --- secrets (ΠΟΤΕ στο git) ---
.env
.env.*
!.env.example
backend/.env
```

και δόθηκε `.env.example` με placeholder αντί για την πραγματική τιμή.

> **Επαλήθευση πριν το πρώτο commit:** έγινε ρητός έλεγχος ότι κανένα `.env`, dataset ή `node_modules` δεν μπήκε στο index — πριν φύγει οτιδήποτε προς το GitHub.

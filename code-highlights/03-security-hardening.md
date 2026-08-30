# 03 — Security and robustness

## Regex injection / ReDoS

Before, org search put unescaped user words into a Mongo `$regex`:

```js
const conditions = words.map((w) => ({
  label: { $regex: w, $options: "i" },
}));
```

The ADAM reverse lookup also joined unescaped strings, some coming from an external API:

```js
.find({ subject: { $regex: adams.join("|") } })
```

Two concrete risks:

- **ReDoS.** A query such as `(a+)+b` or `(.*a){20}` forces catastrophic backtracking; the database burns CPU on a single request.
- **Injection.** Characters like `.`, `*`, `(`, `|` change the meaning of the search. Typing `.*` would match every label; an unbalanced `(` would throw a regex error.

After, with escaping at every point:

```js
function escapeRegex(text) {
  return String(text).replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}

// org search
label: { $regex: escapeRegex(w), $options: "i" }

// ADAM reverse lookup
const pattern = adams.map(escapeRegex).join("|");
.find({ subject: { $regex: pattern } })
```

For ordinary Greek input (for example `ΔΗΜ`) `escapeRegex` is a no-op, so normal substring matching is unchanged; only the dangerous characters are neutralised.

## Errors no longer hide behind a 404

Before, a blanket `catch` turned every failure into a 404:

```js
} catch (error) {
  response.status(404).json({ error: `No data for AFM ${afm}` });
}
```

On a database outage the user saw an empty result and there was no trace for diagnosis.

After, with the cases separated:

```js
try {
  const stats = await statsAggregate(afm, options, afm);
  if (stats === null) {
    return response.status(404).json({ error: `No data for AFM ${afm}` });
  }
  response.json(stats);
} catch (error) {
  console.error(`stats ${afm} ERROR:`, error);
  response.status(500).json({ error: "Error while computing statistics" });
}
```

- 404: genuinely empty result.
- 500: real error, logged.

The same split was applied to the `/export`, `/types` and `/refresh` handlers.

## Secrets out of version control

`backend/.env` holds `MONGO_URI` and was excluded:

```gitignore
.env
.env.*
!.env.example
backend/.env
```

A placeholder `backend/.env.example` was added. Before the first commit, the index was checked explicitly so that no `.env`, dataset or `node_modules` was ever staged:

```bash
git ls-files | grep -E '\.env$|praxis_.*\.(json|txt)|node_modules'
# (no output = clean)
```

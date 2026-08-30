# 05 — Verification without database access

## Constraint

The production MongoDB Atlas cluster was not reachable from the work environment, failing on TLS (IP allowlist):

```
Connection failed: ...SSL alert number 80
```

Verification was done offline rather than changing code untested.

## Approach

```
mongodb-memory-server  —  a real MongoDB engine, in memory
praxis_54446.json      —  the real dataset (9,349 decisions)
```

The aggregation pipeline was tested against a real MongoDB with real data. That is exactly why the `$size` / `$isArray` bug surfaced (see [02](02-stats-aggregation.md)): a mock would not have exposed it.

## 1. Equivalence check

Every metric is compared between the in-memory path (`normalize` + `calculateStats`) and `statsAggregate`. This is committed as `backend/tests/stats-parity.test.mjs` and runs with `npm test`:

```
No filters
  decisionsLength    agg=9349        ref=9349
  withAmountLength   agg=3363        ref=3363
  withoutAmountCount agg=5986        ref=5986
  sum                agg=22061557.92 ref=22061557.92
  average            agg=6560.08     ref=6560.08
  median             agg=932         ref=932
  max                agg=1212323.02  ref=1212323.02
  min                agg=0.01        ref=0.01
  topSponsors        10 rows, identical

Filter type=Β.1.3, minAmount=1000, maxAmount=50000
  decisionsLength    agg=1114        ref=1114
  withAmountLength   agg=1114        ref=1114
  sum                agg=8988430.62  ref=8988430.62
  median             agg=3500        ref=3500
  min                agg=1000        ref=1000
  max                agg=50000       ref=50000
  topSponsors        5 rows, identical

ALL GOOD
```

The test seeds the in-memory server itself, so it needs no external database:

```bash
cd backend && npm install && npm test
```

## 2. HTTP integration check

The Express server uses a lazy, cached `getDb()`. Setting `MONGO_URI` to the in-memory instance before importing the server allows hitting the endpoints over real HTTP:

```
stats 54446: 39.69ms
/stats           200  sum 22061557.92  len 9349  topSponsors 10
/stats filtered  200  len 1163  sum 17952995      (type=Β.1.3, minAmount=1000)
/decisions       200  total 9349  rows 3
/decisions kw    200  total 15                     (keyword=ΣΥΜΒΑΣΗ)
/export csv      200  text/csv; charset=utf-8  lines 10109
/types           200  count 19
/organizations   200  count 1
/stats missing   404
```

The `39.69ms` figure is on a local in-memory MongoDB with 9,349 documents and is not a production benchmark. The substantive gain is architectural: the computation runs in the database, without moving every document into Node.

## 3. Full-pipeline check

The standalone `analyze.mjs` was run over the dataset and produced a CSV in which a `Β.1.3` decision that previously showed 0 now shows 19945 (normalize -> filter -> CSV/PDF), confirming the fix end to end.

## Clean-up

`mongodb-memory-server` is declared as a `devDependency` (used by `npm test`). It was installed with `--no-save` during exploratory runs and only added to `package.json` once the test was made permanent, so it never leaked into the application's runtime dependencies.

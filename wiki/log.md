# Log

Append-only chronological record of wiki activity (ingests, queries, lints, digests).

Each entry starts with `## [YYYY-MM-DD] <op> | ...` so it is greppable:

```bash
grep "^## \[" wiki/log.md | tail -n 10
```

---

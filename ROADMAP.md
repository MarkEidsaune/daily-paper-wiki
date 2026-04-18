# Roadmap

A living list of design questions, deferred features, and operational notes for the Daily Paper Wiki. Items here are deliberately **not** in `AGENTS.md` because they're either unresolved, out of scope for v1, or workflow concerns the agent doesn't need on its hot path. Revisit during quiet sessions.

---

## Deferred design decisions

These all came up during the v1 design but were intentionally postponed. When one of them becomes a real problem, lift it into `AGENTS.md` as a rule.

### 1. arXiv version updates

`AGENTS.md` §3 says we strip the version suffix from arxiv IDs and store `arxiv_version` separately, so the wiki page stays at the same path across versions. **What we haven't decided** is the actual update workflow:

- How does the agent learn that v3 has been published? (HF re-surfaces? user prompt? periodic re-check?)
- When v3 lands, do we overwrite `paper.pdf` and `page.md`, or save them side-by-side as `paper.v3.pdf`?
- Does the wiki page get a "version history" section, or do we silently update?
- Does the cascade re-run (authors/orgs may change between versions)?

**Tentative direction**: when the user says `update <arxiv_id>`, fetch the latest version, diff it against the stored `arxiv_version`, and only re-ingest if newer. Keep `paper.pdf` versioned per-version (`paper.v1.pdf`, `paper.v2.pdf`).

### 2. Same paper appearing on multiple HF dailies

HF sometimes re-surfaces older papers (or a v2 lands and the paper gets re-promoted). v1 will detect this on the second day's ingest because `raw/<old_date>/<arxiv_id>/` already exists at a *different* date.

**Open question**: do we add a back-reference from the new date's clipping notes to the old `raw/<old_date>/<arxiv_id>/`? Do we update the paper page's `date:` to the most recent appearance, or keep it pinned to first sighting? (Lean: keep `date:` as first sighting; add a `re_surfaced_on:` list.)

### 3. HF JSON API as scrape alternative

We currently fetch HF paper pages as HTML/markdown via web fetch. HF also exposes JSON at `https://huggingface.co/api/papers/<arxiv_id>` and `https://huggingface.co/api/daily_papers?date=YYYY-MM-DD`. JSON would be:

- More robust against layout changes,
- Trivially parseable for fields like upvotes, authors, GitHub URL,
- Faster (smaller payload).

Cost: we'd lose the abstract images that come embedded in the HTML render. Probable end state: use JSON for the metadata, keep HTML for `page.md` (as a human-readable archive).

### 4. GitHub repo exploration

v1 captures `github_url` and `github_stars` only. v2 ideas:

- Fetch and store the README to `raw/<date>/<arxiv_id>/github_readme.md`.
- Capture `language`, `topics`, `last_commit_at` from the GitHub API.
- For deep-tier papers, optionally read `src/` to ground the wiki page in actual implementation.

The GitHub REST API is generous on rate limits with a token; this is mostly a "decide what to capture" question, not a feasibility one.

### 5. Video handling

v1 records video URLs in `meta.yaml` only. Future ideas:

- Use `yt-dlp` to download HF-hosted demos and store them under `raw/<date>/<arxiv_id>/videos/`.
- Use `ffmpeg` to extract keyframes (e.g. every 5 seconds) into `assets/` so the agent can "view" the video via the keyframes.
- For deep-tier papers with a video that's clearly a demo of the method, this would meaningfully improve the wiki write-up.

Holding off until v1 reveals whether the lost signal actually matters.

---

## Operational notes

Things that won't fit anywhere on the agent's hot path but are worth keeping in mind during real use.

### Storage trajectory

A typical day is ~10-30 papers, each with a ~2-10 MB PDF, plus a few hundred KB of images. Call it ~100 MB/day on the high end → **~36 GB/year**. Implications:

- Git is fine for a year or two, but eventually `paper.pdf` files should probably move to git-LFS or be `.gitignore`d entirely (they're trivially re-fetchable from arXiv).
- Cheaper interim move: add `raw/**/paper.pdf` to `.gitignore` and rely on the re-fetch idempotency in `AGENTS.md` §5 step 4 to repopulate.
- Decide once `du -sh .` crosses ~5 GB.

### Recovery model

Since `wiki/` is fully agent-generated and `raw/` is mostly re-fetchable, "recovery" is just `git`:

- Bad ingest? `git checkout -- wiki/ raw/<date>/`
- Want to start a date over? `rm -rf raw/<date>/ && rm wiki/papers/<...>.md` then re-run `ingest <date>`.
- Wiki got tangled across many sessions? `git revert` the bad commits, or branch off a known-good point.

The implication for daily use: **commit per ingest**, with a message like `ingest 2026-04-17 (12 papers, 4 deep)`. This makes blame and revert trivial.

### Iterating on AGENTS.md

`AGENTS.md` is the single source of truth for agent behavior, and it will drift away from reality every time a real ingest reveals an unwritten rule. After each session:

1. If the agent had to ask for clarification on something not in `AGENTS.md`, add the resolution to `AGENTS.md`.
2. If the user gave a one-off override that should become standing policy, add it.
3. If a deferred question above became a real problem, promote it from this file to `AGENTS.md`.

Treat `AGENTS.md` as a Constitution, not a Bible — amend it when reality demands.

### Periodic `lint` runs

`lint` is on-demand, but the agent should suggest running it when:

- More than ~5 ingests have happened since the last `lint`,
- A `compare` query surfaces contradictions across papers,
- Before any `digest` operation (so the digest reads from a clean wiki).

The user can ignore the suggestion; the agent shouldn't run `lint` unprompted.

### Cross-machine sync (laptop ↔ server)

If the wiki is mirrored to a server (e.g. for read-only browsing), use `git` for sync:

- Laptop is the canonical write source (has Obsidian + Web Clipper).
- Server pulls (read-only mirror).
- Never write to the server's copy directly — merge conflicts in the wiki are no fun.

---

## How to use this file

- **Add to it freely** during sessions when a question comes up but isn't worth resolving immediately.
- **Promote** items into `AGENTS.md` when they become real rules (and remove them from here).
- **Cite it** from `AGENTS.md` when deferring a question, the way §3 cites it for arXiv version updates.

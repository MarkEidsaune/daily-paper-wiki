# daily-paper-wiki

A personal knowledge base for [Hugging Face Daily Papers](https://huggingface.co/papers), maintained by an LLM agent.

The idea: instead of re-querying raw papers every time you have a question, the agent incrementally builds and maintains a structured wiki — per-paper pages, plus interlinked pages for authors, orgs, models, methods, datasets, benchmarks, and topics. The agent does all the bookkeeping; you do the curation. The full operating manual is in [AGENTS.md](AGENTS.md).

## Layout

```
.
├── AGENTS.md                 # the schema (read this if you want to know how the agent works)
├── raw/                      # immutable source material (never edited after first write)
│   ├── clippings/            # daily HF index pages (Web Clipper output)
│   └── YYYY-MM-DD/<id>/      # per-paper sources (HF page, PDF, images, meta)
└── wiki/                     # agent-owned synthesis layer (browsed in Obsidian)
    ├── index.md
    ├── log.md
    ├── papers/, authors/, orgs/, models/, methods/, datasets/, benchmarks/, topics/
    ├── trends/               # weekly/monthly digests (on-demand)
    └── lint-reports/
```

## Setup (one-time)

### 1. Open the repo as an Obsidian vault

In Obsidian: `Open another vault` → `Open folder as vault` → pick this repo's root. The `.obsidian/` folder is committed, so the vault-wide attachment folder (`raw/clippings/assets/`) and other settings are already configured.

### 2. Install Obsidian plugins

- **Dataview** (Community plugin) — page templates use YAML frontmatter that Dataview can query. Install and enable.
- **Obsidian Web Clipper** (browser extension, separate from in-app plugins) — the daily kickoff. Install in your browser.

### 3. Configure the Web Clipper template

In the Web Clipper extension, create a new template:

| Setting | Value |
| --- | --- |
| Template name | `HF Daily Papers` |
| Trigger (URL match) | `https://huggingface\.co/papers(/date/.*)?` |
| Vault | this vault |
| Output folder | `raw/clippings/` |
| Filename | `{{url\|split:"/"\|last}}` |
| Image handling | Download attachments → `raw/clippings/assets/` |

That's all the per-machine setup.

#### How the filename extraction works

Hugging Face's `https://huggingface.co/papers` URL 302-redirects to `https://huggingface.co/papers/date/YYYY-MM-DD` (today's papers). The Web Clipper captures the post-redirect URL, so `{{url}}` always contains the date — whether you visited `/papers` or a specific `/papers/date/YYYY-MM-DD` page.

The filter chain `{{url|split:"/"|last}}` works like this:

1. `split:"/"` breaks the URL on slashes:
   `https://huggingface.co/papers/date/2026-04-17` → `["https:", "", "huggingface.co", "papers", "date", "2026-04-17"]`
2. `last` returns the final element: `"2026-04-17"`.

So clipping today's page or any past day's page (e.g. `https://huggingface.co/papers/date/2025-08-12`) writes to `raw/clippings/<that-date>.md` — exactly what we want for backfilling history.

#### Defensive variant (Web Clipper 1.0.0+)

If you want a safety net in case `{{url}}` ever doesn't contain a date (manually-entered URL without redirect, future HF URL changes, etc.), use a conditional with a fallback to today's date:

```twig
{% if url contains "/papers/date/" %}{{url|split:"/"|last}}{% else %}{{date|date:"YYYY-MM-DD"}}{% endif %}
```

This requires Web Clipper 1.0.0 or later (logic syntax). The simpler `{{url|split:"/"|last}}` form works on all versions and is fine for normal use.

## Daily usage

1. **Clip the day.** Visit `https://huggingface.co/papers` for today, or `https://huggingface.co/papers/date/YYYY-MM-DD` for any past day, and hit the Web Clipper. It writes `raw/clippings/<that-date>.md` — the filename is derived from the URL, so backfilling old days works the same way as ingesting today.
2. **Kick off the agent.** Open this repo in Cursor and say:

   ```
   ingest today
   ```

   The agent reads the clipping, proposes a triage table (deep vs skim) with reasons, and waits for you to confirm or override. Then it processes everything: fetches each paper from HF + arXiv, writes per-paper wiki pages, and cascades updates to entity pages.
3. **Browse the wiki.** Switch to Obsidian and explore the result — graph view, page links, Dataview tables.

## Command vocabulary

The agent recognizes (see [AGENTS.md §7](AGENTS.md) for full details):

| Command | What it does |
| --- | --- |
| `ingest [date]` | The daily flow above. |
| `query <question>` | Answer using the wiki, with citations. Often files the answer back as a new wiki page. |
| `lint` | Health check (orphans, contradictions, stale claims, missing entity pages). Output to `wiki/lint-reports/`. |
| `digest week <YYYY-Wnn>` / `digest month <YYYY-MM>` | Generate a trend page for that period. |
| `compare <A> <B>` | Comparison page. |
| `deep <arxiv_id>` | Promote a previously-skimmed paper to a full deep-read. |
| `skim <arxiv_id>` | Override a deep pick during triage. |

## Design choices

- **Vault root = repo root.** Obsidian sees the same tree git does. `.obsidian/` is committed (with UI ephemera like `workspace.json` ignored via `.obsidian/.gitignore`).
- **No Python tooling in v1.** The agent does all fetching itself via `curl` and built-in web fetch. No `pyproject.toml` / `uv.lock` until something needs them.
- **Agent decides triage; abstracts are the floor.** Every paper gets a wiki page; deep-reads are bonus. Missing a deep pick is recoverable later via `deep <arxiv_id>`.
- **GitHub URLs captured but not fetched** — see [ROADMAP.md](ROADMAP.md).
- **Trends are on-demand.** Daily ingest doesn't write `wiki/trends/`; you run `digest week` / `digest month` when you want one.

## Reference

- [AGENTS.md](AGENTS.md) — the operating manual the agent follows
- [ROADMAP.md](ROADMAP.md) — deferred design questions and operational notes (storage, recovery, sync, etc.)

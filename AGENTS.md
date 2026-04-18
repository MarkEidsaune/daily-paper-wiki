# AGENTS.md — daily-paper-wiki schema

You are the maintainer of a personal wiki for [Hugging Face Daily Papers](https://huggingface.co/papers). This file is your operating manual — read it before every session. It's the single source of truth for how this project works.

---

## 0. The pattern (why this exists)

Most LLM-plus-documents systems are RAG: documents go in, the LLM retrieves chunks at query time and generates an answer. Knowledge is rediscovered on every question; nothing accumulates.

This project does the opposite. Instead of retrieving from raw sources at query time, you **incrementally build and maintain a persistent wiki** — a structured, interlinked collection of markdown files that sits between the user and the raw HF + arXiv sources. When a new paper arrives, you don't just index it. You read it, extract the key information, and integrate it into the existing wiki — updating entity pages (authors, orgs, models, methods), revising topic summaries, noting where new data contradicts or supersedes old claims. The knowledge is compiled once and then *kept current*.

The wiki is a **persistent, compounding artifact.** The cross-references are already there. The contradictions have already been flagged. The synthesis already reflects everything ingested. Every new paper makes the wiki richer, not just longer.

The user **never (or rarely) writes the wiki themselves.** You write and maintain all of it. The user is in charge of sourcing (clipping the daily HF page), exploration, and asking the right questions. You do all the grunt work — the summarizing, cross-referencing, filing, and bookkeeping that makes a knowledge base actually useful over time.

**Why it works:** the tedious part of maintaining a knowledge base isn't the reading or the thinking — it's the bookkeeping. Updating cross-references, keeping summaries current, noting when new data supersedes old claims, maintaining consistency across dozens of pages. Humans abandon wikis because the maintenance burden grows faster than the value. You don't get bored, don't forget to update a cross-reference, and can touch 15 files in one pass. The wiki stays maintained because the cost of maintenance is near zero.

There are three core operations:

- **Ingest** — the daily flow. User clips a source; you read it, write a paper page, and cascade updates across entity pages. See §5.
- **Query** — answer questions against the wiki, with citations. Good answers can be filed back to the wiki as new pages so explorations compound. See §7.
- **Lint** — periodic health check for orphans, contradictions, stale claims, and missing pages. See §7.

Everything else (Compare, Digest, Deep, Skim) is derivative.

---

## 1. Three layers

| Layer | Path | Owner | Mutability |
| --- | --- | --- | --- |
| Raw sources | `raw/` | User clips into `raw/clippings/`; you fetch into `raw/<date>/<arxiv_id>/` | **Immutable after first write.** Once a file lands here, never modify it. |
| Wiki | `wiki/` | You | You write everything here. User reads it through Obsidian. |
| Schema | `AGENTS.md` (this file) | Co-evolved with user | Update only when the user asks you to. |

**`raw/` is sacred** — with one narrow exception: `raw/clippings/<date>.md` is owned by the Web Clipper, and re-clipping a date will overwrite that file (HF adds papers throughout the day, so users may re-clip later for fuller coverage). Per-paper `raw/<date>/<arxiv_id>/` folders, in contrast, are write-once. When in doubt, ask before touching anything under `raw/` that isn't a brand-new download or a Web Clipper re-write.

---

## 2. Directory layout

```
.
├── AGENTS.md                           # this file (the schema, single source of truth)
├── README.md                           # human-facing setup
├── .obsidian/                          # vault config (committed; ephemera ignored)
├── raw/
│   ├── clippings/
│   │   ├── YYYY-MM-DD.md               # daily HF index page (Web Clipper output)
│   │   └── assets/                     # Obsidian Web Clipper attachment dump
│   └── YYYY-MM-DD/
│       └── <arxiv_id>/
│           ├── page.md                 # HF paper page contents (you save)
│           ├── paper.pdf               # arXiv full PDF (you curl)
│           ├── meta.yaml               # structured metadata
│           └── assets/                 # images from HF page (you curl)
└── wiki/
    ├── index.md                        # catalog of all pages
    ├── log.md                          # append-only chronological
    ├── papers/<arxiv_id> - <slug>.md
    ├── authors/<Author Name>.md
    ├── orgs/<Org Name>.md
    ├── models/<Model Name>.md
    ├── methods/<Method Name>.md
    ├── datasets/<Dataset Name>.md
    ├── benchmarks/<Benchmark Name>.md
    ├── topics/<Topic Name>.md
    ├── trends/YYYY-Wnn.md, YYYY-MM.md  # on-demand only
    ├── lint-reports/YYYY-MM-DD.md      # output of `lint`
    └── assets/                         # images embedded in wiki pages
```

---

## 3. Conventions

### Filenames
- **Use spaces and Title Case** (Obsidian-native). Example: `wiki/authors/Yann LeCun.md`, not `Yann_LeCun.md`.
- **Paper page filename**: `<arxiv_id> - <short slug>.md`. Example: `2604.14268 - HY-World 2.0.md`. The arXiv ID prefix gives stable, sortable, unique names — slug collisions are harmless.
- **Date-keyed files** use ISO format: `2026-04-17.md`, `2026-W16.md`, `2026-04.md`.

### Paper slug derivation

The `<short slug>` part of a paper filename is derived from the title with this exact procedure (apply in order):

1. If the title contains `:`, take the part before the first `:`. Otherwise take the full title.
2. Strip LaTeX scaffolding: drop `^`, `_`, `{`, `}`, `\` characters (keep their contents). So `DR^{3}-Eval` → `DR3-Eval`.
3. Replace filesystem-illegal characters `/ \\ : * ? " < > |` with a single space.
4. Collapse runs of whitespace to single spaces; trim leading/trailing whitespace.
5. If longer than 50 characters, truncate at the last word boundary <=50 chars.

Worked examples:

| Title | Slug |
| --- | --- |
| `HY-World 2.0: A Multi-Modal World Model for ...` | `HY-World 2.0` |
| `DR^{3}-Eval: Towards Realistic and Reproducible Deep Research Evaluation` | `DR3-Eval` |
| `KV Packet: Recomputation-Free Context-Independent KV Caching for LLMs` | `KV Packet` |
| `Three-Phase Transformer` (no colon) | `Three-Phase Transformer` |
| `RAD-2: Scaling Reinforcement Learning in a Generator-Discriminator Framework` | `RAD-2` |

### arXiv IDs

- **Modern format**: `YYMM.NNNNN` (5-digit serial), e.g. `2604.14268`.
- **Older format**: `YYMM.NNNN` (4-digit serial), e.g. `2211.16780`. Both are valid; ingest both as-is.
- **Versioned IDs**: arXiv papers carry a version suffix when not on v1, e.g. `2604.14268v2`. Always **strip the version** before using the ID anywhere structural — folder names, filenames, wikilinks. The version is recorded in metadata fields:
  - Per-paper folder: `raw/<date>/2604.14268/` (no `v`).
  - Wiki page: `wiki/papers/2604.14268 - <slug>.md` (no `v`).
  - `meta.yaml` records both `arxiv_id: 2604.14268` and (when present) `arxiv_version: v2` as separate fields.
  - Paper page frontmatter mirrors the same.

  This way, when a new version lands later we can update the same wiki page rather than fragmenting across versions. The full version-update workflow is in [ROADMAP.md](ROADMAP.md).

### Wikilinks
- Always use `[[Name]]` style (not relative markdown links). Obsidian resolves by filename across folders, so `[[Yann LeCun]]` works from any page.
- Cross-references are **bidirectional**. If `wiki/papers/X.md` lists `[[Yann LeCun]]`, then `wiki/authors/Yann LeCun.md` must list `[[X]]` back. Never leave one side dangling.

### Frontmatter
- Every wiki page has YAML frontmatter (Dataview-queryable).
- Required on every page: `type`, `tags`.
- Type-specific fields: see the templates in §6.
- Lists in frontmatter use inline YAML arrays: `orgs: [DeepMind, Stanford NLP]`.

### Log format
Append entries to `wiki/log.md` in this exact shape so it's greppable:

```
## [YYYY-MM-DD] <op> | <one-line summary>

- detail
- detail
```

`<op>` is one of: `ingest`, `query`, `lint`, `digest`, `compare`, `deep`, `skim`.

`grep "^## \[" wiki/log.md | tail -n 10` should always give a useful timeline.

---

## 4. Tools you use

All native to Cursor + the shell. **No Python tooling exists in v1** — don't try to import a `dpw` package or anything like that.

| Need | Tool |
| --- | --- |
| Fetch HTML page | Web fetch |
| Download binary (PDF, image) | `curl -L -o <path> <url>` |
| Read PDF | Cursor's Read tool (handles PDFs natively) |
| View image | Cursor's Read tool (jpeg/png/gif/webp supported) |
| Watch video | **Not supported.** Record URL in `meta.yaml` only. |
| Read GitHub repo | **Not supported in v1.** Record `github_url` and `github_stars` only. |

### Concrete fetch recipes

```bash
# arXiv PDF — the /pdf/ URL serves the raw PDF
curl -L -o raw/2026-04-17/2604.14268/paper.pdf "https://arxiv.org/pdf/2604.14268"

# Image from HF
curl -L -o raw/2026-04-17/2604.14268/assets/teaser.png "<image-url>"
```

---

## 5. Daily workflow: `ingest`

Triggered by user prompt like "ingest today", "ingest 2026-04-17", or just "ingest". The user can also backfill old days — clippings in `raw/clippings/` are named by the date the HF page covers (parsed from the URL), not the date they were clipped, so old days look identical to today's day from your perspective.

### Step 1 — Find the clipping

Look in `raw/clippings/` for the requested date.

- "ingest 2026-04-17" → look for `raw/clippings/2026-04-17.md`.
- "ingest today" → look for `raw/clippings/<today's date>.md`. If not present, fall back to the most recent clipping in `raw/clippings/`.
- "ingest" with no argument → most recent clipping.

If no matching clipping exists, stop and tell the user to run the Web Clipper first (visiting `https://huggingface.co/papers/date/<the date>`).

The date the clipping covers is also recorded in the clipping's frontmatter — confirm `published:` (or the `source:` URL's last path segment) matches what you expect before processing. If you ever process a clipping for date X, every paper page you write should use date X (not today's date) in `published:`, in the per-paper `meta.yaml`, in `raw/<date>/...` paths, and in the `wiki/papers/...` page frontmatter `date:` field.

**Re-clipping a date is a normal scenario.** HF adds papers throughout the day, so the user may re-clip `2026-04-17` after their first clip. When the parsed paper list contains arxiv_ids you've already ingested for that date:

- New arxiv_ids (no `raw/<date>/<arxiv_id>/` folder yet) → fetch and ingest as new.
- Existing arxiv_ids → skip per the idempotency rules in step 4.
- Arxiv_ids in `raw/<date>/` that are no longer in the new clipping → leave untouched. (HF may have removed them from the index but we keep what we have.)

Log re-ingests as a separate `## [YYYY-MM-DD] ingest` entry in `wiki/log.md`, noting it's a follow-up to an earlier ingest for the same date.

### Step 2 — Parse the clipping

The clipping is a markdown render of `https://huggingface.co/papers/date/YYYY-MM-DD`. Each paper appears as a chunk like:

```markdown
[<video src="..."></video>](https://huggingface.co/papers/2604.14268 "View paper")  <!-- optional -->
[![](https://cdn-thumbnails.huggingface.co/.../2604.14268.png)](https://huggingface.co/papers/2604.14268)  <!-- alternative -->

Submitted by <handle>

### <Title>

[
- ·
	N authors
](https://huggingface.co/papers/<arxiv_id>)

OR

[
<Org Name>
](https://huggingface.co/<org-slug>)

[<upvotes>](https://huggingface.co/papers/<arxiv_id>) [<comments>](https://huggingface.co/papers/<arxiv_id>#community)
```

For each paper, extract:
- `arxiv_id` (from any `/papers/<arxiv_id>` link)
- `title` (from the `### ` heading)
- `submitted_by` (from "Submitted by ...")
- `org` (if shown — otherwise leave blank, you'll get it from the HF page)
- `hf_upvotes` (the first bracketed number; may be missing — treat as 0)
- `hf_comments` (the second bracketed number; may be missing)
- `video_urls` (if a `<video src="...">` is present)

Some papers have no upvote count (only a community count); ingest them anyway.

### Step 3 — Triage (you decide; user can override)

You autonomously pick a tier for each paper using a vibe of:

- **Upvotes** — high upvotes (~100+) is a strong deep signal.
- **GitHub presence + stars** — repo linked from the HF page, especially with stars, is a deep signal.
- **Topic vibes** — does it extend a thread already in the wiki? (Check `wiki/index.md` and recent entries.) Does it sound interesting / methodologically novel? Use judgment.
- **Author/org track record** — has this lab/author appeared before with strong work? (Check `wiki/authors/` and `wiki/orgs/`.)

Default split: aim for ~20-30% deep, the rest skim. Adjust based on the day's quality.

**On the very first ingest** (or any time `wiki/index.md` is sparse), the "extends a thread" and "author/org track record" signals are unavailable. Fall back to upvotes + GitHub stars + raw topical novelty alone. Don't apologize for it — every wiki has a day one. Subsequent ingests get richer signals as the wiki accretes.

**Announce your picks before processing**, as a markdown table:

```
| arXiv ID | Title | ↑ | GH ★ | Tier | Why |
| --- | --- | --- | --- | --- | --- |
| 2604.14268 | HY-World 2.0 | 1090 | 1.2k | deep | huge upvotes, github with stars, extends [[World Models]] |
| 2604.15308 | RAD-2 | 207 | — | deep | extends our RL thread |
| 2604.14683 | DR^3-Eval | 21 | — | skim | benchmark paper, low signal vs novelty |
| ... |
```

The user may then say `deep <arxiv_id>` or `skim <arxiv_id>` to override before you start. They may also say "go" to proceed as-is.

**Abstracts are the floor.** Every paper gets a wiki page, even skim-tier. A missed deep-read is recoverable later with `deep <arxiv_id>`.

### Step 4 — For each paper (in upvote-descending order)

**Per-paper steps are idempotent.** A re-run of `ingest <date>` after a network blip, cancel, or restart should be safe. Before each step below, check whether the artifact already exists and is non-empty; if so, skip that step. Specifically:

| Step | Skip if |
| --- | --- |
| 1 (fetch HF page) | `raw/<date>/<arxiv_id>/page.md` exists and is non-empty |
| 2 (write `meta.yaml`) | `meta.yaml` exists |
| 3 (download HF assets) | the asset already exists in `assets/` |
| 4 (download arXiv PDF) | `paper.pdf` exists and is >1 KB (sanity check against truncated downloads) |
| 5 (read PDF, deep tier only) | n/a — reading is cheap, just re-do |
| 6 (write wiki page) | `wiki/papers/<arxiv_id> - *.md` exists, **unless** the user said `ingest --force` |
| 7 (entity cascades) | always re-run; upserting a wikilink that already exists is a no-op |

If the user says `ingest --force <date>`, re-fetch and re-write everything for that date. If the user says `deep <arxiv_id>` or `skim <arxiv_id>` to change the tier on a previously-ingested paper, re-write only the wiki page (not the raw sources) and update `meta.yaml`'s `depth:` field.

1. **Fetch HF page** → save to `raw/<date>/<arxiv_id>/page.md`. Use Web fetch to get the page as markdown, then **prepend YAML frontmatter that matches the Web Clipper convention** (so all `raw/` markdown — both daily clippings and per-paper pages — has the same shape):

   ```yaml
   ---
   title: "HY-World 2.0: A Multi-Modal World Model for Reconstructing, Generating, and Simulating 3D Worlds"
   source: "https://huggingface.co/papers/2604.14268"
   author:
     - "Author One"
     - "Author Two"
   published: 2026-04-17        # paper's HF submission date (from the clipping)
   created: 2026-04-18          # when you fetched it
   description: "First sentence or two of the abstract."
   tags:
     - "papers"
   ---
   ```

   The keys (`title`, `source`, `author`, `published`, `created`, `description`, `tags`) and their string-quoting style mirror what Obsidian Web Clipper produces for the daily index clip in `raw/clippings/`. `author` is a YAML list (one entry per paper author); empty list `[]` if unknown. Then append the page body as returned by Web fetch.

2. **Save metadata** to `raw/<date>/<arxiv_id>/meta.yaml`:

   ```yaml
   arxiv_id: 2604.14268
   arxiv_version: v2          # omit if v1 / unversioned
   title: HY-World 2.0: A Multi-Modal World Model for Reconstructing, Generating, and Simulating 3D Worlds
   date: 2026-04-17
   hf_url: https://huggingface.co/papers/2604.14268
   arxiv_url: https://arxiv.org/abs/2604.14268
   hf_upvotes: 1090
   hf_comments: 4
   submitted_by: taesiri
   authors: [...]
   orgs: [Tencent]
   github_url: https://github.com/Tencent/HY-World
   github_stars: 1240
   video_urls: [https://cdn-uploads.huggingface.co/.../nxkkpAYJMbxhdq2PyUMC9.mp4]
   depth: deep
   depth_reason: 1k+ upvotes, github with stars, extends World Models thread
   ```

3. **Download images** referenced on the HF page → `raw/<date>/<arxiv_id>/assets/`. Use original filenames where sane.
4. **Download arXiv PDF** → `raw/<date>/<arxiv_id>/paper.pdf` via `curl -L -o ... "https://arxiv.org/pdf/<arxiv_id>"`.
5. **Read at the chosen depth**:
   - **Skim**: read `page.md` (abstract + figure captions) and view the most relevant 1-2 images. Don't open the PDF.
   - **Deep**: read the full PDF (Cursor's Read tool handles it), view all images in `assets/`, optionally pause to discuss notable findings with the user before writing.
6. **Write the paper page** at `wiki/papers/<arxiv_id> - <slug>.md` using the template in §6.1.
7. **Cascade entity updates**:
   - For each author → upsert `wiki/authors/<Name>.md`.
   - For each org → upsert `wiki/orgs/<Name>.md`.
   - For each model introduced or used → upsert `wiki/models/<Name>.md`.
   - For each method named (e.g. GRPO, MoE, speculative decoding) → upsert `wiki/methods/<Name>.md`.
   - For each dataset / benchmark → upsert `wiki/datasets/<Name>.md` / `wiki/benchmarks/<Name>.md`.
   - For each high-level topic (e.g. World Models, Video Generation) → upsert `wiki/topics/<Name>.md`.
   - **Always add bidirectional `[[wikilinks]]`.** Every entity page lists this paper; the paper page lists every entity.
   - Skim-tier papers still create entity entries (just one bullet under each entity's "Papers" list, no deep prose).

### Step 5 — Maintenance pass (after all papers processed)

1. **Update `wiki/index.md`**: add new pages under their categories. Keep entries one-line: `- [[Name]] — <one-line summary>`.
2. **Append `wiki/log.md`**:

   ```markdown
   ## [YYYY-MM-DD] ingest | <N> papers | <D> deep | <S> skim

   - deep:
     - [[2604.14268 - HY-World 2.0]] — 1090↑, Tencent
     - ...
   - skim:
     - [[2604.14683 - DR^3-Eval]] — 21↑
     - ...
   - new entities: 4 authors, 1 org ([[DeepGlint]]), 2 methods ([[GRPO]], [[Visual Switch KD]])
   ```

3. **Lint** (lightweight inline version — full lint is on-demand via the `lint` command):
   - Scan the new entity pages for orphans (no inbound links). If found, fix or note in the log entry.
   - Look for entity name collisions or near-duplicates (e.g. you wrote `[[Llama 4]]` but `wiki/models/Llama4.md` exists). Resolve.
4. **Stale-check**: when a new paper makes a claim that contradicts or supersedes an existing wiki page, edit the older page to add a "Superseded by [[X]]" or "Contested by [[X]]" note (with date), and mention the conflict in the new paper's "Notes & Open Questions". Surface a brief summary in the log entry.

**Do not write trend pages during ingest.** Trends are on-demand (see §7).

---

## 6. Page templates

Use these as the skeleton. Add or remove sections as the content warrants — don't pad just to fill structure.

### 6.1 Paper

`wiki/papers/<arxiv_id> - <slug>.md`

```markdown
---
type: paper
arxiv_id: 2604.14268
arxiv_version: v2          # omit if v1 / unversioned
title: HY-World 2.0: A Multi-Modal World Model for Reconstructing, Generating, and Simulating 3D Worlds
date: 2026-04-17
hf_url: https://huggingface.co/papers/2604.14268
arxiv_url: https://arxiv.org/abs/2604.14268
hf_upvotes: 1090
submitted_by: taesiri
github_url: https://github.com/Tencent/HY-World
github_stars: 1240
authors: [Author One, Author Two]
orgs: [Tencent]
models: [HY-World 2.0]
methods: [Diffusion, World Model]
datasets: []
benchmarks: []
topics: [3D Generation, World Models]
depth: deep
depth_reason: 1k+ upvotes, github with stars, extends World Models thread
tags: [paper]
---

# HY-World 2.0: ...

## TL;DR
One paragraph. What it is, why it matters, the headline result.

## Contributions
- Bullet 1
- Bullet 2

## Method
(Deep tier only — describe the approach in enough detail that someone reading just this page understands the technique.)

## Results
(Deep tier only — key numbers, comparisons, ablations worth remembering.)

## Notes & Open Questions
- Personal observations, things to follow up on, contradictions vs prior wiki claims.

## Related
- [[Other Paper Title]] — same method
- [[Another Paper]] — competing approach

## Sources
- HF: ../../raw/2026-04-17/2604.14268/page.md
- PDF: ../../raw/2026-04-17/2604.14268/paper.pdf
- Repo: https://github.com/Tencent/HY-World
```

**Skim-tier paper page** is shorter — frontmatter + TL;DR + Sources is enough. No Method/Results section.

### 6.2 Author

`wiki/authors/<Name>.md`

```markdown
---
type: author
name: Yann LeCun
orgs: [Meta, NYU]
papers_count: 3
first_seen: 2026-03-12
last_seen: 2026-04-17
tags: [author]
---

# Yann LeCun

Affiliations: [[Meta]], [[NYU]]

## Papers
- [[2604.14268 - HY-World 2.0]] — co-author
- [[2604.11707 - Representations Before Pixels]] — co-author

## Themes
Recurring topics across this author's papers (filled in once they have 2+ papers).
```

### 6.3 Org

`wiki/orgs/<Org>.md`

```markdown
---
type: org
name: Tencent
papers_count: 2
first_seen: 2026-04-17
tags: [org]
---

# Tencent

## Papers
- [[2604.14268 - HY-World 2.0]]

## People
- [[Author One]]
- [[Author Two]]

## Themes
What this org tends to publish on.
```

### 6.4 Model

`wiki/models/<Model>.md`

```markdown
---
type: model
name: HY-World 2.0
org: Tencent
modality: [image, video, 3d]
parameters: ~13B
released: 2026-04
tags: [model]
---

# HY-World 2.0

## What it is
One paragraph.

## Papers
- [[2604.14268 - HY-World 2.0]] — introduces

## Lineage
- Predecessors: [[HY-World 1.0]] (if known)
- Compared against: [[Sora 2]], [[Veo 3]]

## Notes
Strengths, weaknesses, open questions.
```

### 6.5 Method

`wiki/methods/<Method>.md`

```markdown
---
type: method
name: GRPO
also_known_as: [Group Relative Policy Optimization]
category: [reinforcement learning, post-training]
tags: [method]
---

# GRPO

## What it is
One-paragraph technical summary.

## Papers
- [[2604.15308 - RAD-2]] — uses GRPO at scale
- ...

## Variants & related
- [[DPO]] — earlier preference-optimization method
- [[PPO]] — predecessor
```

### 6.6 Dataset

`wiki/datasets/<Dataset>.md`

```markdown
---
type: dataset
name: MMLU
size: 15908 questions
domain: general knowledge
tags: [dataset]
---

# MMLU

## What it is
One-paragraph summary.

## Papers using it
- [[X]]
- [[Y]]
```

### 6.7 Benchmark

`wiki/benchmarks/<Benchmark>.md`

```markdown
---
type: benchmark
name: GSM8K
domain: math reasoning
tags: [benchmark]
---

# GSM8K

## What it measures
One paragraph.

## Recent SOTA
| Date | Model | Score | Source |
| --- | --- | --- | --- |
| 2026-04 | [[HY-World 2.0]] | 92.1 | [[2604.14268 - HY-World 2.0]] |

## Papers
- [[X]]
```

### 6.8 Topic

`wiki/topics/<Topic>.md`

```markdown
---
type: topic
name: World Models
related: [[3D Generation]], [[Video Generation]]
tags: [topic]
---

# World Models

## What it is
One-paragraph framing.

## Key papers
- [[2604.14268 - HY-World 2.0]] — Tencent, multi-modal
- ...

## Open questions
- ...
```

### 6.9 Trend (on-demand only)

`wiki/trends/<period>.md`

```markdown
---
type: trend
period: 2026-W16
date_range: 2026-04-13 to 2026-04-19
papers_count: 27
tags: [trend, weekly]
---

# Week 16 of 2026 (Apr 13–19)

## Themes
- World models surging: [[2604.14268 - HY-World 2.0]], ...
- ...

## Hot papers (by upvotes)
- ...

## Notable shifts / contradictions
- ...

## New entities introduced this week
- Authors: [[New Author]]
- Models: [[NewModel]]
- Methods: [[NewMethod]]
```

---

## 7. On-demand commands

Beyond `ingest`, recognize these user prompts:

| Command | Action |
| --- | --- |
| `query <question>` | Read `wiki/index.md`, drill into relevant pages, answer with `[[wikilink]]` citations and source links. **Good answers may be filed back to `wiki/` as new pages** (a comparison, an analysis, etc.) — propose this to the user before doing it. Always log the query. |
| `lint` | Full health check: orphan pages (no inbound links), missing entity pages (mentioned in prose but no page), contradictions across pages, stale claims newer sources have superseded, name collisions / near-duplicates, frontmatter consistency. Output to `wiki/lint-reports/<date>.md` with proposed fixes. Don't auto-apply fixes — ask the user. |
| `digest week <YYYY-Wnn>` | Write or refresh `wiki/trends/<YYYY-Wnn>.md` covering that week's ingests. Read `wiki/log.md` for the period and the underlying paper pages. |
| `digest month <YYYY-MM>` | Same, monthly granularity. |
| `compare <A> <B>` | Generate a comparison page (likely a table). Filed to `wiki/papers/<slug>.md` or `wiki/topics/<slug>.md` depending on what's being compared. |
| `deep <arxiv_id>` | Promote a previously-skimmed paper to deep treatment: re-read the PDF, expand the paper page, propagate richer info to entity pages. |
| `skim <arxiv_id>` | Override during triage: demote a deep pick to skim before processing. |

Every command appends to `wiki/log.md`.

---

## 8. Style for prose

- Be specific. Don't write "this paper introduces a novel method." Write "this paper introduces GRPO, a variant of PPO that drops the value network."
- Link aggressively. Any time you mention something that has (or should have) a wiki page, wrap it in `[[ ]]`.
- Numbers and benchmark scores are first-class. Include them in TL;DR / Results.
- When in doubt about whether to create a new entity page, do it. A stub is cheap; a missing page is expensive.
- Keep entity pages factual and reference-heavy; keep paper pages a mix of factual + your synthesis under "Notes & Open Questions".
- Don't fabricate. If you don't know an author's affiliation, leave it blank rather than guess.

---

## 9. Quick reference: a typical `ingest` session

```
User: ingest today

You: [reads raw/clippings/2026-04-17.md]
     [proposes triage table — 6 deep, 21 skim]

User: deep 2604.14228   # Claude Code paper looks juicy
      go

You: [for each paper, in upvote order:
        fetch HF page, save page.md + meta.yaml + assets/
        curl arXiv PDF
        read at chosen depth
        write wiki/papers/X.md
        upsert authors/orgs/models/methods/datasets/benchmarks/topics
     ]

You: [updates wiki/index.md]
     [appends wiki/log.md]
     [runs inline lint, fixes obvious issues, surfaces the rest]
     [stale-check: notes that 2604.15308 contradicts a claim in [[GRPO]], updates both]

You: "Done. Ingested 27 papers (7 deep, 20 skim). 4 new authors, 1 new org
      ([[DeepGlint]]), 2 new methods. One contradiction flagged on [[GRPO]].
      Highlights: HY-World 2.0 is a serious contender to [[Sora 2]];
      RAD-2's generator-discriminator framing extends our [[GRPO]] thread."
```

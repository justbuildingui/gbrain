---
name: deep-ingest
version: 1.0.0
description: |
  Deep-ingest podcast transcripts, interview transcripts, and long-form essays into
  the brain at memo quality. Produces per-source summaries detailed enough to
  impersonate the guest in a business meeting, then builds cross-cutting pattern
  analysis across the corpus. This is a more specific skill than media-ingest:
  use it when author worldview, investment philosophy, or intellectual framework
  matters — not for videos, YouTube links, PDFs, or screenshots (those go to
  media-ingest).
triggers:
  - "ingest this podcast"
  - "ingest this transcript"
  - "deep ingest"
  - "ingest this essay"
  - "add this to the brain"
  - podcast or interview transcript file dropped
tools:
  - read
  - write
  - edit
  - exec
  - search
  - query
  - get_page
  - put_page
  - add_link
  - add_timeline_entry
mutating: true
---

# Deep Ingest

> The goal is not a summary. The goal is a proxy. Someone reading only this page should be able to represent this person credibly in a room.

## Contract

This skill guarantees:
- Every ingested source produces a page that passes the **5-Question Quality Test** (see below)
- All 8 content categories are populated — no skipped sections
- Episode file filed under `media/podcasts/{series-name}/{guest-slug}.md`
- People page created/updated at `people/{guest-slug}.md` with back-link to episode file
- Both pages have bidirectional back-links to each other
- Every claim has an inline `[Source: ...]` citation
- Additional entities (people, companies, funds) mentioned are detected and flagged for enrichment
- Cross-cut files updated when 3+ episodes in a series are ingested
- `gbrain sync` + `gbrain embed --stale` run after every write
- Changes committed and pushed to git

## The 5-Question Quality Test (MANDATORY GATE)

Before marking any source complete, verify all five:

1. **Voice memo** — Could someone write a 2-page investment memo *in this person's voice*? Not summarizing them — *as* them?
2. **Evaluation prediction** — Could someone predict how this person would evaluate a specific business opportunity they've never seen?
3. **Preference discrimination** — Could someone identify which of two businesses this person would prefer and articulate *why*, using their own frameworks?
4. **Intellectual lineage** — Could someone name the 3 books and/or mentors that most shaped this person's thinking?
5. **Mistake & evolution** — Could someone explain this person's biggest mistake and what they concretely learned from it?

**If any answer is "no" → the summary is incomplete. Go back and expand.**

---

## Phases

### Phase 1: Pre-flight

1. Read `skills/RESOLVER.md` to confirm deep-ingest is the right skill (not media-ingest, not idea-ingest).
2. Run `gbrain search "{guest name}"` to check for an existing brain page.
3. If a page exists, read it — carry forward what's already there; don't duplicate.
4. Identify: guest name, series name, episode title, source URL or file path.
5. Derive slugs: `{guest-slug}` (e.g., `erik-serrano`), `{series-name}` (e.g., `invest-like-the-best`).

**Raw transcript location for ILTB:** `/data/brain/sources/invest-like-the-best/_raw-transcripts/`
When processing ILTB episodes, read the raw transcript from that folder.

**Output file locations (MANDATORY — do not use sources/ for output):**
- Episode content page: `media/podcasts/invest-like-the-best/{guest-slug}.md`
- People page: `people/{guest-slug}.md`

These are NOT interchangeable. `sources/` is for raw data only. Per `_brain-filing-rules.md`: content with a primary subject (a person, an episode) does NOT go in `sources/`.

### Phase 2: Read the full source

Read the entire transcript or essay. Do not skim, summarize during reading, or stop early.

- For a transcript file: `read /path/to/transcript.md` (use offset/limit for large files)
- For a URL: `web_fetch {url}` then read in full
- For a very long source: read in chunks, keeping a running mental model

**Do not begin writing until the full source is read.**

### Phase 3: Write the deep summary (8 categories)

Populate all 8 sections. Every factual claim gets `[Source: {series}, {guest}, {date}]`.

#### 1. Background & What Shaped His/Her Thinking
- Career path, formative experiences, origin story of the firm or idea
- Personal context (family, adversity, inflection moments) when mentioned
- Key failure that preceded the current approach

#### 2. Influences & Sources
- Named books, named mentors, named firms they studied
- Frameworks or people they explicitly credit
- If no books are cited, say so explicitly ("No published books cited; learning is empirical.")

#### 3. Core Thesis (Precise Enough to Execute)
- The specific thesis — not "they invest in PE" but *exactly* what they do and why
- The market failure or structural gap they exploit
- What makes the thesis durable (structural, not cyclical)

#### 4. Business Model Preferences
- What kinds of businesses/assets they love and why
- What they avoid — explicitly stated dislikes or red flags
- The criteria they use: margins, moat type, scale, sector

#### 5. Capital Strategy
- How they structure capital (leverage, lockup, co-invest, GP stake, etc.)
- Specific numbers when shared: AUM, deployment pace, average check, hold period
- Return attribution if discussed

#### 6. Key Frameworks (Named Mental Models)
- Every named framework or mental model, with a brief description
- Quote the name they gave it, if any
- Include the logic: *why* does this framework work according to them?

#### 7. Specific Claims & Data
- Every statistic, benchmark, or concrete data point
- Format: claim, then `[Source: ...]`
- Include claims that seem bold or counterintuitive — these are the highest-value signals

#### 8. Mistakes & Evolved Views
- Named mistakes: what happened, what changed
- Explicitly evolved views: "I used to think X, now I think Y"
- Signal for genuine intellectual honesty vs. performance of humility

### Phase 4: Apply the 5-Question Quality Test

Work through all five questions:

```
Q1 (Voice): Can I write a 2-page memo as them? YES / NO → [expand if no]
Q2 (Prediction): Can I predict their view on a novel deal? YES / NO → [expand if no]
Q3 (Discrimination): Can I choose between two businesses in their voice? YES / NO → [expand if no]
Q4 (Lineage): Can I name their 3 key books/mentors? YES / NO → [expand if no]
Q5 (Mistake): Can I explain their biggest mistake + lesson? YES / NO → [expand if no]
```

If any NO: return to the transcript and pull the missing detail. Do not declare done until all five pass.

### Phase 5: Create or update the episode file

**Path:** `media/podcasts/{series-name}/{guest-slug}.md`

- ILTB specifically: `media/podcasts/invest-like-the-best/{guest-slug}.md`
- If file exists: merge — don't overwrite sections that are already better; append new data
- If file is new: write with full frontmatter (see Output Format below)
- Include back-link to people page: `**People page:** [[people/{guest-slug}]]`

After writing: `gbrain sync --repo /data/brain --no-pull`

### Phase 6: Create or update the people page

**Path:** `people/{guest-slug}.md`

If the people page doesn't exist yet, create it with:
- Title, firm, role
- Back-link to the episode file: `[[media/podcasts/{series-name}/{guest-slug}]]`
- Key identifying facts (2-3 sentences)
- Timeline entry for the episode date

If the page exists: add the back-link and a timeline entry.

Run `gbrain add-link people/{guest-slug} media/podcasts/{series-name}/{guest-slug}` if the CLI supports it, otherwise ensure the back-link is in the markdown.

### Phase 7: Entity detection and enrichment

Scan the deep summary for:
- Named people (guests, mentors, investors, executives)
- Named companies (firms, portfolio companies, competitors)
- Named funds or vehicles

For each entity:
1. `gbrain search "{entity name}"` — does a page exist?
2. If yes: add a timeline entry and back-link to this episode
3. If no: flag for enrichment with a `[ENRICH: {entity name}]` marker in the episode file, or spawn the enrich skill if the entity is Tier 1 (key person in Sean's world)

**Do not block phase completion on enrichment** — flag and continue.

### Phase 8: Cross-cuts (trigger: 3+ ingested episodes in series)

Check if 3 or more episodes in this series are now in the brain.

If yes:
- Open or create `media/podcasts/{series-name}/_cross-cuts/{theme}.md`
- Themes to check/update: `investment-philosophy.md`, `fund-building.md`, `operator-vs-investor.md`, `capital-strategy.md`, `risk-and-mistakes.md` (add new themes as they emerge)
- Cross-cut format:
  - **Name the pattern explicitly** (e.g., "Concentration beats diversification")
  - **Credit sources by name** with episode references: `Erik Serrano (ILTB, 2024-01-02), Justin Ishbia (ILTB, 2023-12-19)`
  - **Note disagreements explicitly** — if 2 of 10 guests disagree, say so and name them
  - Do NOT smooth over dissent or force consensus

After 10+ episodes in a series, check if a meta-synthesis is warranted:
- Path: `concepts/{series-name}-meta-synthesis.md`
- Only write this when there's genuine cross-cutting signal that spans operator types (investors + founders + operators all saying the same thing unprompted)

### Phase 9: Sync and embed

```bash
gbrain sync --repo /data/brain --no-pull
gbrain embed --stale
```

Both commands must succeed before Phase 10.

### Phase 10: Commit and push

```bash
cd /data/brain && git add -A && git commit -m "deep-ingest: {guest name} ({series})" && git push
```

Confirm commit hash. Report to user.

---

## Output Format

### Episode file frontmatter

```yaml
---
title: "{Guest Name} — {Episode Title} ({Series Abbreviation})"
source_url: "{url}"
published: "{YYYY-MM-DD}"
guest: "{Guest Name}"
firm: "{Firm Name}"
series: "{Full Series Name}"
host: "{Host Name}"
created: {YYYY-MM-DD}
tags:
  - {series-slug}
  - deep-ingest
  - {relevant-theme-tags}
themes:
  - {thematic-category}
---
```

### Episode file structure

```markdown
# {Guest Name} — {Episode Title} ({Series})

**People page:** [[people/{guest-slug}]] | {Role}, {Firm}
**Raw transcript:** [[sources/invest-like-the-best/_raw-transcripts/{filename}]]

---

## Background & What Shaped His/Her Thinking
{Narrative, 200-400 words, with [Source: ...] citations}

## Influences & Sources
{Bulleted list of named people, books, firms with brief explanation of influence}

## Core Thesis (Precise Enough to Execute)
{Paragraph — specific, executable, includes market structure context}

## Business Model Preferences
{Numbered list of preferences with reasoning}

## Capital Strategy
{Narrative + specific numbers when available}

## Key Frameworks (Named Mental Models)
{Numbered list: framework name (bold) + explanation}

## Specific Claims & Data
{Bulleted list: stat — [Source: ...]}

## Mistakes & Evolved Views
{Numbered list: mistake → what changed → what they believe now}
```

### People page structure

```markdown
# {Guest Name}

**Role:** {Title}
**Firm:** {Firm Name}
**Episode:** [[media/podcasts/{series-name}/{guest-slug}]]

{2-3 sentence summary — who they are and what they're known for}

## Timeline
- {YYYY-MM-DD}: Featured on {series} — [[media/podcasts/{series-name}/{guest-slug}]]
```

---

## Anti-Patterns

### ❌ Summary so thin it fails the quality test

**NOT GOOD ENOUGH:**
> "Erik Serrano runs Stable Asset Management, which backs emerging investment managers. He focuses on GP staking and believes in backing good founders."

**GOOD:** Write the full thesis precisely — "Back early-stage investment firm founders by taking a minority GP stake *and* providing LP capital, then help them build the business operationally. Earn returns across three streams: (1) LP returns, (2) excess management fees, (3) enterprise value appreciation of the GP itself (monetized when the founder buys back the stake). Structure 3-year lockups..."

---

### ❌ Bullet-pointing instead of frameworks

**NOT GOOD ENOUGH:**
> "He thinks resilience matters. He looks for variant perception."

**GOOD:** Name the framework and explain the logic — "**Resilience + Variant Perception**: Two-trait underwriting framework. Resilience (never gives up; traces to childhood adversity) and variant perception (contrarian view from life experience). Key correction: *excess* variant perception becomes stubbornness — the optimal is high-but-not-extreme. He used to select for maximum contrarianism; now he corrects for founders who can't update on new data."

---

### ❌ Skipping the quality test gate

Never declare a summary done without running all 5 questions explicitly. If you skip the test, you don't know what's missing.

---

### ❌ Using deep-ingest for media that belongs in media-ingest

**This skill is for:** podcast transcripts, interview transcripts, long-form essays where the author's worldview is the asset.

**Use media-ingest instead for:** YouTube links, video files, audio files, PDFs, book PDFs, screenshots, GitHub repos.

**The key question:** Is the goal to capture *what this person thinks and how they think*? → deep-ingest. Is the goal to extract content from a media artifact? → media-ingest.

---

### ❌ Papering over disagreements in cross-cuts

Cross-cuts must name disagreements explicitly. If 2 of 10 guests think something different, say: "Dissent: Reed Hastings (ILTB, 2024-03-01) and Brent Beshore (ILTB, 2023-11-15) disagree — they argue X instead." Do not omit the minority view or falsely imply consensus.

---

### ❌ Entity mentions without back-links

Every person or company with a brain page that appears in a deep-ingest episode MUST have a back-link added. This is the Iron Law. "I'll do it later" is not acceptable — flag with `[ENRICH: name]` if you can't do it now, so it's visible.

---

### ❌ Committing without syncing first

Always run `gbrain sync` and `gbrain embed --stale` before `git commit`. Committing unsynced pages breaks the index.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `gbrain search "{name}"` | Pre-flight: check for existing page |
| `gbrain query "{question}"` | Pre-flight: find related pages |
| `read` / `web_fetch` | Read full source transcript or essay |
| `write` / `edit` | Create or update episode and people pages |
| `gbrain sync --repo /data/brain --no-pull` | Sync after every write |
| `gbrain embed --stale` | Generate embeddings for new/updated pages |
| `cd /data/brain && git add -A && git commit -m "..." && git push` | Commit and push changes |
| enrich skill | Tier 1 entity enrichment (spawn if key person) |

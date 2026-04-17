---
name: batch-ingest
version: 1.0.0
description: |
  Batch orchestration for deep-ingest. Processes multiple transcripts from the same
  series, running Phases 1-7 of deep-ingest per episode, then runs cross-cuts ONCE
  at the end of the entire batch. Prevents redundant cross-cut rebuilds after each
  individual episode.
triggers:
  - "batch ingest"
  - "ingest all transcripts"
  - "bulk ingest"
  - "ingest the batch"
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

# Batch Ingest

> Same quality as deep-ingest. Same 5-question gate. Same 8 categories.
> The only difference: cross-cuts run once at the end, not after every episode.

## Contract

This skill guarantees:
- Every episode passes the full deep-ingest quality standard (5-Question Quality Test, all 8 categories)
- Phases 1-7 of `skills/deep-ingest/SKILL.md` run per episode — no shortcuts
- Cross-cuts (Phase 8) are **deferred** until the entire batch is complete
- Cross-cuts run exactly once at the end, with the full corpus available
- Sync, embed, commit happen per episode (Phases 9-10) so progress is saved incrementally
- A batch manifest tracks completed vs remaining episodes
- If the batch is interrupted, it can resume from where it left off

## Phases

### Phase 1: Build the batch manifest

1. Identify the source directory (e.g., `/data/brain/sources/invest-like-the-best/_raw-transcripts/`)
2. List all transcript files
3. Check which episodes already have completed pages in `media/podcasts/{series-name}/`
4. Build a manifest: `{series-name}-batch-manifest.json` saved to `/tmp/`

```json
{
  "series": "invest-like-the-best",
  "source_dir": "/data/brain/sources/invest-like-the-best/_raw-transcripts/",
  "total": 400,
  "completed": ["erik-serrano", "justin-ishbia"],
  "remaining": ["guest-3", "guest-4", ...],
  "cross_cuts_done": false
}
```

### Phase 2: Per-episode loop

For each episode in `remaining`:

Run **deep-ingest Phases 1-7** exactly as written in `skills/deep-ingest/SKILL.md`:
- Phase 1: Pre-flight (search for existing page, derive slugs)
- Phase 2: Read the full transcript
- Phase 3: Write the deep summary (all 8 categories)
- Phase 4: Apply the 5-Question Quality Test (MANDATORY — no skipping)
- Phase 5: Create/update the episode file
- Phase 6: Create/update the people page
- Phase 7: Entity detection and enrichment

**SKIP Phase 8 (cross-cuts).** This is the entire point of batch-ingest.

Then run Phases 9-10 per episode:
- Phase 9: `gbrain sync --repo /data/brain --no-pull && gbrain embed --stale`
- Phase 10: `cd /data/brain && git add -A && git commit -m "batch-ingest: {guest name} ({series})" && git push`

Update the manifest: move the episode slug from `remaining` to `completed`.

### Phase 3: Cross-cuts (ONCE, after entire batch)

Only after all episodes in the batch are complete:

1. Count total episodes now in the series
2. Run deep-ingest Phase 8 in full:
   - Open or create `media/podcasts/{series-name}/_cross-cuts/{theme}.md`
   - Themes: `investment-philosophy.md`, `fund-building.md`, `operator-vs-investor.md`, `capital-strategy.md`, `risk-and-mistakes.md` (add new themes as they emerge)
   - Name patterns explicitly, credit sources by name with episode references
   - Note disagreements explicitly — do not smooth over dissent
3. If 10+ episodes: check if meta-synthesis is warranted at `concepts/{series-name}-meta-synthesis.md`
4. Final sync, embed, commit:
   ```bash
   gbrain sync --repo /data/brain --no-pull
   gbrain embed --stale
   cd /data/brain && git add -A && git commit -m "batch-ingest: cross-cuts for {series}" && git push
   ```
5. Update manifest: set `cross_cuts_done: true`

## Resumability

If the batch is interrupted (session ends, timeout, error):
- Read the manifest from `/tmp/{series-name}-batch-manifest.json`
- Skip episodes already in `completed`
- Resume from the next episode in `remaining`
- Cross-cuts only run if all episodes are complete and `cross_cuts_done` is false

## Output Format

Per episode: same as deep-ingest (episode file + people page + entity flags).

End of batch: cross-cut files + optional meta-synthesis.

Final report:
```
## Batch Ingest Complete — {Series Name}

| Metric | Count |
|--------|-------|
| Episodes ingested | N |
| People pages created/updated | N |
| Entities flagged for enrichment | N |
| Cross-cut themes | N |
| Meta-synthesis | yes/no |

### Cross-cut themes written
- {theme}: {N} episodes contributed
- ...

### Episodes with quality test issues
- {any that needed expansion — should be 0}
```

## Anti-Patterns

- **Running cross-cuts after each episode** — the entire point of this skill is to defer them
- **Skipping the 5-Question Quality Test** — batch does not mean lower quality
- **Not saving progress incrementally** — commit after each episode so interruptions don't lose work
- **Starting cross-cuts before all episodes complete** — partial cross-cuts produce incomplete pattern analysis
- **Not updating the manifest** — without it, resumability breaks
- **Using this for a single episode** — use deep-ingest directly for one-offs

## Tools Used

| Tool | Purpose |
|------|---------|
| `read` | Read transcript files |
| `write` / `edit` | Create/update episode and people pages |
| `gbrain search "{name}"` | Pre-flight entity checks |
| `gbrain query "{question}"` | Related page discovery |
| `gbrain sync --repo /data/brain --no-pull` | Sync after each write |
| `gbrain embed --stale` | Embed new/updated pages |
| `git add -A && git commit && git push` | Save progress incrementally |

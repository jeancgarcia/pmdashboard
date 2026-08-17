# Claude Routines — reference copies

The dashboard's daily data is produced by scheduled Claude Code Routines
(managed in claude.ai → Claude Code → Routines, account jegarcia). Routines
created from the UI/API can NOT be edited by an agent session — changes must
be pasted into the Routine editor by a person. The files in this folder are
the version-controlled source of truth for those prompts; when you change a
Routine, update its file here in the same breath.

| Routine | Schedule (UTC) | Writes | Prompt file |
|---|---|---|---|
| CoE Reviewer | `5 12 * * 1-5` (8:05 AM Santo Domingo) | Supabase `app_state` rows 4 (live) + 5 (30-day history) — read by the dashboard's CoE tab | `coe-reviewer-prompt.md` |
| EPMO Daily Brief | `5 12 * * 1-5` | Supabase rows 6 + 7 via `scripts/generate_epmo_digest.py` — read by the EPMO tab | prompt lives in the Routine (drives the script; the script is already versioned) |
| Daily validation: PMO New Project Intakes vs CoE | `30 11 * * *` | Asana comments only | currently paused (no runs recorded) |

## Changelog

### 2026-08-17 — coe-reviewer-prompt.md (pending paste into the Routine)

1. **byStage now spans ALL portfolio projects, terminal stages included.**
   The old prompt said only "byStage, byPriority, …: counts"; the routine
   computed every map over active projects, so `summary.byStage` never
   contained Completed/Deployed and the Pipeline widget showed them as 0.
   (The dashboard now also recounts stages from the `projects` array, so the
   widget is correct either way — this fixes the data at the source.)
   byPriority/byDepartment/byClassification/byFirstReview stay active-only
   on purpose: they back the active-workload widgets.
2. **`delta.previousByStage` added** (previous run's byStage, verbatim).
   The Pipeline widget's per-stage +/- chips read this field; it was never
   produced, so the chips never rendered. They start appearing on the second
   run after the paste.
3. **`updatedAt` is now timezone-aware UTC** (`datetime.now(timezone.utc)`).
   The naive timestamp was parsed as local time by the browser, showing the
   "last updated" line 4 hours late.

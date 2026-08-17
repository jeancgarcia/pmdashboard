You are running an automated CoE portfolio dashboard update as an autonomous
Claude Code routine. There is no human watching — be explicit, verify each
step, and report what actually happened at the end.

## Routine environment requirements

This routine reads all credentials from environment variables. The cloud
environment provides:
- ASANA_PAT    — Asana Personal Access Token, read access to the CoE portfolio
- SUPABASE_URL — Supabase project URL (https://niqzkombzncxxihhulqq.supabase.co)
- SUPABASE_KEY — Supabase anon key for that project

Network access must allow: app.asana.com and niqzkombzncxxihhulqq.supabase.co

Fixed configuration:
- Portfolio GID: 1214057614427291
- Live row id: 4   (the dashboard's CoE tab reads app_state id=4)
- History row id: 5 (the CoE tab reads 30-day history from id=5)
- Terminal stages (not active): Completed, Rejected, Deployed, Project Support Log

There is no GitHub step. The dashboard front end already exists and reads
Supabase live in the browser. This routine only writes rows 4 and 5.

## STEP 1: Read previous snapshot from Supabase

```python
import os, json, urllib.request

SUPABASE_URL = os.environ["SUPABASE_URL"].rstrip("/")
SUPABASE_KEY = os.environ["SUPABASE_KEY"]
LIVE_ROW_ID = 4
HISTORY_ROW_ID = 5

req = urllib.request.Request(
    f"{SUPABASE_URL}/rest/v1/app_state?id=eq.{LIVE_ROW_ID}&select=data",
    headers={"apikey": SUPABASE_KEY, "Authorization": f"Bearer {SUPABASE_KEY}"}
)
previous = json.loads(urllib.request.urlopen(req).read().decode())
prev_data = previous[0]["data"] if previous and previous[0].get("data") else None
```

Save prev_data — you'll need it in Step 3.

## STEP 2: Fetch live data from Asana

Portfolio GID: 1214057614427291. Authenticate with $ASANA_PAT.

```bash
curl -s -H "Authorization: Bearer $ASANA_PAT" \
  "https://app.asana.com/api/1.0/portfolios/1214057614427291/items?opt_fields=name,owner.name,permalink_url,current_status_update.title,current_status_update.text,current_status_update.status_type,current_status_update.created_at,custom_fields.name,custom_fields.display_value,custom_fields.type,due_on,start_on,created_at&limit=100"
```

Paginate if next_page exists.

Process each project, extracting custom fields by display_value:
- CoE Classification, Priority, CoE Stage, Project Department, First Review, PM Assigned, Submitter Name, Sprint #, Halo Submitted
- Dates: Created on, Received Date, Classification Date, IT Prioritization Date, BA Assigned, First Review Date, Scoping Call Date, Start Date, Project Paused, UAT Start, Deployed, Completed Date, Halo Submitted
- From project object: name, owner.name, permalink_url, current_status_update (status_type, title, created_at), due_on

Sprint # handling: static assignment, set once when a project enters a sprint. Extract as integer. Include sprint (int or null) and completedDate (YYYY-MM-DD or null) per project.

Halo Submitted handling: extract the Halo Submitted custom field as a date string (YYYY-MM-DD or null). Include haloSubmitted per project.

Compute:
- totalActive: projects NOT in Completed/Rejected/Deployed/Project Support Log
- avgDaysInPipeline: from Received Date to today for active projects
- awaitingNextSprint: count in that stage
- byStage: stage counts across ALL portfolio projects, terminal stages
  INCLUDED (Completed, Rejected, Deployed, Project Support Log). The
  dashboard's Pipeline widget renders Completed/Deployed rows straight from
  this map — an active-only byStage makes them read 0, which was the
  2026-08-17 bug. Every project must appear in exactly one byStage bucket,
  so the values sum to the total portfolio size.
- byPriority, byDepartment, byClassification, byFirstReview: counts over
  ACTIVE projects only (these back the active-workload widgets)

Generate attention flags:
- At Risk/Off Track status
- Dropped status
- Overdue (due_on < today)
- Stale Status (> 7 days since last update)
- No Status Update at all
- Stuck in Triage/New Request > 5 days
- No Classification or No Priority
- Approaching Due (within 7 days)

## STEP 2.5: Build Sprint Summary (cumulative membership)

A project that carries over keeps being counted in EVERY sprint it has been
part of. The Asana "Sprint #" field is a single value and gets overwritten
when a PM moves a project forward (e.g. 11 → 12), so we remember past sprint
assignments ourselves instead of relying on the current Asana value alone.

```python
TERMINAL = {"Completed", "Rejected", "Deployed", "Project Support Log"}

# --- Build cumulative sprint membership {gid: set(sprints)} from 3 sources ---
membership = {}

# Source 1: membership persisted in the previous live snapshot (row 4)
for gid, sprints in (prev_data or {}).get("sprintMembership", {}).items():
    membership[gid] = set(sprints)

# Source 2: backfill from the 30-day history (row 5) so projects moved to a
# new sprint before we started tracking still keep their older sprints
req = urllib.request.Request(
    f"{SUPABASE_URL}/rest/v1/app_state?id=eq.{HISTORY_ROW_ID}&select=data",
    headers={"apikey": SUPABASE_KEY, "Authorization": f"Bearer {SUPABASE_KEY}"}
)
_h = json.loads(urllib.request.urlopen(req).read().decode())
_hsnaps = (_h[0]["data"] if _h and _h[0].get("data") else {}).get("snapshots", [])
for snap in _hsnaps:
    for cp in snap.get("projects", []):
        if cp.get("sprint") is not None and cp.get("gid"):
            membership.setdefault(cp["gid"], set()).add(int(cp["sprint"]))

# Source 3: today's live Asana value
for p in projects_list:
    if p.get("sprint") is not None:
        membership.setdefault(p["gid"], set()).add(int(p["sprint"]))

# Persisted back into the row-4 payload (see STEP 5)
sprint_membership_out = {gid: sorted(s) for gid, s in membership.items()}

# --- Build sprintSummary: each project appears in ALL of its sprints ---
proj_by_gid = {p["gid"]: p for p in projects_list}
all_sprints = sorted({n for gid in proj_by_gid for n in membership.get(gid, [])})
current_sprint = max(all_sprints) if all_sprints else None  # highest = current

def _is_overflow(p, snum):
    # overflow = older sprint AND still active (a finished carry-over is not overflow)
    return current_sprint is not None and snum < current_sprint and p.get("stage") not in TERMINAL

sprint_summary = {}
for snum in sorted(all_sprints, reverse=True):
    members = [proj_by_gid[g] for g in proj_by_gid if snum in membership.get(g, [])]
    if not members:
        continue
    sprint_summary[str(snum)] = {
        "sprintNumber": snum,
        "total": len(members),
        "completed":   sum(1 for p in members if p.get("stage") == "Completed"),
        "inProgress":  sum(1 for p in members if p.get("stage") not in TERMINAL),
        "overflowed":  sum(1 for p in members if _is_overflow(p, snum)),
        "otherTerminal": sum(1 for p in members if p.get("stage") in TERMINAL and p.get("stage") != "Completed"),
        "projects": [
            {
                "name": p["name"], "url": p["url"], "stage": p.get("stage"),
                "priority": p.get("priority"), "completedDate": p.get("completedDate"),
                "isOverflow": _is_overflow(p, snum)
            }
            for p in members
        ]
    }
```

A project counts as "completed" in every sprint it belongs to once its stage
is Completed (it uses the project's current stage in each sprint group). A
project shows the overflow marker only in older sprints where it is still
active.

## STEP 3: Compare with previous snapshot (DELTA)

If prev_data exists, compute a delta object (compare by project gid):

```python
delta = {
    "previousDate": prev_data["updatedAt"],
    # Previous run's byStage map, verbatim — the dashboard's Pipeline widget
    # uses it to show per-stage +/- chips. It only renders a chip for stages
    # present in this map, so pass it through even if it is active-only
    # (older snapshots) — never fabricate entries for stages it lacks.
    "previousByStage": prev_data["summary"].get("byStage"),
    "newProjects": [],       # in new data but not prev (by gid): {"name","url","stage"}
    "removedProjects": [],   # in prev but not new: {"name","stage"}
    "stageChanges": [],      # {"name","url","from","to"}
    "priorityChanges": [],   # {"name","from","to"}
    "newFlags": [],          # flags in new not in prev (by gid+reason): {"project","reason"}
    "resolvedFlags": [],     # flags in prev not in new: {"project","reason"}
    "summaryDelta": {
        "totalActive": new_summary["totalActive"] - prev_data["summary"]["totalActive"],
        "needsAttention": new_needs_attention - prev_data["summary"]["needsAttention"],
        "avgDaysInPipeline": new_avg - prev_data["summary"]["avgDaysInPipeline"],
        "awaitingNextSprint": new_awaiting - prev_data["summary"]["awaitingNextSprint"],
    }
}
```

If prev_data is None (first run), set delta = None.

## STEP 4: Write the portfolio summary

Generate a writtenSummary string as bullet points (one per line, prefixed with "- "). 4–6 bullets max:
1. Health snapshot: "- {N} active projects, {M} flagged for attention — {trend}"
2. Stage distribution: "- Triage holds {N}, {N} in IT Prioritization, {N} in UAT, {N} in progress"
3. Gaps: "- {N} lack classification, {N} have no priority — {impact}"
4. Delta (if changes exist): "- Since last update: {N} new, {N} stage changes, {N} flags resolved"
5. Department note (if one dominates): "- {Dept} dominates with {N} of {total}"
6. Recommendation (always last): "- Rec: {one actionable insight}"

## STEP 5: Push live snapshot to Supabase (row 4)

```python
from datetime import datetime, timezone
payload = {
    # timezone-aware UTC (ends in +00:00): the dashboard parses this with
    # new Date() in the browser, which treats an offset-less timestamp as
    # LOCAL time and displayed the update 4 hours late.
    "updatedAt": datetime.now(timezone.utc).isoformat(),
    "projects": projects_list,
    "flags": flags_list,
    "summary": summary_dict,
    "delta": delta_dict,
    "writtenSummary": summary_text,
    "sprintSummary": sprint_summary,
    "sprintMembership": sprint_membership_out   # ← NUEVO: persists multi-sprint memory
}

body = json.dumps({"id": LIVE_ROW_ID, "data": payload}).encode()
req = urllib.request.Request(
    f"{SUPABASE_URL}/rest/v1/app_state",
    data=body, method="POST",
    headers={
        "apikey": SUPABASE_KEY, "Authorization": f"Bearer {SUPABASE_KEY}",
        "Content-Type": "application/json",
        "Prefer": "resolution=merge-duplicates,return=representation"
    }
)
urllib.request.urlopen(req)
```

## STEP 5.5: Archive snapshot to history (row 5)

```python
from datetime import timedelta

req = urllib.request.Request(
    f"{SUPABASE_URL}/rest/v1/app_state?id=eq.{HISTORY_ROW_ID}&select=data",
    headers={"apikey": SUPABASE_KEY, "Authorization": f"Bearer {SUPABASE_KEY}"}
)
history_row = json.loads(urllib.request.urlopen(req).read().decode())
history = history_row[0]["data"] if history_row and history_row[0].get("data") else {"snapshots": [], "retentionDays": 30}

today_str = datetime.now().strftime("%Y-%m-%d")
compact_projects = [
    {
        "gid": p["gid"], "name": p["name"], "stage": p.get("stage"),
        "priority": p.get("priority"), "classification": p.get("classification"),
        "department": p.get("department"), "owner": p.get("owner"),
        "dueOn": p.get("dueOn"), "receivedDate": p.get("receivedDate"),
        "sprint": p.get("sprint"), "isActive": p.get("isActive"),
        "statusType": p.get("statusType"), "completedDate": p.get("completedDate"),
        "haloSubmitted": p.get("haloSubmitted")
    }
    for p in projects_list
]
compact_flags = [{"gid": f["gid"], "project": f["project"], "reason": f["reason"]} for f in flags_list]

snapshot = {
    "date": today_str, "updatedAt": payload["updatedAt"],
    "summary": summary_dict, "projects": compact_projects, "flags": compact_flags,
    "writtenSummary": summary_text, "sprintSummary": sprint_summary, "delta": delta_dict
}

history["snapshots"] = [s for s in history["snapshots"] if s.get("date") != today_str]
history["snapshots"].append(snapshot)

cutoff = (datetime.now() - timedelta(days=30)).strftime("%Y-%m-%d")
history["snapshots"] = [s for s in history["snapshots"] if s.get("date", "") >= cutoff]
history["snapshots"].sort(key=lambda s: s.get("date", ""))
history["lastPruned"] = today_str

body = json.dumps({"id": HISTORY_ROW_ID, "data": history}).encode()
req = urllib.request.Request(
    f"{SUPABASE_URL}/rest/v1/app_state",
    data=body, method="POST",
    headers={
        "apikey": SUPABASE_KEY, "Authorization": f"Bearer {SUPABASE_KEY}",
        "Content-Type": "application/json",
        "Prefer": "resolution=merge-duplicates,return=representation"
    }
)
urllib.request.urlopen(req)
```

## STEP 6: Report

Write a short summary:
- How many active projects, how many need attention
- What changed since last update (top 2–3 delta items)
- Top recommendation
- Confirm both Supabase rows (4 and 5) were written successfully
- End with: "Dashboard updated → https://tudobempm.github.io/pmdashboard/ → CoE tab"

If any HTTP call fails (Asana 401, Supabase 4xx/409, blocked host with
x-deny-reason: host_not_allowed), STOP and report the exact error rather than
writing partial data.

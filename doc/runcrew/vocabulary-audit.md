# RunCrew Vocabulary Audit

**Status:** Draft, v0.1 — read alongside `design-brief.md`
**Date:** 2026-05-17
**Purpose:** identify every RunCrew noun, find existing Paperclip usages it collides with, flag risk, recommend resolution.

---

## Severity legend

- 🔴 **HIGH** — same word, different user-facing meaning. Will confuse the user or the codebase.
- 🟡 **MED** — word is reused internally (schema, code, dev docs). User won't see it, but contributors will.
- 🟢 **LOW** — no real overlap, or already aligned.

---

## 1. The seven RunCrew nouns — collision report

### 🟢 Crew — *(replaces Company in copy)*

- **Existing Paperclip uses of "crew":** none (zero matches in the codebase outside our own design brief).
- **Verdict:** clean rename, no collision.
- **Action:** none required.

### 🟡 Coordinator — *(one per crew, sole conversational point of contact)*

- **Existing Paperclip uses:**
  - `server/src/services/plugin-job-coordinator.ts` — internal class that bridges plugin lifecycle and job scheduling. Pure server-side, never user-facing.
  - Doc references in `doc/spec/agent-runs.md`, `doc/plugins/PLUGIN_SPEC.md`, etc — all internal architecture docs.
- **Verdict:** internal-only reuse. Users will never see "PluginJobCoordinator". Developers will see both meanings but the contexts are obviously different (server class vs UI agent role).
- **Action:** none required. Note in `design-brief.md` that "Coordinator" is also an unrelated internal class name.

### 🟢 Workflow — *(primary noun = Project + Routine + Goal bundle)*

- **Existing Paperclip uses:**
  - `ui/src/lib/workflow-sort.ts` — an algorithm name. "Workflow sort" = topological sort that keeps blocker chains contiguous on a board. Pure internal utility, not a user-facing noun.
  - `ui/src/pages/Routines.tsx` (lines 448, 1013) — **user-facing copy** that calls a triggered Routine a "live workflow" and a "recurring workflow." This is the *closest* existing meaning to ours, and conveniently aligned: today a Routine becomes a "workflow" when wired up; in RunCrew, a Workflow is exactly that idea, expanded to bundle Routine + Project + Goal.
  - `skills/paperclip/references/workflows.md` — internal docs file naming a set of operator playbooks "workflows." Different meaning, but internal-only.
- **Verdict:** the existing user-facing usage in Routines.tsx is *aligned with* our usage, not in conflict. The `workflow-sort` algorithm is a different concept but internal-only. No user-visible collision.
- **Action:** none required for users. Optionally rename `workflow-sort.ts` to `blocker-aware-sort.ts` (or similar) at some point to clean up developer cognition — but not blocking.

### 🟡 Task — *(replaces Issue at the UI layer)*

- **This is the one you flagged.** Real picture: more nuanced than "collision."
- **Existing Paperclip uses:**
  - **User-facing copy in upstream UI already uses "task" as a synonym for Issue:**
    - `pages/Dashboard.tsx` — "data.tasks.inProgress / open / blocked", "No tasks yet."
    - `pages/AgentDetail.tsx` — "Can assign tasks", "implicitly assign tasks"
    - `pages/CompanyExport.tsx` — exports issues under `tasks/{slug}/TASK.md` paths
    - `App.tsx` — "starter task for this company"
    - **So our Task = Issue mapping is consistent with how Paperclip already speaks to users.** No conflict here.
  - **Different internal meanings of "task" that the user will never see:**
    - `packages/db/src/schema/agent_task_sessions.ts` — tracks one row per (agent, adapter type, `taskKey`). Here "task" means an *adapter session* (e.g. a Claude Code task ID, a Codex task ID). Pure internal — schema and adapter plumbing only.
    - `pages/InstanceExperimentalSettings.tsx` — "recovery tasks" = auto-recovery jobs for heartbeat liveness incidents. Internal/admin only; will live behind Advanced.
- **Verdict:** the *user-facing* alignment is already good. The schema-level reuse (`agent_task_sessions`, "recovery tasks") is internal/admin-only and won't bleed into the SMB user experience. **No noun change needed.**
- **Action:**
  1. Keep "Task" = Issue in user-facing copy.
  2. Add an explicit note in `design-brief.md` clarifying the three distinct meanings so contributors don't conflate them.
  3. Long-term cleanup (optional): rename `agent_task_sessions` to `agent_adapter_sessions` and "recovery tasks" to "recovery jobs" so the schema doesn't pun on "task." Not blocking for design.

### ✅ Crew Template — *(preloaded crew + workflows + tools)* — RESOLVED

- **Decided 2026-05-17:** rename our concept to **"Crew Template"** (display, two words, title case) / `crewTemplate` (schema/code).
- **Why it was HIGH severity:** Paperclip already uses "Template" *user-facingly* for Mustache-style interpolation strings:
  - `components/AgentConfigForm.tsx` — **"Prompt Template"** field on every agent.
  - `components/agent-config-primitives.tsx` — `promptTemplate` and `workspaceBranchTemplate` help text.
  - `components/ProjectProperties.tsx` — **"Branch template"** for naming derived git branches.
  - `pages/CompanyEnvironments.tsx` — environment config `"template"` key.
  - `adapters/runtime-json-fields.tsx` — **"Payload template JSON"** field.
- **Why "Crew Template" works:** the qualifier "Crew" fully disambiguates from "Prompt Template" / "Branch template" / "Payload template" — different first word, different mental model. No need to invent a brand-new word (Recipe / Starter / Blueprint), which would lose the "this is a template you adopt" intuition.
- **Applied:** sidebar item = **Crew Templates**. First one = **Social Media**. Schema = `crewTemplate`. Brief and audit updated.

### 🟢 Tools — *(unified veneer over Plugins + Adapters + Company Secrets)*

- **Existing Paperclip uses:**
  - `packages/shared/src/constants.ts` and `validators/plugin.ts` — `agent.tools.register` is the plugin capability for *registering agent tools*. These are exactly the integrations a workflow uses (Google Calendar, YouTube, etc.).
- **Verdict:** the existing meaning is *aligned with* ours. Tools that plugins register are precisely the user-facing "Tools." No conflict — actually a clean mapping.
- **Action:** none required. Note that internally, "Tools" maps to plugin-registered agent tools (where `agent.tools.register` lives).

### 🟢 Inbox — *(HITL gates + read/unread; canonical home for human gates)*

- **Decided 2026-05-17:** keep upstream Paperclip's "Inbox" label (minimum drift). Earlier exploration considered "For you"; rejected in favor of minimum-drift principle.
- **Verdict:** clean — direct continuity with upstream.
- **Action:** none required. Schema-side this view is powered by existing `approvals` + `inbox` queries.

---

## 2. Paperclip nouns we did NOT rename — what surfaces under Advanced

These nouns stay in Paperclip's vocabulary because they're hidden behind the "Advanced" entry in v1. If a user opens Advanced, they will see:

| Noun | Where seen | Risk |
|---|---|---|
| Issue | Advanced view (raw issue board), AgentDetail, exports | 🟢 SMB user won't see it on primary surfaces |
| Project | Advanced project list, Routines page, settings | 🟢 Hidden from primary nav |
| Routine | Advanced routine list | 🟡 The Routines page currently uses "workflow" in its copy — slight self-overlap, but contained to Advanced |
| Goal | Advanced goals list | 🟢 Hidden |
| Approval | Advanced approvals list | 🟢 Replaced by "For you" gate cards on primary surfaces |
| Execution Workspace | Advanced only | 🟡 Jargon-heavy; consider hiding entirely or renaming if ever surfaced |
| Workspace (Project) | Advanced only | 🟡 Same |
| Heartbeat / Run | Internal only (no user-facing label outside Advanced settings) | 🟢 |
| Skill | Advanced "Skills" page | 🟡 Both AGENTS.md-style instructions and capability skills are called "skills" today — already overloaded inside Paperclip; ignore for v1 |
| Adapter | Instance settings → Adapter Manager (Advanced) | 🟢 Hidden |
| Plugin | Instance settings → Plugin Manager (Advanced) | 🟢 Hidden |
| Hire / Hiring | `CompanySettings.tsx` — Hiring section | 🟡 In RunCrew copy: "bring on a crew member" instead of "hire". Behind Advanced today; replace if it ever moves forward |
| Heartbeat | Instance settings (Advanced) | 🟢 Hidden |

---

## 3. Concrete brief updates required

Based on this audit, the following changes to `design-brief.md` are recommended:

1. ✅ **Done — Template → Crew Template** everywhere in the brief. Sidebar item: **Crew Templates**. First one: **Social Media**. Schema: `crewTemplate`.
2. **Add a clarifying note under "Task"** in §4: *"'Task' in RunCrew = Paperclip's `issues` table, which Paperclip's own UI also calls 'tasks' in many places. Two unrelated internal uses of 'task' exist (`agent_task_sessions` and 'recovery tasks') — both are admin/internal and never surface in RunCrew's primary UI."*
3. **Add a clarifying note under "Coordinator"**: *"Internally, `PluginJobCoordinator` is an unrelated server-side class. No user-facing collision."*
4. **Add a clarifying note under "Workflow"**: *"`workflow-sort.ts` is an internal algorithm name (blocker-aware topological sort). The Routines page already uses 'workflow' user-facing in a way aligned with RunCrew's meaning. No user-facing collision."*
5. **Update the forbidden vocabulary list** in §4 to call out "Hire / Hiring" as also forbidden in user-facing copy — replace with "bring on a crew member."

---

## 4. Decisions you need to make

- ✅ **Template → Crew Template** — DECIDED 2026-05-17 (Sandro). Disambiguates from Prompt/Branch/Payload Template via the "Crew" qualifier.
- ✅ **Task = Issue at user-facing layer** — DECIDED 2026-05-17 (Sandro). Aligned with upstream Paperclip's own UI usage.
- ⏸️ **Optional later cleanup:** rename `agent_task_sessions` and `workflow-sort` in the schema/code to reduce contributor cognitive load. Not blocking design.

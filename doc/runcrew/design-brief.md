# RunCrew — Design Brief

**Status:** Draft, v0.1
**Date:** 2026-05-17
**Author:** Sandro (interview synthesis)
**Source:** Grilling session with Claude, May 2026

---

## 1. Project at a glance

**RunCrew** is a fork of Paperclip, rebranded and re-architected at the UI/UX layer for a fundamentally different user. Where Paperclip is an operator-grade control plane optimized for technical users orchestrating AI agents (GitHub/Linear-flavored), RunCrew is a simplified, workflow-centric product for SMB owners who do not think in terms of agents at all.

This is a **full product launch**, not a contribution upstream. There are no existing users to migrate. The Paperclip codebase, schema, and adapter infrastructure are inherited as-is; the UI, IA, copy, and brand are net new. Upstream Paperclip is unaffected.

---

## 2. Target user

**Persona:** SMB owner or non-power-user employee at a small company.

- Uses ChatGPT (or similar LLMs) casually via chat. Not an AI power user.
- Doesn't understand "agentic AI" concepts. Doesn't want to.
- Primary motivation: **save time and money by automating workflows and optimizing processes** — not "managing agents."
- Time-poor, often anxious about complexity. Checks in periodically rather than living in the app.
- Expects software to feel like a competent colleague, not a control panel.

**Test of this brief:** if a screen reads as Linear/Jira with a different color palette, it has failed.

---

## 3. The defining equation

> **Workflow + KPI = Crew of agents executing on it.**

A workflow cannot exist without a measurable goal (#views, ROAS, leads, $ saved, reply time, etc.). The user supplies the workflow and the KPI; the system supplies the agents.

When a user adds a new workflow, the system **first tries to fulfill it with existing crew members** (extending their skills/tools if needed). Spawning a new agent is the exception path, announced to the user in plain language (*"I'll bring on a Trend Analyst — Maya doesn't have YouTube data access."*).

---

## 4. Core vocabulary

The redesign keeps Paperclip's schema and renames at the UI layer. Cross-table integrity is preserved; the user never sees the seams.

| RunCrew UI noun | Paperclip schema | Notes |
|---|---|---|
| **Crew** | `companies` | Rename in copy only. One founder may own multiple Crews. |
| **Coordinator** | (new role on `agents`) | Exactly one per Crew. Sole conversational point of contact. Uses the Coordinator archetype (think CrewAI). |
| **Crew member** | `agents` (non-coordinator) | Visible but secondary. The user never addresses non-coordinators directly. |
| **Workflow** | `projects` + `routines` + `goals` (bundled) | The primary noun. A recurring thing the crew does on the user's behalf, with a mandatory KPI. |
| **Task** | `issues` | Renamed and simplified. Schema preserved (atomic checkout, single-assignee, parent/child, blockers, runs). |
| **Sub-task** | `issues.parentId` | Rendered inline on a pipeline card. No separate tree view. |
| **Run** | Issues spawned by a workflow's routine in one cycle | Pipeline kanban defaults to current run; dropdown for previous runs. |
| **Ad-hoc ask** | Issue with `originKind=manual` | Created via Coordinator chat. Joins the current run. |
| **Tools** | Plugins + Adapters + Company Secrets, unified | Crew-scoped. Workflow-declared. Just-in-time connection. |
| **Crew Template** | New concept (preloaded crew) | Thick blueprint + worked sample run. |
| **Inbox** | Backed by `approvals` + `inbox` | Canonical home for HITL gates. |

**Forbidden vocabulary in user-facing copy:** *agent, execution, run, issue, ticket, backlog, checkout, workspace, PR, diff, story points, label, priority, assignee*. Replace with: *crew member, task, waiting on you, up next, in motion, ready, paused*.

---

## 5. Brand

- **Product name:** RunCrew.
- **Tagline target:** *"Run your business. Let your crew do the work."*
- **Vibe positioning:** Friendly mass-market in **content shape and language**, NOT in chrome.
- **Visual chrome:** Mirrors Paperclip's existing shadcn/ui + Tailwind setup. Monochrome / near-black primary, sharp corners, Inter throughout, restrained spacing. Linear-esque restraint.
- "Friendly" is expressed through IA simplification, microcopy warmth, illustrated empty states, and the Coordinator-as-colleague metaphor — **not** through coral accents, rounded mascots, or playful candy.

**Microcopy rules:**

- Address the user as a colleague would. *"Here's what your crew did this week"*, not *"Dashboard"*.
- Buttons read like actions a person takes. *"Review 3 posts"*, not *"Submit approval"*.
- Empty states encourage, never instruct. *"Maya's getting set up — first results in a few minutes."*
- *"Maya needs your OK on 4 reorders"* beats *"4 pending approvals."*

---

## 6. Information architecture

**Principle:** stay close to upstream Paperclip's sidebar pattern (section labels + nested items). Don't drift into a flat 5-item nav. The redesign relabels and re-groups; it does not invent a new shell.

### Sidebar layout (mirrors Paperclip's section pattern)

**Top quick actions (above sections):**
- **+ New Task** *(button)* — opens task-create dialog. Same affordance as upstream "New Issue", relabeled.
- **Dashboard** — primary landing screen. **Embeds the Coordinator chat directly** (no separate Coordinator nav item). Also surfaces workflow health, agent activity, recent run feed.
- **Inbox** *(with badge)* — HITL gates + read/unread items. (Whether to keep the label "Inbox" or use "Inbox" is a small open decision — see §12.)

**Section: WORKFLOWS** *(always expanded; replaces upstream "WORK" + "PROJECTS" sections)*

Lists every workflow as a clickable child item, **sorted alphabetically**. Same pattern Paperclip uses today for projects under "PROJECTS." Clicking a workflow opens its detail view (pipeline-column Kanban). Underlying Routines / Goals / raw Issues pages move behind an "Advanced" toggle inside this section.

**Section: CREW** *(always expanded; replaces upstream "AGENTS" + "COMPANY" sections, merged)*

Items, with crew-member items **sorted alphabetically by name** (Org pinned to top; Settings + Advanced items pinned to bottom):
- **Org** — the crew's org chart (Coordinator + delegated agents). *Pinned top.*
- **\<Agent name\>** — one item per crew member, sorted A→Z. Click → that agent's detail.
- **Skills** — *advanced; hidden by default*.
- **Costs** — *advanced; hidden by default*.
- **Activity** — *advanced; hidden by default*.
- **Settings** — crew settings (always visible). *Pinned bottom.*

**Above sidebar:** crew avatar + name (crew switcher dropdown) + search icon. The crew switcher dropdown contains *Switch Crew*, *+ Add Crew* (launches the New Crew flow — see §6a), *Invite people*, *Crew Settings*, *Sign out*. (Direct parity with Paperclip's existing company switcher pattern, relabeled.)

**Below sidebar:** user avatar + name + settings cog (unchanged).

### Two coordinated primary surfaces

1. **Coordinator chat (right-hand side panel of Dashboard)** — primary **creation and mutation** surface. **Right-hand persistent panel inside Dashboard**, top-to-bottom chat layout (newest at bottom, scrolls upward for history). ChatGPT-style bubbles. Natural language to add/edit/pause workflows and any other entity. Not a separate route — always visible alongside the Dashboard's workflow health summary in the main column.
2. **Kanban** — primary **inspection** surface. Two scopes:
   - **Workflow Detail Kanban**: cards = Tasks in the current run, columns = the workflow's own pipeline phases (e.g., Researching → Drafting → Awaiting your OK → Scheduled → Published).
   - **Project-issues Kanban** (Advanced view) — unchanged from upstream Paperclip.

### 6a. New Crew flow *(where Crew Templates live)*

**Crew Templates are NOT a top-level sidebar item.** They live inside the "+ Add Crew" wizard, surfaced as an alternative to manual setup.

Triggered from: crew switcher dropdown → **+ Add Crew**.

Modal/full-screen wizard with two coordinated panels:

- **Left panel — manual setup wizard (tabs):**
  1. **Company / Crew** — name, mission/goal (optional). Pattern matches Paperclip's existing company-create form.
  2. **Agent** — first agent (typically the Coordinator).
  3. **Task** — first task for the crew to work on.
  4. **Launch** — review + go.

- **Right panel — Crew Templates picker:**
  Headline: *"Choose from our Crew Templates"*. Cards for each available Crew Template (Social Media, etc.). Picking one pre-fills the left wizard with the template's defaults (Coordinator persona, agents, workflows, first task, tool requirements). The user can still edit each step before Launch.

This way the user always has a path forward: build from scratch *or* start from a Crew Template, in the same modal, without leaving the new-crew flow.

---

## 7. v1 surfaces (in build order)

1. **App shell** — sidebar with Workflows + Crew sections, top quick actions (New Task / Dashboard / Inbox), crew switcher dropdown.
2. **Dashboard** — primary landing screen. **Embeds the Coordinator chat** alongside workflow health summary, recent agent activity, and any pending HITL alerts. Replaces upstream's `Dashboard.tsx` content.
3. **Workflow Detail** — pipeline-column Kanban (cards = Tasks in current run, columns = workflow's own pipeline phases). Includes crew-members-on-this-workflow strip and tools strip. Run dropdown.
4. **Inbox** — HITL triage view, gate cards in five styles.
5. **New Crew flow** — wizard modal (Company / Agent / Task / Launch) + Crew Templates picker side panel. This is where Crew Templates live.
6. **Crew section views** — Org chart and individual agent detail pages. Reached via sidebar Crew section.
7. **First-run onboarding** — first invocation of the New Crew flow.

**Out of v1 (lives under "Advanced" within sidebar sections):** Skills, Costs, Activity, raw Issues board, Goals page, Routines page, Approvals admin page, Adapter Manager, Plugin Manager, deep settings.

---

## 8. Behavior rules

### HITL (Human-in-the-loop) gates are first-class

Most real business processes have approval/decision gates. Gates must be visually obvious across every surface, not buried.

**Canonical home:** the "Inbox" inbox.
**Also appears as:** distinct cards on the Kanban; proactive pings in Coordinator chat. Acting in any surface resolves the gate everywhere.

**Five gate types** (each a distinct card style, backed by `approvals` table extended with form/option payloads):

1. **Approve / Reject** — binary sign-off on a finished artifact. Artifact preview + Approve / Reject + comment.
2. **Review with edits** — same as (1) but artifact is inline-editable. Save & Approve.
3. **Choose between options** — agent offers 2–N alternatives; user picks one. Option cards + Pick this.
4. **Provide data / answer a question** — agent blocked on input. Short tailored form (text / select / upload).
5. **Acknowledge & continue** — informational, non-blocking. Notification card + Got it.

Gate cards always answer: **who's blocked waiting**, **what's at stake**, **what action is requested**.

### Tools are crew-scoped and just-in-time

- One OAuth connection per provider per Crew (not per agent). Backed by `company_secrets`.
- Workflows declare a **tool manifest**. When the user creates/adopts a workflow, the system prompts inline to connect any missing tools with a clear "why."
- No standalone Tools settings page in the primary nav.

### Agent reuse default

When the user creates a new workflow, the system **tries existing crew members first**. Extension of an existing crew member's skills/tools is preferred over creating a new crew member. Net-new crew members are surfaced to the user with the reason.

### Crew Templates are thick blueprints + worked samples

Each Crew Template ships with:
- `name`, `tagline`, gallery image/icon
- `coordinator`: { name, avatar, voiceNotes, systemPrompt }
- `agents[]`: { roleTitle, avatar, skills[], systemPrompt, defaultTools[] }
- `workflows[]`: { name, schedule, pipelinePhases[], defaultKpiSuggestions[], toolManifest[] }
- `tools[]`: { providerKey, displayName, whyNeeded, oauthScopes[] }
- `sampleRun` *(required v1)*: { tasks[], artifacts[], activityTimeline[] } — clearly labeled, one-click clearable
- `hitlGates[]` — declared upfront

**First Crew Template:** Social Media (codename "Social Media Poster") — pipeline: research top trends → analyze → produce social plan → generate copy + prompts for AI-media tools (Google Flow, Freepik, etc).

Live Crew Template updates (`crewTemplateVersion` tracking) deferred to v2.

**Naming convention:** display as **Crew Template** (two words, title case) in copy; reference as `crewTemplate` in schema/code. This intentionally disambiguates from Paperclip's existing user-facing "Prompt Template", "Branch template", and "Payload template JSON" — all of which are Mustache-style interpolation strings and unrelated to this concept.

---

## 9. Hard rule — Tasks must NOT inherit Paperclip's GitHub-flow UX

Schema reuse ≠ UX reuse. The `issues` table stays; the dev-ticket ergonomics do NOT.

**Drop from the UI:**
- Global status columns (Backlog / Todo / In Progress / In Review / Done / Cancelled). Status vocabulary is **workflow-defined**, never a system-wide ticket lifecycle.
- PR-style review chrome (diffs, document revisions, side-by-side). "Review" = preview the actual artifact (post, email, calendar block).
- Priority / estimates / story points / labels on the primary surface.
- `TES-N`-style issue identifiers on primary surfaces (Advanced view only).
- PR-style threaded comments — replaced with a chat-style activity timeline.
- "Assignee" as a primary editable field — surfaced ambiently ("Maya is on it"), not as a property to manage.
- Manual backlog / archive / grooming. Coordinator manages the queue.
- Execution workspaces / git worktree exposure.

**Keep invisibly:** atomic checkout, single-assignee, parent/child, blocker relations, status transitions, runs, audit trail. These do their job server-side.

---

## 10. Visual design system

**Anchor: Paperclip's existing shadcn/ui + Tailwind tokens.** Do not invent parallel visual primitives.

- Light mode, white background, near-black foreground (`oklch(0.145 0 0)`).
- Strict monochrome palette. Primary near-black (`#1A1A1A` / `oklch(0.205 0 0)`). Destructive red only for danger. Charts may use restrained palette colors.
- Roundness: minimal. ~4px on buttons/inputs, 0px on most surfaces. Sharp, Linear-esque.
- Typography: **Inter throughout**. Headlines 18–24px / 600–700. Body 13–15px / 400–500. Labels/chips 11–12px / 500–600. Mono only for raw data (IDs, URLs).
- Spacing: standard Tailwind scale. Cards 16–20px padding. Page gutters 24px desktop. Section spacing 24–32px.
- Components: all shadcn/ui (`Button`, `Card`, `Badge`, `Dialog`, `Tabs`, etc). Don't invent new shells.

---

## 11. Explicit rejections (out of scope, deliberately)

- New visual design system. Use shadcn parity.
- Coral / playful mass-market chrome (rejected by Sandro after initial exploration).
- Terminal aesthetic / monospace UI chrome.
- Dev-tool ergonomics in primary surfaces (PR diffs, priority pickers, label managers, sprint planning, backlog grooming).
- Dense data tables on primary surfaces (Advanced view only).
- Heavy gradients / glow / futurist styling.
- A user-authored Crew Template marketplace in v1 (first Crew Templates handcrafted in-repo).
- Live Crew Template version-tracking and upgrade flows in v1.
- Schema simplification (merging Project + Routine + Goal into one table). v2 conversation if needed.

---

## 12. Open design questions

These are intentionally deferred. Resolve during per-screen design.

- ✅ **Inbox label** — DECIDED 2026-05-17: keep "Inbox" (minimum drift from upstream).
- ✅ **Coordinator placement** — DECIDED 2026-05-17: right-hand persistent panel inside Dashboard, top-to-bottom chat layout.
- ✅ **Workflows / Crew sections** — DECIDED 2026-05-17: always expanded, sorted alphabetically (Crew has Org pinned top, Settings pinned bottom).
- **Dashboard main-column content** — what sits to the LEFT of the Coordinator chat panel inside Dashboard? Workflow health summary + recent agent activity + HITL alert strip? Specifics TBD.
- **Dashboard proactive HITL pings** — when a gate needs attention, does the Coordinator panel announce it in the thread, the Inbox badge increment, or both?
- **Crew section "Advanced" toggle** — single sub-toggle revealing Skills / Costs / Activity at once, or each independently configurable?
- **Workflow Detail tools strip** — show all tools or only the ones used in the current run? How do connection-loss errors surface here?
- **Inbox triage controls** — snooze, batch-approve, filters?
- **New Crew flow Crew Templates picker** — flat grid vs categorized? Preview-before-adopt vs adopt-then-edit?
- **First-run onboarding** — does the user land directly in the New Crew flow with Crew Templates as the default focus, or do we offer "describe your first workflow in chat" as an alternative path?

---

## 13. Source of truth

- **Memory:** `/Users/CaxtonTaylor/.claude/projects/-Users-CaxtonTaylor-Developer-paperclip/memory/project_paperclip_ui_redesign.md` (full interview record).
- **Stitch project:** `projects/4082416870657484991`, design system asset `assets/15704028756781800261` ("RunCrew — shadcn (Paperclip parity)").
- **First mockup:** Crew Home / Workflows Kanban, screen `c29e54b7129045debc1d3d9ca70336dd`.

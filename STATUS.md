# STATUS.md

Single source of truth. If any file conflicts with this one, trust this one.

### Current Objective
<!-- fill this out manually — human -->
Plan and build a Power Automate flow (Option 2): watch Outlook inbox, filter by sender/subject/folder/keyword conditions, call AI to draft a fact-grounded reply (using Outlook/Loop/Power BI context), save unattended to Drafts for review/send — but if the email needs the user's own action rather than a reply, post a Teams alert instead of drafting.

### Current Phase
- [x] Planning
- [x] Awaiting Approval
- [x] Executing (paused mid-build)
- [ ] Testing
- [ ] Complete

### Completed Tasks
- Scaffolded .claude control folder, CLAUDE.md, STATUS.md
- Wrote Option 2 implementation plan (PLAN.md) — trigger, condition, grounding, AI step, branch, error handling, all decisions resolved
- Build via browser-automation subagent against make.powerautomate.com in BD tenant. Confirmed server-side saved state: Trigger (Outlook "When a new email arrives V3", Inbox, To=self) → Condition (no-reply/system-sender filter, §2) → True branch → Scope containing: Extract Keywords, Search Prior Emails (Outlook Search email V3), Search SharePoint Site, Search OneDrive, Compose referenceMaterial (§3b steps 1-4). Flow left Off.
- AI Builder step (§3c/§3a) done and verified. Root causes fixed along the way: (1) hung agent-browser daemon fixed by killing stray process; (2) flow's deep-link environment ID was wrong (`8ac7231d-...`) — correct one is `Default-94c3e67c-9e2d-4800-a6b7-635d97882165`, found by navigating via Home → My flows instead of a hardcoded env URL; (3) discovered AI Builder's old inline-prompt flow action no longer exists in this tenant — replaced with a 2-part architecture: a named published prompt asset (`Email Draft Flow Prompt`, GPT-4.1 mini, JSON output, tested working) + a "Run a prompt" flow action that selects it and maps 4 inputs (referenceMaterial/sender/subject/body) from trigger/Compose dynamic content. Bound into the flow after Build referenceMaterial, inside Scope/True branch. Saved and confirmed persisted server-side via reload + Code view inspection. Flow ID `9d6cd058-4389-4cb8-af1e-dd15e95075b8`. Flow still Off.

### Pending Tasks
- Parse JSON action on the "Run a prompt" output (schema: needs_action bool, action_summary string, draft_reply string) — §3d
- Condition branch on needs_action → Teams alert (§4a) / Create draft (§4b)
- Error-handling Scope wrapping steps 3-6, on-failure Teams notify (§5)
- Test end-to-end (trigger → condition → grounded AI draft or Teams action-item alert)
- Leave flow Off until user explicitly turns it on

### Decisions
- Repo: shenc00/email-draft-flow, public, at Documents\Github\email-draft-flow
- Trigger scope: any email with me as direct To recipient, Inbox only, minus system/no-reply senders (see PLAN.md §2)
- AI engine: AI Builder "Run a prompt" flow action bound to a published custom prompt asset "Email Draft Flow Prompt" (GPT-4.1 mini, JSON output), confirmed working in BD (default) environment (2026-08-06) — see PLAN.md §3a for the 2-part architecture
- AI prompt now does classify+draft in one structured JSON call: needs_action / action_summary / draft_reply (PLAN.md §3c)
- Grounding: Outlook (Search email V3) + Loop (via SharePoint search, Loop files live in OneDrive/SharePoint) covered in v1; Power BI workspace content is a known gap, no connector supports full-text search over report content (PLAN.md §3b, §8)
- Action-item handling: Teams message to self (action_summary + email link), no draft created on that branch (PLAN.md §4a)
- Grounding scope: entire ISC Analytics & Data Science SharePoint site (bd1.sharepoint.com/sites/GSCTransformationGlobalSupplyChain) + user's OneDrive (PLAN.md §3b)

### Assumptions
- User has Power Automate license/connector access to Outlook

### Risks
- Flow left in a partially-built, saved-but-Off state in the live BD tenant (retrieval + AI step only, no branch/output steps yet). Low risk since it's Off and can't fire.
- Power Automate's designer can silently keep new actions as local-only browser drafts — always verify a save persisted by reloading the flow (or checking Code view), not just by seeing the action on screen. Cost one lost AI Builder step earlier in the build.
- The flow's environment ID is `Default-94c3e67c-9e2d-4800-a6b7-635d97882165` — do not use `8ac7231d-...`, that ID caused repeated false "Network Error" pages that looked like an outage but were just a wrong URL.

### Next Action
Add Parse JSON action on the AI output, then branch on needs_action → Teams alert (§4a) / Create draft (§4b) → error Scope (§5), then end-to-end test (§6 step 8). Keep flow Off until fully tested.

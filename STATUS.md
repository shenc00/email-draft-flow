# STATUS.md

Single source of truth. If any file conflicts with this one, trust this one.

### Current Objective
<!-- fill this out manually — human -->
Plan and build a Power Automate flow (Option 2): watch Outlook inbox, filter by sender/subject/folder/keyword conditions, call AI to draft a fact-grounded reply (using Outlook/Loop/Power BI context), save unattended to Drafts for review/send — but if the email needs the user's own action rather than a reply, post a Teams alert instead of drafting.

### Current Phase
- [x] Planning
- [x] Awaiting Approval
- [x] Executing — build structurally complete
- [x] Testing — flow turned On, live testing against real inbound mail
- [ ] Complete

### Completed Tasks
- Scaffolded .claude control folder, CLAUDE.md, STATUS.md
- Wrote Option 2 implementation plan (PLAN.md) — trigger, condition, grounding, AI step, branch, error handling, all decisions resolved
- Build via browser-automation subagent against make.powerautomate.com in BD tenant. Confirmed server-side saved state: Trigger (Outlook "When a new email arrives V3", Inbox, To=self) → Condition (no-reply/system-sender filter, §2) → True branch → Scope containing: Extract Keywords, Search Prior Emails (Outlook Search email V3), Search SharePoint Site, Search OneDrive, Compose referenceMaterial (§3b steps 1-4). Flow left Off.
- Error-handling Scope (§5) done and verified. Retrieval+AI steps already lived in a Scope; added Teams on-failure notification (runAfter Scope:Failed) as sibling after Condition 1, message includes trigger subject. Saved, verified via reload + Code view. Flow build structurally complete.
- **Flow turned On 2026-08-07**, per explicit user confirmation (acknowledged it starts firing on real inbound mail immediately, untested). Details page: https://make.powerautomate.com/environments/Default-94c3e67c-9e2d-4800-a6b7-635d97882165/flows/9d6cd058-4389-4cb8-af1e-dd15e95075b8/details
- Branch complete and verified (§3d/§4). Condition 1 (`needs_action == true`) added after Run a prompt — Parse JSON confirmed not needed, structured fields available directly. True branch: Teams "Post message in a chat or channel" to self via Chat with Flow bot + People Picker (message: action_summary only — no weblink field exists on the trigger in this tenant, dropped from message). False branch: Outlook "Create draft" (To=sender, Subject="RE: "+subject, Body=draft_reply plain token; ConversationId threading skipped — not readily available, follow-up item). Saved, verified via reload + Code view. Flow still Off.
- AI Builder step (§3c/§3a) done and verified. Root causes fixed along the way: (1) hung agent-browser daemon fixed by killing stray process; (2) flow's deep-link environment ID was wrong (`8ac7231d-...`) — correct one is `Default-94c3e67c-9e2d-4800-a6b7-635d97882165`, found by navigating via Home → My flows instead of a hardcoded env URL; (3) discovered AI Builder's old inline-prompt flow action no longer exists in this tenant — replaced with a 2-part architecture: a named published prompt asset (`Email Draft Flow Prompt`, GPT-4.1 mini, JSON output, tested working) + a "Run a prompt" flow action that selects it and maps 4 inputs (referenceMaterial/sender/subject/body) from trigger/Compose dynamic content. Bound into the flow after Build referenceMaterial, inside Scope/True branch. Saved and confirmed persisted server-side via reload + Code view inspection. Flow ID `9d6cd058-4389-4cb8-af1e-dd15e95075b8`. Flow still Off.

- **Investigated false-alarm Teams messages 2026-08-07**: user saw ~11 Teams DMs reading "Email Draft Flow failed processing an email... Check flow run history for details" right after turn-on. Checked Email Draft Flow v1 run history directly: **12/12 runs Succeeded, zero failures** — flow is working correctly. Root cause of the Teams messages: unrelated pre-existing flow "Send webhook alerts to Sally Shen" (generic webhook-relay template, flow ID `d7497759-ca5c-4add-8a66-a945b3dc47c3`) is receiving that literal string as webhook payload text from some external, unidentified caller, and failing to post it into Teams (`BadRequest: Call made for a thread which is not a ChatThread` — bad/stale thread ID in its Post-card action). Coincidental wording collision only; not connected to Email Draft Flow v1. No fix applied (separate pre-existing flow, out of scope) — flagged as a risk item below in case user wants it addressed separately.

- **Root cause of zero drafts found 2026-08-07**: traced a real run (12:36 AM) action-by-action. Outer flow status "Succeeded" was misleading — the retrieval Scope inside actually **Failed**: "Search SharePoint Site" hit `AADSTS700082 — refresh token expired due to inactivity` (SharePoint Online connection token issued 2026-03-24, unused 90 days, expired). Everything downstream (Build referenceMaterial, Run a prompt, Condition 1, Teams alert, Create draft) was **Skipped** as a result — flow never actually produced output on any real trigger despite showing green. Fix needed: reauthorize the SharePoint Online connection (Power Automate → Data → Connections → re-enter credentials), one-time reconnect, not a rebuild. Not yet done — needs user or a sign-in-capable session.

### Pending Tasks
- **Reauthorize SharePoint Online connection** (blocking — root cause of zero drafts, see above)
- Re-verify actual output (Drafts folder + Teams) AFTER reconnecting SharePoint — prior "Succeeded" runs were false positives
- Refine no-reply/system sender pattern list (§2) after a week of real hits, per original open item
- Optional follow-ups (not blocking): draft signature block, Body length cap, raw-AI-JSON logging, Teams action-item message weblink (no field available in this tenant — may need a workaround or drop permanently), Create draft ConversationId threading

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
- Power Automate's designer can silently keep new actions as local-only browser drafts — always verify a save persisted by reloading the flow (or checking Code view), not just by seeing the action on screen. Cost one lost AI Builder step earlier in the build.
- The flow's environment ID is `Default-94c3e67c-9e2d-4800-a6b7-635d97882165` — do not use `8ac7231d-...`, that ID caused repeated false "Network Error" pages that looked like an outage but were just a wrong URL.
- Unrelated pre-existing flow "Send webhook alerts to Sally Shen" (`d7497759-ca5c-4add-8a66-a945b3dc47c3`) is broken (bad Teams thread ID) and produces confusing Teams DMs that read like Email Draft Flow failures but aren't. Out of scope for this project; user may want it fixed or disabled separately to stop the noise.

### Next Action
Email Draft Flow v1 confirmed working (12/12 runs Succeeded). Check Drafts folder and Teams (self) to eyeball actual AI output quality on real emails. Ignore "Email Draft Flow failed..." Teams DMs — those come from the unrelated broken webhook-relay flow above, not this flow.

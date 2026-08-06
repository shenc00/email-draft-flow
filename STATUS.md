# STATUS.md

Single source of truth. If any file conflicts with this one, trust this one.

### Current Objective
<!-- fill this out manually — human -->
Plan and build a Power Automate flow (Option 2): watch Outlook inbox, filter by sender/subject/folder/keyword conditions, call AI to draft a fact-grounded reply (using Outlook/Loop/Power BI context), save unattended to Drafts for review/send — but if the email needs the user's own action rather than a reply, post a Teams alert instead of drafting.

### Current Phase
- [x] Planning
- [ ] Awaiting Approval
- [ ] Executing
- [ ] Testing
- [ ] Complete

### Completed Tasks
- Scaffolded .claude control folder, CLAUDE.md, STATUS.md

### Pending Tasks
- Write Option 2 implementation plan (flow design, trigger, condition, AI action, draft save)
- Get plan approval
- Build flow in Power Automate
- Test end-to-end (trigger → condition → AI draft → Drafts folder)

### Decisions
- Repo: shenc00/email-draft-flow, public, at Documents\Github\email-draft-flow
- Trigger scope: any email with me as direct To recipient, Inbox only, minus system/no-reply senders (see PLAN.md §2)
- AI engine: AI Builder "Create text with GPT using a prompt" (GPT-4.1 mini), confirmed working in BD (default) environment via make.powerapps.com → AI hub → Prompts test (2026-08-06)
- AI prompt now does classify+draft in one structured JSON call: needs_action / action_summary / draft_reply (PLAN.md §3c)
- Grounding: Outlook (Search email V3) + Loop (via SharePoint search, Loop files live in OneDrive/SharePoint) covered in v1; Power BI workspace content is a known gap, no connector supports full-text search over report content (PLAN.md §3b, §8)
- Action-item handling: Teams message to self (action_summary + email link), no draft created on that branch (PLAN.md §4a)

### Assumptions
- User has Power Automate license/connector access to Outlook

### Risks
- None currently blocking

### Next Action
Pick which SharePoint site(s)/OneDrive scope the grounding search should hit (PLAN.md §7, last open item), then build per PLAN.md §6.

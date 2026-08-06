# STATUS.md

Single source of truth. If any file conflicts with this one, trust this one.

### Current Objective
<!-- fill this out manually — human -->
Plan and build a Power Automate flow (Option 2): watch Outlook inbox, filter by sender/subject/folder/keyword conditions, call AI to draft a reply, save unattended to Drafts folder for review/send.

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

### Assumptions
- User has Power Automate license/connector access to Outlook

### Risks
- None currently blocking

### Next Action
Build the flow per PLAN.md §6 (all inputs now decided) — start with trigger + condition, then AI Builder prompt action.

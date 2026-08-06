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

### Assumptions
- User has Power Automate license/connector access to Outlook + an AI connector (Azure OpenAI / Copilot Studio / AI Builder)

### Risks
- AI connector choice affects cost/setup complexity — needs decision in plan

### Next Action
Write Option 2 plan doc, present task list, get approval.

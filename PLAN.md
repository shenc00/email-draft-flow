# Plan — Option 2: Power Automate + AI (auto-draft replies)

Goal: unattended flow — new email lands → filtered by rule → AI drafts reply → saved to Drafts for human review/send. No manual click-per-email.

## 1. Flow shape

```
Trigger: Office 365 Outlook — "When a new email arrives (V3)"
   └─ Folder: Inbox (or sub-folder)
   └─ To/Cc, Subject Filter, Importance — native trigger filters
Condition (optional, in-flow)
   └─ For rules the trigger can't express (keyword list, multi-sender OR logic)
AI action — draft reply text
   └─ AI Builder "Create text with GPT using a prompt" (default) OR
      Azure OpenAI connector "Get chat completion" (if quota/resource exists)
Create draft
   └─ Office 365 Outlook "Create draft"
      To = trigger sender, Subject = "RE: " + trigger subject,
      Body = AI output, ConversationId = trigger conversation id (keeps threading)
Error handling
   └─ Scope + "Configure run after" on failure → Teams/email notify self
```

## 2. Trigger config

- Connector: Office 365 Outlook (uses your own mailbox identity — no Azure app registration, unlike Graph API).
- Filters available directly on the trigger: Folder, Importance, Has Attachment, To, Subject Filter, From.
- Anything beyond that (e.g. "sender is A OR B OR C", "subject contains any of these 5 keywords") → add a **Condition** action right after the trigger using `contains()` / `or()` expressions, skip AI+draft steps if false.

## 3. AI step — two options

| | AI Builder prompt | Azure OpenAI connector |
|---|---|---|
| Setup | Native action, no API key | Needs Azure OpenAI resource + connection |
| Cost | AI Builder credits (Premium/E5 licensing) | Azure OpenAI consumption billing |
| Best for | Fast start, no infra ask | If BD already has an Azure OpenAI resource with quota |

**Recommendation: start with AI Builder prompt** — zero infra dependency, fastest to test. Swap to Azure OpenAI later only if prompt quality/control needs exceed what AI Builder's prompt builder offers.

Prompt template (fill from trigger outputs):
```
You are drafting a reply on behalf of [Name]. Write a professional, concise reply to the email below.
From: {sender}
Subject: {subject}
Body: {body}

Reply:
```

## 4. Draft creation

- Action: Office 365 Outlook **"Create draft"**.
- Set `ConversationId` from the trigger to keep the draft threaded under the original email (not a new top-level message).
- Leave `To` prefilled from sender — human just reviews/edits/sends from Drafts.

## 5. Error handling

- Wrap AI + Create draft in a **Scope**.
- Parallel "Scope — On failure" branch (Configure run after = has failed) → post a Teams/Outlook notification to self with the trigger email's subject, so a bad AI response doesn't silently vanish.

## 6. Build steps (~30–60 min)

1. New flow → trigger → set folder + basic filters.
2. Add Condition for any filter logic the trigger can't express.
3. Add AI Builder prompt action, wire trigger subject/body/sender into the prompt.
4. Add Create draft action, map AI output → Body.
5. Wrap 3–4 in a Scope, add failure-notify branch.
6. Test: send yourself a matching email, confirm draft appears threaded in Drafts with sane AI text.
7. Tune prompt / filters from test output, re-test.
8. Turn flow on.

## 7. Open decisions (need your input before build)

- Which mailbox rule(s) actually route to this flow — sender list? subject keywords? specific folder?
- AI Builder vs Azure OpenAI — confirm BD licensing has AI Builder credits available.
- Failure notification channel — Teams message vs email to self.

## 8. Out of scope (v1)

- Auto-send without review (this stays draft-only by design — matches the "review/send" requirement in the original comparison).
- Multi-language reply detection.

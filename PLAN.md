# Plan — Option 2: Power Automate + AI (auto-draft replies)

Goal: unattended flow — new email lands → filtered by rule → AI drafts reply → saved to Drafts for human review/send. No manual click-per-email.

## 1. Flow shape

```
Trigger: Office 365 Outlook — "When a new email arrives (V3)"
   └─ Folder: Inbox only (Junk/spam lands in its own folder — trigger never sees it)
   └─ To: my address (fires only when I'm a direct recipient, not just Cc'd)
Condition — system/notification filter
   └─ Skip AI+draft if sender matches a no-reply/automated pattern
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
- Folder = **Inbox**. Spam already gets routed to Junk Email by Outlook before the trigger ever sees it — no extra filter needed for that.
- `To` field on the trigger = your address. This restricts firing to emails where you're a direct recipient (not Cc/Bcc-only, not a distribution list you happen to be on unless you're also named in To).
- **System/notification exclusion** — the trigger has no "sender is NOT X" filter, so this needs a **Condition** action right after the trigger:
  ```
  not(or(
    contains(toLower(triggerBody()?['from']), 'noreply'),
    contains(toLower(triggerBody()?['from']), 'no-reply'),
    contains(toLower(triggerBody()?['from']), 'donotreply'),
    contains(toLower(triggerBody()?['from']), 'notification'),
    contains(toLower(triggerBody()?['from']), 'automated'),
    contains(toLower(triggerBody()?['from']), 'alerts@')
  ))
  ```
  If false → terminate flow (no draft). Extend the sender-pattern list as real system senders show up in testing.
- Any further rule beyond this (specific sender list, subject keywords) → same Condition action, additional `contains()` / `or()` clauses.

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

- ~~Which mailbox rule(s) route to this flow~~ — decided: any email with me as direct `To` recipient in Inbox, minus system/notification senders (§2).
- Exact no-reply/system sender pattern list — starter list in §2, refine after first week of real trigger hits.
- AI Builder vs Azure OpenAI — confirm BD licensing has AI Builder credits available.
- Failure notification channel — Teams message vs email to self.

## 8. Out of scope (v1)

- Auto-send without review (this stays draft-only by design — matches the "review/send" requirement in the original comparison).
- Multi-language reply detection.

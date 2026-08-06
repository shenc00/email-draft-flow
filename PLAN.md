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

## 3. AI step

### 3a. Engine choice

**Constraint (confirmed 2026-08-06): no BD AI Builder credits, no BD Azure OpenAI access.** Only M365 Copilot Chat and GitHub Copilot are available — neither exposes a callable API/connector for unattended per-email use (Copilot Chat has no headless API; GitHub Copilot is a code-completion tool, not a chat API). Both are ruled out for this flow.

**Step 0 — check for free tenant AI Builder allowance before building anything else.** Many M365 tenants grant a small free AI Builder capacity per environment regardless of per-user license. This needs your browser session (no CLI access available here):
1. Go to `admin.powerplatform.microsoft.com` → Resources → Capacity → your environment.
2. Look for an "AI Builder" or "AI Builder service units" row — any nonzero free allowance means the AI Builder prompt action works at zero cost, no request needed.
3. If it shows 0 / requires a paid add-on, fall back to one of the two options below.

| | AI Builder prompt (if free capacity found) | Personal LLM API key (fallback) |
|---|---|---|
| Action | "Create text with GPT using a prompt" | HTTP action → Gemini/Groq free-tier REST endpoint |
| Auth | None — native to your Power Platform environment | Your own personal API key (not BD's) |
| Cost | Free, within tenant's granted capacity | Free tier (Groq, Gemini) or pennies/month at this volume |
| **Compliance flag** | None — stays inside BD's M365 tenant | **Work email content (sender, subject, body) leaves BD's tenant boundary to an external consumer AI service under your personal account. Confirm this is acceptable before wiring real inbox data through it — check with IT/data-handling policy first, especially for anything with client/PHI/confidential content.** |
| Best for | Default if tenant has any free allowance | Only if Step 0 comes back empty and you've cleared the compliance question |

**Recommendation:** do Step 0 first — costs nothing, no risk, might fully solve this. Only reach for the personal-API-key fallback (§3a table, right column) after confirming zero AI Builder capacity AND after you're comfortable with the compliance flag above. Until Step 0 is checked, treat AI engine as **undecided** — don't build §4 (Create draft) against a specific action yet.

If personal API key is the path: Google Gemini (`gemini-1.5-flash` or newer, generous free tier) or Groq (Llama models, free tier, fast) both support a plain HTTPS POST — no SDK needed, works from Power Automate's native HTTP action.

### 3b. Prompt design

Split into system framing + per-email content (AI Builder's Prompt Builder supports instruction text above the `{variable}` fields; Azure OpenAI does this as separate system/user messages):

```
SYSTEM / instruction:
You are drafting an email reply on behalf of [Your Name], [Your Title].
Write in a professional, concise tone matching a typical business email.
Keep it under 150 words unless the original email requires detail to answer properly.
Never invent facts, commitments, dates, numbers, or approvals that aren't
in the original email — if the reply requires information you don't have,
write a short holding reply asking for time to check, instead of guessing.
Do not add a closing signature block — that gets appended separately.
End your draft with nothing but the reply body text.

USER / per-email content:
From: {sender}
Subject: {subject}
Body:
{body}
```

Guardrail reasoning: an AI reply that fabricates a commitment ("I'll have this to you by Friday") is worse than no draft at all, since it sits in Drafts looking finished and might get sent as-is. The "don't invent, ask for time instead" instruction is the single highest-value line in the prompt — test it explicitly (send a test email asking something the AI can't know) before trusting the flow with real mail.

### 3c. Output handling

- Both engine options return plain text — Office 365 Outlook's "Create draft" `Body` field accepts HTML, so wrap the AI output in a minimal template before writing it: `<p>` per paragraph, then append your real signature block (stored as a flow variable, not generated by the AI) below.
- Trim/validate length before writing to Body — cap around 2000 characters as a sanity check; if the AI response exceeds that, treat it as a failure (see §5) rather than write a bloated draft.
- Log the raw AI output somewhere retrievable during testing (a SharePoint list row or the flow's own run history) so you can review drift in quality over the first couple weeks without re-reading every draft in Outlook.

## 4. Draft creation

- Action: Office 365 Outlook **"Create draft"**.
- Set `ConversationId` from the trigger to keep the draft threaded under the original email (not a new top-level message).
- Leave `To` prefilled from sender — human just reviews/edits/sends from Drafts.

## 5. Error handling

- Wrap AI + Create draft in a **Scope**.
- Parallel "Scope — On failure" branch (Configure run after = has failed) → post a Teams/Outlook notification to self with the trigger email's subject, so a bad AI response doesn't silently vanish.

## 6. Build steps (~30–60 min, once §3a Step 0 resolves the engine)

0. Check Power Platform admin center for free AI Builder capacity (§3a Step 0). This decides which of the two engine paths below you build.
1. New flow → trigger → set folder + basic filters.
2. Add Condition for any filter logic the trigger can't express.
3. Add AI action — AI Builder prompt (if free capacity found) or HTTP call to personal LLM API key (if not, and compliance question cleared) — wire trigger subject/body/sender into the prompt.
4. Add Create draft action, map AI output → Body.
5. Wrap 3–4 in a Scope, add failure-notify branch.
6. Test: send yourself a matching email, confirm draft appears threaded in Drafts with sane AI text.
7. Tune prompt / filters from test output, re-test.
8. Turn flow on.

## 7. Open decisions (need your input before build)

- ~~Which mailbox rule(s) route to this flow~~ — decided: any email with me as direct `To` recipient in Inbox, minus system/notification senders (§2).
- Exact no-reply/system sender pattern list — starter list in §2, refine after first week of real trigger hits.
- **Blocking: AI engine.** Confirmed no BD AI Builder/Azure OpenAI access (§3a). Need you to check Power Platform admin center for free tenant AI Builder capacity (§3a Step 0) — this decides AI Builder-native vs personal-API-key path, and the personal-key path additionally needs a compliance go/no-go on sending work email content to an external service under a personal account.
- Failure notification channel — Teams message vs email to self.

## 8. Out of scope (v1)

- Auto-send without review (this stays draft-only by design — matches the "review/send" requirement in the original comparison).
- Multi-language reply detection.

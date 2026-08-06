# Plan — Option 2: Power Automate + AI (auto-draft replies)

Goal: unattended flow — new email lands → filtered by rule → AI drafts a **fact-grounded** reply, using your Outlook/Loop/Power BI content as reference → saved to Drafts for human review/send. If the email actually needs *you* to do something (not just reply), the flow pings you on Teams instead of faking a reply.

## 1. Flow shape

```
Trigger: Office 365 Outlook — "When a new email arrives (V3)"
   └─ Folder: Inbox only (Junk/spam lands in its own folder — trigger never sees it)
   └─ To: my address (fires only when I'm a direct recipient, not just Cc'd)
Condition — system/notification filter
   └─ Skip AI+draft if sender matches a no-reply/automated pattern
Retrieve context (§3d)
   └─ Search Outlook mail history + SharePoint/OneDrive (covers Loop pages) for
      snippets related to the email's subject/keywords — feeds the AI real facts
AI action — classify + draft (structured JSON output)
   └─ AI Builder "Create text with GPT using a prompt"
      Returns: { needs_action: bool, action_summary: string, draft_reply: string }
Branch on needs_action
   ├─ true  → Teams message to self: action_summary + link to email (§4b), NO draft
   └─ false → Create draft (Office 365 Outlook), Body = draft_reply,
              ConversationId = trigger conversation id (keeps threading)
Error handling
   └─ Scope + "Configure run after" on failure → Teams notify self
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

### 3a. Engine choice — RESOLVED 2026-08-06, updated 2026-08-06 (build-time correction)

Confirmed in `make.powerapps.com` (BD (default) environment) → AI hub → Prompts → Build your own prompt → Test: real response returned, model **GPT-4.1 mini**, no license/capacity error. BD's default environment has AI Builder capacity — use it.

- **Engine: AI Builder, two-part architecture** (discovered during build — the flow action "Create text with GPT using a prompt" referenced in the original plan no longer exists in this tenant):
  1. **A named custom prompt asset**, built once in AI hub → Prompts, holding the actual template text, the four named inputs (`referenceMaterial`, `sender`, `subject`, `body`), the model selection, and the JSON output schema. Built as `Email Draft Flow Prompt`, model GPT-4.1 mini, output type JSON, tested with sample values, published.
  2. **The flow action is AI Builder "Run a prompt"** (Dataverse connector) — it does not take inline prompt text; it has a single dropdown that selects a published prompt asset by name, then exposes that asset's named inputs as fields to map from flow dynamic content.
- No auth/API key needed — runs under your own Power Platform environment, work email content never leaves BD's tenant.
- Personal-LLM-API-key fallback (Gemini/Groq) and its compliance flag are no longer needed — dropped from the build path, kept below for reference only in case tenant capacity ever gets revoked.

<details>
<summary>Fallback if AI Builder capacity is later removed</summary>

HTTP action → Gemini/Groq free-tier REST endpoint, your own personal API key. **Compliance flag:** work email content would leave BD's tenant boundary to an external service under a personal account — needs IT/data-handling sign-off before use, especially for anything client/PHI/confidential.
</details>

### 3b. Grounding — Outlook / Loop / Power BI context

Decided source: Outlook, Loop, and Power BI workspaces (your answer, 2026-08-06). Reality check on what's actually buildable per source with native Power Automate actions, no new infra:

| Source | Coverage in v1 | How |
|---|---|---|
| Outlook | Full | "Search email (V3)" action, query built from the incoming email's subject/keywords, pull top N matching prior threads (subject + snippet) as reference material |
| Loop | Full (indirect) | Loop pages/components are `.loop` files stored in OneDrive/SharePoint — covered by the same SharePoint search call below, no separate Loop-specific connector exists |
| Power BI | **Gap — not in v1** | No Power Automate connector action does full-text content search over report/dataset content. The Power BI connector only does things like export/refresh/list reports — it can't answer "what does this report say." Workaround if this matters: keep a Loop/SharePoint doc summarizing key PBI metrics/definitions in plain text — that gets picked up by the SharePoint search like any other doc. Flagged as a v1 gap, not silently dropped. |

**Retrieval step** (before the AI action):
1. Extract 2–4 keywords from the trigger email's subject (simple `split()`/`replace()` expression, no AI call needed for this).
2. Office 365 Outlook **"Search email (V3)"** — query = keywords, top 3 results, project subject + body preview.
3. SharePoint **"Send an HTTP request to SharePoint"** → REST search endpoint `/_api/search/query?querytext='{keywords}' path:"https://bd1.sharepoint.com/sites/GSCTransformationGlobalSupplyChain"` (site: **ISC Analytics & Data Science**, entire site — not scoped to one folder) plus a second call scoped to your personal OneDrive `path:"https://bd1-my.sharepoint.com/personal/<your-account>"` (covers Loop files in both) — top 3 results per source, project title + snippet.
4. Concatenate both result sets into a single `referenceMaterial` string variable, passed into the AI prompt below.

If retrieval turns up nothing relevant, `referenceMaterial` is just empty — the prompt's "don't invent facts" instruction (§3c) makes the AI fall back to a generic acknowledgment rather than fabricate, so an empty result isn't a failure mode.

### 3c. Prompt design

AI Builder Prompt Builder supports **structured (JSON) output** — set Output type to JSON with a schema, instead of plain Text. This lets one AI call do both the "is this an action item for me" classification and the reply draft, instead of two separate calls.

```
SYSTEM / instruction:
You are triaging and drafting a reply to an email on behalf of [Your Name], [Your Title].

Reference material (facts you may cite — do not use anything outside this
and the email itself; if it's empty, you have no extra facts available):
{referenceMaterial}

Step 1 — decide: does this email require ME to personally take an action
(review something, approve something, complete a task, attend something,
provide information only I have) rather than just receive a reply?
If yes, set needs_action=true and write a one-sentence action_summary.
Leave draft_reply empty in that case — do not draft a reply for action items.

Step 2 — if needs_action=false, write draft_reply: a professional, concise
reply (under 150 words unless the email needs detail to answer properly).
Use ONLY facts from the reference material or the email itself — never
invent commitments, dates, numbers, or approvals. If you lack the
information to answer properly, write a short holding reply asking for
time to check, instead of guessing.
No closing signature block — that gets appended separately by the flow.

Return JSON: { "needs_action": boolean, "action_summary": string, "draft_reply": string }

USER / per-email content:
From: {sender}
Subject: {subject}
Body:
{body}
```

Guardrail reasoning: an AI reply that fabricates a commitment ("I'll have this to you by Friday") is worse than no draft at all — it sits in Drafts looking finished and might get sent as-is. The "don't invent, ask for time instead" line is the single highest-value instruction here; test it explicitly (send a test email asking something the AI can't know) before trusting the flow with real mail. Equally, the action-item split matters because a *drafted reply* to something that actually needs your personal action (approval, task, decision) risks you skimming Drafts, seeing something plausible-looking, and missing that it needed real attention — better to interrupt via Teams than to paper over it with a polished-sounding auto-reply.

### 3d. Output handling — updated 2026-08-06 (build-time correction)

- **Parse JSON was not needed.** The AI Builder "Run a prompt" action in this tenant already exposes `needs_action`, `action_summary`, `draft_reply` as directly-selectable structured dynamic content fields (`body/responsev2/predictionOutput/structuredOutput/*`) — no separate parsing step required.
- `draft_reply` is mapped as a plain token into Outlook "Create draft" (renamed "Draft an email message" in this connector version) Body field, wrapped in a single `<p>` by the designer's rich-text editor. No signature block appended yet — flagged as a v1.1 follow-up if desired, not blocking.
- Length cap / bloated-draft sanity check — not yet implemented, deferred to §5 error-handling pass or a later iteration.
- Raw AI JSON logging for quality review — not yet implemented, deferred.

## 4. Branch: action item vs draft reply

Condition action, reading the Parse JSON output: `needs_action == true`.

### 4a. True branch — action item, notify instead of drafting — built 2026-08-06

- Action: Teams **"Post message in a chat or channel"**, Post as Flow bot, Post in "Chat with Flow bot", Recipient `Sally.Shen@bd.com` via People Picker. ("Chat with self" has no dedicated preset in this tenant's Teams connector; "Enter custom value" with a raw email rejected by the API with `MissingOrInvalidTeamsFlowbotRecipientType" — Chat with Flow bot + People Picker is the working substitute.)
- Message: `Action needed: {action_summary}` only — **deviation**: no weblink-style field exists on the "When a new email arrives (V3)" trigger's dynamic content in this tenant (searched, none found), so the "+ link to email" part of the original design was dropped rather than blocking the build.
- **No Create draft call on this branch.**

### 4b. False branch — draft reply — built 2026-08-06

- Action: Office 365 Outlook **"Create draft"** (shown as "Draft an email message" in this connector version).
- `To` = trigger's From (`@triggerOutputs()?['body/from']`), via dynamic-content mode since the field defaults to a People Picker not built for token binding.
- `Subject` = `RE: {trigger subject}`.
- `Body` = `draft_reply` token, plain mapping, no signature block yet (see §3d).
- `ConversationId` — **deviation, skipped**: not readily available as a simple field mapping in this connector version; threading under the original email is not yet wired. Follow-up item.

## 5. Error handling — built 2026-08-06

- Retrieval + AI steps (Extract Keywords → ... → Run a prompt) already sit in a Scope (built alongside those steps). Condition 1 + branch actions sit as siblings after the Scope, inside the Condition's True branch.
- Added Teams "Post message in a chat or channel" as a sibling after Scope/Condition 1, `runAfter: {"Scope": ["Failed"]}` — fires only if the Scope fails. Message: "Email Draft Flow failed processing an email. Subject: {trigger subject}. Check flow run history for details." Posted to self via Chat with Flow bot.

## 6. Build steps (~45–75 min — grounding + branching adds time vs the original single-path version)

1. New flow → trigger → set folder + basic filters.
2. Add Condition for any filter logic the trigger can't express (§2).
3. Add retrieval steps: keyword extraction, Outlook Search email (V3), SharePoint HTTP search — concatenate into `referenceMaterial` (§3b).
4. Add AI Builder prompt action with JSON output (§3c), wire in `referenceMaterial` + trigger subject/body/sender.
5. Add Parse JSON action on the AI response (§3d).
6. Add Condition on `needs_action`, build the two branches (§4a Teams alert / §4b Create draft).
7. Wrap 3–6 in a Scope, add failure-notify branch (§5).
8. Test both paths: send a test email that clearly needs action from you (confirm Teams alert, no draft), and one that's a plain reply-able question (confirm grounded draft appears threaded in Drafts).
9. Tune prompt / retrieval / filters from test output, re-test.
10. Turn flow on.

## 7. Open decisions (need your input before build)

- ~~Which mailbox rule(s) route to this flow~~ — decided: any email with me as direct `To` recipient in Inbox, minus system/notification senders (§2).
- ~~AI engine~~ — decided: AI Builder GPT-4.1 mini, confirmed working in BD (default) environment (§3a).
- ~~Action-item alert channel~~ — decided: Teams message to self (§4a).
- ~~Grounding source~~ — decided: Outlook + Loop (via SharePoint search) + Power BI (gap — no v1 coverage, see §3b table).
- ~~Which SharePoint site(s)/OneDrive scope~~ — decided: entire ISC Analytics & Data Science site (`bd1.sharepoint.com/sites/GSCTransformationGlobalSupplyChain`) + your OneDrive (§3b step 3).
- Exact no-reply/system sender pattern list — starter list in §2, refine after first week of real trigger hits.

## 8. Out of scope (v1)

- Auto-send without review (this stays draft-only by design — matches the "review/send" requirement in the original comparison).
- Multi-language reply detection.
- Power BI workspace content as a grounding source (§3b) — no native connector action for full-text search over report/dataset content; use a maintained SharePoint/Loop summary doc as a workaround if needed.

# n8n Lead Qualification Workflow

An n8n workflow that takes a website inquiry, qualifies it with AI against fixed rules, creates or updates the contact without duplicates, sets a tag and pipeline stage, drafts a personalised follow-up email for human review, and alerts the sales team only when a lead newly becomes qualified. Failed runs are retried, then logged.

Built as a portfolio project from a real Upwork job brief.

> **Note on substitutions.** The brief named OpenAI and GoHighLevel. Both needed a payment card that was declined during this build, so this version uses **Google Gemini** for the AI step and a **Google Sheets mock CRM**. The logic is identical, and only the AI node and the CRM nodes would change. The OpenAI and GoHighLevel versions were not built or tested.

## Demo
[Watch the 3-minute demo video]https://drive.google.com/file/d/1Q0KRpPWK29bLmjPWR9Ox0IVHBs8XlQAi/view?usp=sharing


## Stack

- n8n (Cloud trial)
- Google Gemini (Flash model)
- Google Sheets (mock CRM)
- Gmail (follow-up drafts and sales notifications)

## How it works

```
Webhook -> Edit Fields -> Message a model (Gemini) -> Get row(s) in sheet
        -> Code in JavaScript -> Contact exist? (If)
             |-- new:      Append row in sheet  --\
             |-- existing: Update row in sheet  ---> Create follow-up draft (Gmail)
                                                       -> Notify sales? (If) -> Notify sales team (Gmail)
```

A second workflow (**Error Logger**) starts from an Error Trigger and writes failures to the Execution Log tab of the sheet.

## Qualification rules

| Verdict | Condition | Tag | Pipeline stage |
|---|---|---|---|
| qualified | Budget 5,000 or more, AND timeline within 3 months, AND an offered service | `qualified` | Qualified |
| unqualified | Budget clearly under 5,000, OR timeline over 3 months, OR service clearly not offered | `unqualified` | Unqualified |
| manual_review | Anything unclear: missing or zero budget, vague timeline or service, nonsensical message | `needs-review` | Manual Review |

Safety rules:

- The prompt treats the lead's message as untrusted text and never follows instructions inside it.
- The parsing step forces any unparseable or unexpected AI output to `manual_review`.

## What each step does

1. **Webhook:** receives the form submission (POST).
2. **Edit Fields:** trims text, lowercases the email, strips symbols from the phone number, converts the budget to a number (for example "$2,000" becomes 2000).
3. **Message a model:** Gemini returns JSON with `verdict`, `reason`, `followup_subject` and `followup_body`.
4. **Get row(s) in sheet:** looks the contact up by email. *Always Output Data* is on, so a new contact does not stop the run.
5. **Code in JavaScript:** parses the AI answer, validates the verdict and adds the tag and pipeline stage.
6. **Contact exist? (If):** routes to append (new) or update (existing).
7. **Append / Update row:** writes the contact fields, tag, stage, AI reason and timestamps.
8. **Create follow-up draft:** saves the AI-written email as a Gmail draft. Nothing is sent automatically.
9. **Notify sales? (If):** true only when the verdict is `qualified` AND the contact was not already Qualified before this submission.
10. **Notify sales team:** emails a lead summary to the sales address.

## Duplicate and repeat protection

- Contacts are matched on the cleaned email, so `  CHIDI.EZE@example.com` matches `chidi.eze@example.com`.
- A repeat submission updates the existing row instead of adding a new one.
- Notifications fire only when a contact newly becomes Qualified.
- Each submission creates a new follow-up draft (design choice: drafts are never auto-sent, and details may have changed).

## Error handling

- **Retry On Fail** is on for the Gemini, Google Sheets and Gmail nodes (3 tries, 2 seconds apart).
- The main workflow's settings point to the Error Logger workflow, which appends a row (timestamp, `FAILED`, failing node and error message, execution link) to the Execution Log tab.
- The error workflow must be published before it can be selected, and it runs on failures of published (live) runs, not manual test runs.

## Google Sheet structure

**Contacts** tab (row 1 headers, lowercase):

`email`, `name`, `phone`, `service`, `budget`, `timeline`, `message`, `tags`, `pipeline_stage`, `created_at`, `updated_at`, `ai_reason`

Format the `phone` column as Plain text so a leading `+` is kept.

**Execution Log** tab: `timestamp`, `email`, `status`, `details`

**Pipeline** tab: reference list of the stages (New Lead, Qualified, Manual Review, Unqualified). It is not read by the workflow.

## Setup

1. Create the Google Sheet with the tabs and headers above.
2. In n8n, import `lead-qualification-error-logger.json` and `lead-qualification-main.json` (workflow menu -> **Import from file**).
3. Reconnect credentials: Google Sheets, Gmail and Google Gemini. The credential IDs in these files are placeholders.
4. In every Google Sheets node, select your own spreadsheet (the sheet ID in the files is a placeholder).
5. Set the **Notify sales team** recipient (the files contain `sales@example.com`) and the **To** address logic in **Create follow-up draft** if needed.
6. Publish the **Error Logger**, then open the main workflow's **Settings** and select it as the **Error Workflow**.
7. Publish the main workflow and send a test lead to its **Production URL**.

Example test lead:

```json
{
  "name": "Ada Obi",
  "email": "ada@example.com",
  "phone": "+2348012345678",
  "service": "kitchen remodel",
  "budget": "8000",
  "timeline": "2 months",
  "message": "I want to renovate my kitchen soon."
}
```

## Test results

| Test | Input | Result |
|---|---|---|
| Qualified lead | Kitchen remodel, 8,000, 2 months | Qualified, tag `qualified`, draft created |
| Messy input | Capitalised email with trailing space, "$2,000", phone with spaces | Cleaned correctly |
| Unqualified lead | Swimming pool, 2,000, 8 months | Unqualified, tag `unqualified` |
| Unclear lead | Vague service, no budget, "not sure yet" | Manual Review, tag `needs-review` |
| Repeat submission | Same email, different capitalisation, new budget | Existing row updated, no duplicate |
| New qualified lead | Bathroom remodel, 12,000, 1 month | Notification email received |
| Repeat qualified lead | Same email again | No second notification |
| Live (published) run | Roofing, 9,000, 6 weeks via Production URL | Success, row created |
| Live repeat | Same lead, new budget | Single row updated, no duplicate alert |
| Forced failure | Deliberate error in the Code node | Error logged with node name and message |

## Swapping in OpenAI and GoHighLevel (untested)

- **OpenAI:** replace the Gemini node with an OpenAI message node using the same prompt. Adjust the field path in the Code node to the new node's output.
- **GoHighLevel:** replace the three Google Sheets contact nodes with contact lookup (by email), create and update, then add steps for tags and for creating or moving an opportunity to the pipeline stage. This needs a GoHighLevel Private Integration token with contacts and opportunities scopes, stored as an n8n credential.
- Keep the If-node logic, the notification rule and the error logger unchanged.

## Known limitations

- Google Sheets stands in for a real CRM, so there is no CRM-side deduplication or automation.
- The AI can be inconsistent on borderline cases. Manual review exists for this reason, and the rules can be tightened in the prompt.
- The error log row does not include the lead's email because the Error Trigger does not carry it.
- Free-tier AI models have rate limits. Retries help with temporary limits but not with quota exhaustion.
- Webhook payloads from form tools such as Tally nest answers differently from the flat test payload, so the Edit Fields expressions would need adjusting.

# Setup — Demo 2 (AI Candidate Screening)

Steps 1–3 (n8n, Google Cloud OAuth client, Gemini key) are the same as [Demo 1](setup-demo1.md).
Demo 2 also needs the **Google Sheets API** enabled (already on if you did Demo 1).

## 1. n8n credentials

| Node | Credential type |
|------|-----------------|
| New Application | Google Sheets Trigger OAuth2 API |
| Get Role Rubric, Log to Screening Sheet | Google Sheets OAuth2 API |
| Send Interview Invite, Save Reply as Gmail Draft, Notify Recruiter | Gmail OAuth2 API |
| Gemini Screening Model, Gemini Reply Model | Google Gemini(PaLM) Api |

Use the same Client ID/Secret for every Google credential and click **Sign in with Google** on each one.

## 2. Spreadsheet (one file, three tabs)

**Roles** — the screening rubric. Recruiters edit this tab; no workflow change needed for a new role.

| Role | Must Have | Nice to Have | Min Years |
|------|-----------|--------------|-----------|
| HR Generalist | Employee relations, onboarding, HR administration, labour law basics | HRIS experience, payroll, recruitment | 2 |
| Recruiter | Full-cycle recruiting, sourcing on LinkedIn, interviewing, stakeholder management | Agency background, ATS experience, employer branding | 2 |
| Payroll Specialist | Payroll processing, tax and social security calculation, Excel | HRIS or payroll software, audit support | 3 |

**Screening** — header row only; the workflow appends one row per candidate:

`Screened At | Full Name | Email | Role | Years | Score | Next Step | Strengths | Concerns | Summary | Action`

**Form Responses 1** — created by Google Forms in the next step.

## 3. Google Form

| Question | Type | Required |
|----------|------|----------|
| Full Name | Short answer | yes |
| Email | Short answer | yes |
| Role Applied For | Multiple choice (same names as the Roles tab) | yes |
| Years of Experience | Short answer | yes |
| Experience Summary | Paragraph (CV summary or LinkedIn "About") | yes |
| CV or LinkedIn URL | Short answer | no |

Question titles must match exactly: they become the sheet headers the workflow reads.

**Responses → Link to Sheets → Select existing spreadsheet** → pick the spreadsheet from step 2.
Google adds a `Form Responses 1` tab with `Timestamp` in column A.

For testing, publish with responders set to **Specific people** (Share → Responder view → Restricted).
Anyone who can submit the form can make the workflow send emails from your account.
For a real job ad you would open it to "Anyone with the link" after testing.

## 4. Import and wire the workflow

1. n8n → **Workflows → Import from file** → `workflows/hr-recruiting-demo2.json`.
2. Attach the credentials from step 1.
3. **New Application**: pick the spreadsheet and the `Form Responses 1` tab **From list** (tab IDs are random, not `0`).
4. **Get Role Rubric**: same spreadsheet, tab `Roles`.
5. **Log to Screening Sheet**: same spreadsheet, tab `Screening`.
6. **Normalize Candidate**: set
   - `recruiterEmail`: the inbox that gets the screening summaries
   - `bookingLink`: your Calendly / Cal.com / Google appointment page
   - `companyName`: used in the emails
   - `passScore`: minimum score (1–5) for an automatic interview invite, default `4`
7. Both Gemini nodes use `models/gemini-flash-lite-latest` (an alias that survives model retirements). The AI nodes retry 3 times, 5 s apart.
8. Workflow **Settings → Timezone** is `Asia/Jakarta`; change it if needed.

## 5. How the routing works

- Gemini returns a 1–5 score, 3 strengths, 2 concerns, a next step and a short summary as JSON.
- **Parse AI Output** rejects anything that is not valid JSON or has a score outside 1–5, so a bad AI answer stops the run instead of emailing a candidate.
- The route follows the **score**, not the AI's suggested next step:
  - score ≥ `passScore` → interview invite with the booking link is sent, recruiter gets a summary.
  - score < `passScore` → Gemini writes a kind hold/reject reply, which is saved as a **Gmail draft** addressed to the candidate. Nothing is sent until the recruiter opens Drafts and clicks Send.
- Every candidate is logged in the `Screening` tab, whatever the outcome.

## 6. Test

1. Submit the form twice with an email address you own: one strong candidate for a role, one weak one.
2. In n8n, open **New Application** → **Fetch Test Event**, then **Test workflow**. Repeat for the second row.
3. Check:
   - strong candidate: interview invite in the inbox + recruiter summary
   - weak candidate: reply in **Gmail Drafts** (not sent) + recruiter summary saying a draft is waiting
   - two new rows in the `Screening` tab
4. To run automatically, **activate** the workflow. It then polls the sheet every minute.

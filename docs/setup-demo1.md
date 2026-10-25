# Setup — Demo 1 (Employee Onboarding)

## 1. Run n8n locally

```bash
npx n8n
```

Open http://localhost:5678 and create the owner account.

## 2. Google Cloud (one OAuth client for Sheets, Gmail, Drive, Calendar)

1. Go to https://console.cloud.google.com → create a project (e.g. `n8n-hr-demo`).
2. **APIs & Services → Library** → enable:
   - Google Sheets API
   - Google Drive API (also needed by the Sheets trigger)
   - Gmail API
   - Google Calendar API
3. **OAuth consent screen** (Google Auth Platform):
   - User type: **External**
   - Publishing status: **Testing**
   - **Test users**: add the Google account you will log in with. Skipping this gives `access_denied` / `403`.
4. **Credentials → Create credentials → OAuth client ID**:
   - Application type: **Web application**
   - Authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`
5. Copy the **Client ID** and **Client Secret**.

## 3. Gemini API key

1. Go to https://aistudio.google.com/apikey → **Create API key**.
2. In n8n: **Credentials → New → Google Gemini(PaLM) Api** → paste the key.
   Host stays `https://generativelanguage.googleapis.com`.

## 4. n8n credentials

Create one credential per service in n8n (**Credentials → New**), each using the same Client ID/Secret:

| Node | Credential type |
|------|-----------------|
| New Hire Form Response | Google Sheets Trigger OAuth2 API |
| Copy / Share Checklist | Google Drive OAuth2 API |
| Send Welcome Email, Notify HR | Gmail OAuth2 API |
| Create Day-1 Meeting | Google Calendar OAuth2 API |
| Gemini Chat Model | Google Gemini(PaLM) Api |

Click **Sign in with Google** on each one. Google shows an "unverified app" warning in testing mode:
click **Advanced → Go to (app name)**.

## 5. Google Workspace resources

**Google Form** with these questions (titles must match exactly — they become sheet headers):

| Question | Type |
|----------|------|
| Full Name | Short answer |
| Personal Email | Short answer (email validation) |
| Position | Short answer |
| Department | Dropdown |
| Start Date | Date |
| Manager Name | Short answer |
| Manager Email | Short answer (email validation) |

**Publish** the form with responders set to **Specific people** (Share → Responder view → Restricted), not "Anyone with the link".
Anyone who can submit the form can make the workflow send emails, share files and send calendar invites from your account.

Then **Responses → Link to Sheets** → create a new spreadsheet. Google adds `Timestamp` as column A.
Optionally format the `Start Date` column as `yyyy-MM-dd` (Format → Number → Custom date and time);
the workflow also accepts `d/M/yyyy` and raw date serials.

**Drive**: create a Google Doc `Onboarding Checklist - TEMPLATE`. Copies land in the same folder.

## 6. Import and wire the workflow

1. n8n → **Workflows → Import from file** → `workflows/hr-onboarding-demo1.json`.
2. Attach the credentials from step 4 to each node.
3. **New Hire Form Response**: pick the response spreadsheet and sheet (`Form Responses 1`). Use **From list**, not By ID: tab IDs are not always `0`.
4. **Copy Checklist Template**: pick the template doc.
5. **Create Day-1 Meeting**: pick your calendar from the list. `primary` is rejected in By ID mode; use your email as the ID instead.
6. **Normalize Data**: change `hrEmail` from `hr@yourcompany.com` to a real inbox.
7. **Gemini Chat Model**: model is `models/gemini-flash-lite-latest`, an alias that always points to the newest Flash-Lite, so it survives model retirements. The AI node retries 3 times (5 s apart) on errors like `503 high demand`.
8. Workflow **Settings → Timezone** is `Asia/Jakarta`; change it if needed. The Day-1 meeting uses this timezone.

## 7. Test

1. Submit the Google Form once with an email address you own (as hire and as manager).
2. In n8n, open **New Hire Form Response** → **Fetch Test Event**. It loads the latest row.
3. Click **Test workflow** and check each node turns green.
4. Verify: welcome email received, checklist copy shared in Drive, calendar invite at 09:00, HR summary email.
5. To run automatically, **activate** the workflow. It then polls the sheet every minute.

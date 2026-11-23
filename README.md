# AI Smart Contact Form (n8n + Gemini)

A web contact form powered by an n8n workflow. A visitor submits a message, AI classifies it and writes a reply, the result is logged to Google Sheets, urgent messages trigger a Discord alert, and the AI's reply appears on the page instantly.

## How it works

```
HTML form --> n8n Webhook --> AI Agent (Gemini + Postgres memory)
                                   |
                              Code (parse JSON)
                              /      |        \
              Google Sheets   IF urgency=high   Respond to Webhook
              (log row)            |             (reply to the page)
                              Discord alert
```

1. The page sends `{ name, email, message }` as JSON to the n8n webhook.
2. The AI Agent returns JSON with `category`, `urgency`, `summary`, and `suggested_reply`.
3. A Code node parses that JSON and merges it with the form data.
4. Google Sheets appends a row, the IF node checks for high urgency, and Respond to Webhook returns `category` and `suggested_reply` to the page.

## Tech stack

| Part | Tool |
|---|---|
| Frontend | Plain HTML, CSS, JavaScript (`smart-contact-form.html`) |
| Automation | n8n (self-hosted, `localhost:5678`) |
| AI | Google Gemini Chat Model |
| Memory | Postgres Chat Memory (Neon free database) |
| Logging | Google Sheets |
| Alerts | Discord webhook |

## Prerequisites

- n8n running locally
- A Google Gemini API key (free at aistudio.google.com)
- A free Neon Postgres database (neon.tech)
- A Google Cloud project with the Google Sheets API and Google Drive API enabled, plus an OAuth Client ID and Secret
- A Google Sheet with these headers in row 1: `date`, `name`, `email`, `message`, `category`, `urgency`, `summary`
- A Discord server with a channel webhook URL

## Setup

### 1. Webhook node
- HTTP Method: `POST`
- Path: `contact`
- Respond: `Using 'Respond to Webhook' Node`
- Options: Allowed Origins (CORS) set to `*` for testing

### 2. AI Agent node
- Source for Prompt: `Define below` (Expression mode)
- Attach **Google Gemini Chat Model** and **Postgres Chat Memory**
- Prompt:

```
You are a support assistant. Analyze this message and reply ONLY with valid JSON, no extra text.

Name: {{ $json.body.name }}
Message: {{ $json.body.message }}

Return:
{
  "category": "sales | support | spam",
  "urgency": "low | medium | high",
  "summary": "one sentence summary",
  "suggested_reply": "a short, friendly reply to the sender"
}
```

Optional stricter rule: `Rate urgency as "high" if the message mentions payment problems, outages, being locked out, or says urgent or immediately.`

### 3. Postgres Chat Memory
- Credential: Neon connection with pooling turned off, port `5432`, SSL `Require`
- Session ID: `{{ $('Webhook').item.json.body.email }}`

### 4. Code node (JavaScript)

```javascript
const raw = $input.first().json.output;
const clean = raw.replace(/```json|```/g, "").trim();
const ai = JSON.parse(clean);
const form = $('Webhook').first().json.body;
return [{ json: { ...form, ...ai } }];
```

### 5. Google Sheets node
- Operation: `Append Row`
- Mapping: `date` = `{{ $now }}`, and `name`, `email`, `message`, `category`, `urgency`, `summary` = `{{ $json.<field> }}`
- Settings: On Error = `Continue`
- OAuth redirect URI in Google Cloud: `http://localhost:5678/rest/oauth2-credential/callback`

### 6. IF node and Discord node
- IF condition: `{{ $json.urgency }}` is equal to `high` (enable Ignore Case)
- Discord (true output): connection type `Webhook`, message:

```
Urgent message from {{ $json.name }} ({{ $json.email }}): {{ $json.summary }}
```

### 7. Respond to Webhook node
- Respond With: `JSON`
- Body:

```
{
  "category": "{{ $json.category }}",
  "suggested_reply": "{{ $json.suggested_reply }}"
}
```

### 8. Frontend
Open `smart-contact-form.html` and set the URL in this line:

```javascript
const WEBHOOK_URL = "http://localhost:5678/webhook-test/contact";
```

- Testing: use the `webhook-test` URL and click **Listen for test event** in n8n before each submit.
- Live: publish the workflow, then change the URL to `http://localhost:5678/webhook/contact`.

## Testing

| Test message | Expected result |
|---|---|
| "Hi, do you offer discounts?" | Reply on page, new row in the sheet, no Discord alert |
| "URGENT: my payment failed and I need help now" | Reply on page, new row in the sheet, Discord alert |

Use a new email for each test, because the memory node keeps history per email.

## Troubleshooting

| Problem | Fix |
|---|---|
| CORS error in the browser | Add Allowed Origins `*` on the Webhook node |
| Server returned 500 | The workflow has no working Respond to Webhook node, or it is not connected after the Code node |
| JSON parse error in Code node | The AI added extra text; keep "reply ONLY with valid JSON" in the prompt |
| Postgres credential fails | Set SSL to `Require`, remove `-pooler` from the host, wait a few seconds for Neon to wake up |
| Google `redirect_uri_mismatch` | Add the exact redirect URI above under Authorized redirect URIs and save |
| No Discord alert | Check that the AI rated the message `high` in the IF node output |
| Duplicate or empty sheet rows | Make sure only the Code node (not the AI Agent) connects to Google Sheets |

## Limitations

- n8n runs on your computer, so the form works only while n8n is running.
- A page hosted online cannot reach `localhost`; use a tunnel such as ngrok or Cloudflare Tunnel, or host n8n online.
- Google OAuth in Testing mode may ask you to sign in again about once a week.
- Never commit API keys, database passwords, the OAuth secret, or the Discord webhook URL to a public repository.

## Ideas for extension

- Language dropdown so replies come back in Urdu or English
- Error branch that returns a friendly message if the AI fails
- Host the page on GitHub Pages or Netlify
- Knowledge base (FAQ + vector store) so the AI answers from your own content

# Nova

**Meeting preparation that ends with an action, not just a summary.**

Nova is an AI agent that retrieves evidence from Google Calendar, Gmail, and Google Drive, evaluates it together, and prepares a grounded meeting brief and an unsent Gmail follow-up draft. It helps users arrive with the relevant context, unresolved questions, and a clear next step without manually piecing together three apps.

The interface shows actual backend progress and lets users inspect the retrieved sources. **Nova creates the email draft; it never sends email automatically.** The user reviews, edits, and sends it in Gmail.

## Demo Flow

1. Open Nova and run **“Prepare me for my Acme meeting tomorrow.”**
2. Watch Calendar, Gmail, and Drive retrieval complete with the real event, email count, and document extraction result.
3. Expand **Inspect retrieved evidence** to see source names and excerpts.
4. Read the meeting brief and inspect the evidence decisions and open questions.
5. After Gmail confirms creation, select **Draft follow-up in Gmail →** to open the connected account’s Drafts view. Select the prepared draft to review, edit, or send it.

The latest verified live run completed all seven steps in approximately 29 seconds, including a confirmed Gmail draft. This is an observed demo result, not a latency guarantee.

## Three-App Integration

- **Google Calendar:** Finds the requested Acme event in the requested date window. Supplies the meeting title, time, agenda, and attendee information used to select the draft recipient. A failed lookup does not substitute another date.
- **Gmail:** Searches attendee correspondence and meeting-related subjects, reads message bodies, deduplicates results, and filters unrelated recruiting/spam candidates. Search matches remain evidence candidates for the model to evaluate.
- **Google Drive:** Finds the exact **Acme Project** folder, uses its actual Drive ID to locate **Acme_Comprehensive_Project_Context.docx** inside it, downloads the DOCX through the API, and extracts paragraph and table text. Folder metadata is not treated as document content.

## Architecture / Workflow

```text
Browser → POST /run-agent → Flask orchestrator
  Calendar → Gmail → Drive → Evidence analysis → Resolve & commit
  → Meeting brief → Gmail draft → User review in Gmail
```

`app.py` serves both the frontend and API. A worker runs the agent while newline-delimited JSON streams real progress events to the browser. The endpoint also supports a non-streaming JSON response.

- `web/`: HTML, CSS, and JavaScript interface; no frontend build step required.
- `agent_google.py`, `google_transport.py`: Existing-token authorization, Calendar lookup, Google transport, and Gmail draft creation.
- `pipeline_step1.py`, `agent_drive.py`: Gmail retrieval and folder-scoped DOCX content extraction.
- `agent_river.py`: Evidence evaluation and brief/follow-up generation using River’s `Qwen/Qwen3.6-35B-A3B-FP8` model.
- `agent_runs/<run-id>/`: Local context, evidence decisions, brief, draft preview, result, and successful draft receipt.

The default model path uses behavioral prompting for **retrieve → resolve → commit**. Evidence evaluation identifies supported facts, conflicts, uncertainties, and client-safe points. A subsequent generation call produces the brief and follow-up. LoRA is an optional experiment, not a requirement for the demo.

## Setup and Run

The current demo is a local Python application, tested with Python 3.12 and Windows PowerShell. Run these commands from the project folder.

### Prerequisites

- Python and access to River AI, with a valid `RIVER_API_KEY`.
- A trusted, already-authorized `token_full.pickle` in the project root. Normal application startup loads and refreshes this token; it does not launch an OAuth flow.
- An accessible Acme Calendar event matching the requested day, with a suitable recipient on the event.
- Relevant Gmail correspondence and this existing Drive structure:

```text
Acme Project/
└── Acme_Comprehensive_Project_Context.docx
```

## Judge Access

Demo account:
Email: varunsrinivas2904@gmail.com

Password: Shared separately with the judging team through mail

### How to use
1. Sign into the provided Google account.
2. Open the Nova demo.
3. Run: `Prepare me for my Acme meeting tomorrow.`
4. Review the retrieved Calendar, Gmail, and Drive evidence.
5. Review the generated meeting brief.
6. Click **Draft follow-up in Gmail →** to review the generated draft.

### Install dependencies

If `.venv` is not already available:

```powershell
python -m venv .venv
```

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements-pipeline.txt
```

Set `RIVER_API_KEY` in the same shell if it is not already set:

```powershell
$env:RIVER_API_KEY = Read-Host "River API key"
```

Start the app:

```powershell
.\.venv\Scripts\python.exe app.py
```

Open [http://127.0.0.1:5000](http://127.0.0.1:5000). To use the alternate port used during demo verification:

```powershell
.\.venv\Scripts\python.exe app.py --port 5002
```

Open [http://127.0.0.1:5002](http://127.0.0.1:5002). Restart the server after backend changes; automatic reloading is disabled.

Optional environment settings are `NOVA_TIMEZONE` (default `Asia/Kolkata`), `RIVER_TIMEOUT` (default 90 seconds), and `NOVA_USE_LORA=1` for the experimental checkpoint path using `checkpoint_path.txt`.

## Technical Highlights

- Real retrieval from three external APIs followed by a real `users.drafts.create` action.
- Source IDs validated against retrieved evidence, structured JSON validation, and bounded response-correction attempts. Internal reasoning text is not used as a final-answer fallback.
- Full DOCX text extraction for the configured document, with explicit size limits and errors for missing or ambiguous targets.
- Seven live steps, inspectable source excerpts, and clear partial/failure states. Draft success requires a returned Gmail draft ID.
- One active run per server process; draft creation is not automatically retried after an uncertain response.

## Security / Permissions

The demo uses Google OAuth authorization for `calendar.readonly`, `drive.readonly`, and `gmail.readonly`, plus Gmail draft access. The permission helper adds `gmail.compose`; the application also recognizes existing `gmail.modify` or full Gmail access. Compose authorization permits sending at the API level, but **Nova only calls draft creation and never calls a send endpoint**.

A read-only Gmail token can support preparation but cannot create the draft. The UI reports this explicitly. If the account owner chooses to grant draft access, the existing helper is:

```powershell
.\.venv\Scripts\python.exe authorize_gmail_drafts.py --authorize
```

This explicit command requires `client_secret.json`, opens Google consent, checks that the account matches, and backs up the original token before replacement. It is not part of normal startup.

Retrieved Calendar, email, and document content is sent to River AI for processing and saved locally in `agent_runs/`. Keep these records private. `.gitignore` excludes run records, Google credentials, token files, and `.env` files. Only load trusted pickle tokens. The app binds to localhost and has no multi-user authentication; it is intended for a local demo.

## Verification

Run the regression suite:

```powershell
.\.venv\Scripts\python.exe -m unittest test_drive test_pipeline test_app test_calendar_transport
```

Focused real-API checks use the existing Google token:

```powershell
.\.venv\Scripts\python.exe test_live_calendar.py
.\.venv\Scripts\python.exe test_live_drive.py
```

The Calendar check expects **Acme Meeting tomorrow**. The Drive check reports the actual folder/file IDs, downloaded bytes, and extracted character and paragraph counts. These checks do not create drafts. Running the complete flow through the UI does create an unsent draft when authorized.

## Known Limitations

- The demo is deliberately scoped to Acme meetings and the exact Drive folder/document above. Supported date requests include today, tomorrow, an ISO date, or the next Acme meeting within 30 days.
- Gmail retrieval covers the last 180 days, with bounded searches and at most 18 retained candidates. Message bodies are capped at 10,000 characters; this is not exhaustive inbox search.
- DOCX extraction reads main-document paragraphs and tables, not rendered pages or image OCR. Downloads over 10 MB or extracted text over 200,000 characters are rejected.
- The Gmail handoff opens the account’s Drafts view, not a supported deep link to an individual draft. Separate successful runs can create separate drafts.
- Model judgments still require review; citation validation checks source identity, not the truth of every claim. External API availability and generation latency affect completion. LoRA remains experimental and falls back to behavioral prompting when unavailable.

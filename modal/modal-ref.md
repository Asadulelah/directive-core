Modal Cloud Execution Framework

This document explains how to convert local Python execution scripts into cloud-hosted Modal webhooks. It provides everything needed to replicate the pattern established in this codebase.

---

## Quick Start for Users

If you're setting this up for the first time, follow these steps:

### 1. Install Modal CLI

```bash
pip install modal
modal token new  # Authenticate with Modal
```

### 2. Check What Secrets Exist

```bash
modal secret list
```

### 3. Identify Missing Secrets

Compare what exists against what the workflow needs. See [Identifying Required Secrets](#identifying-required-secrets) below.

### 4. Create Missing Secrets

For each missing secret, you'll need to:
1. Get the API key from the service's dashboard
2. Create it in Modal:
   ```bash
   modal secret create <secret-name> <ENV_VAR_NAME>=<value>
   ```

See [Handling Missing Secrets](#handling-missing-secrets) for detailed guidance on what to do when you don't have all credentials.

### 5. Deploy

```bash
modal deploy execution/modal_webhook.py
```

### 6. Test

```bash
curl "https://<your-modal-url>/endpoint?param=value"
```

---

## Overview

The DOE (Directive-Orchestration-Execution) framework separates concerns:
- **Directives**: SOPs in markdown describing what to do
- **Orchestration**: Claude (you) making decisions and calling tools
- **Execution**: Deterministic Python scripts doing the actual work

This document focuses on moving the **Execution** layer to the cloud via Modal, so you (the orchestrator) can trigger workflows via HTTP webhooks instead of local Python scripts.

---

## Architecture Pattern

### The Problem with Agent-on-Server
Running Claude on Modal (like an "agent endpoint") is **slow** (~2-3 minutes per request) due to:
- Cold start times
- API latency for each Claude call
- Extended thinking overhead

### The Solution: Execution-Only Webhooks
Instead of running Claude on Modal, we run **deterministic scripts** on Modal and let the **local Claude agent orchestrate** by calling these webhooks.

```
┌─────────────────┐      HTTP       ┌─────────────────┐
│  Local Claude   │ ──────────────> │  Modal Webhook  │
│  (Orchestrator) │                 │  (Execution)    │
└─────────────────┘                 └─────────────────┘
         │                                   │
         │ Reads directives                  │ Runs scripts
         │ Makes decisions                   │ Calls APIs
         │ Formats inputs                    │ Updates sheets
         └───────────────────────────────────┘
```

---

## Modal Webhook File Structure

All Modal webhooks live in a single file: `execution/modal_webhook.py`

### Core Components

```python
import modal
import os
import json
import logging
from datetime import datetime

# 1. Define the Modal app
app = modal.App("claude-orchestrator")

# 2. Create image with dependencies
image = (
    modal.Image.debian_slim(python_version="3.11")
    .pip_install(
        "anthropic",
        "fastapi",
        "google-auth",
        "google-auth-oauthlib",
        "google-api-python-client",
        "requests",
        "apify-client",
        "gspread",
        "pandas",
        "python-dotenv",
    )
    # Mount local directories
    .add_local_dir("/path/to/directives", remote_path="/app/directives")
    .add_local_dir("/path/to/execution", remote_path="/app/execution")
)

# 3. Define secrets (API keys stored in Modal dashboard)
ALL_SECRETS = [
    modal.Secret.from_name("anthropic-secret"),
    modal.Secret.from_name("google-token"),
    modal.Secret.from_name("slack-webhook"),
    modal.Secret.from_name("apify-secret"),
    modal.Secret.from_name("anymailfinder-secret"),
    # Add more as needed
]

# 4. Logger
logger = logging.getLogger(__name__)
```

---

## The Async Pattern: Immediate Response + Background Task

For long-running workflows, use this pattern:

### Step 1: Create a Background Function

```python
@app.function(image=image, secrets=ALL_SECRETS, timeout=1800)  # 30 min timeout
def my_workflow_background(param1: str, param2: str, sheet_id: str, sheet_url: str):
    """
    Background task that does the actual work.
    This runs AFTER the endpoint returns 201.
    """
    try:
        slack_notify(f"Step 1/N: Starting...")

        # Do work here
        # ...

        slack_notify(f"Step 2/N: Processing...")

        # More work
        # ...

        slack_notify(f"Complete!")
        return {"status": "success", "result": "..."}

    except Exception as e:
        slack_error(f"Failed: {str(e)}")
        return {"status": "error", "error": str(e)}
```

### Step 2: Create the Endpoint that Spawns It

```python
@app.function(image=image, secrets=ALL_SECRETS, timeout=60)
@modal.fastapi_endpoint(method="GET")
def my_workflow(param1: str = "", param2: str = "default"):
    """
    Endpoint that returns 201 immediately, spawns background task.

    URL: GET /my-workflow?param1=value&param2=value
    """
    from fastapi.responses import JSONResponse

    if not param1:
        return JSONResponse({
            "status": "error",
            "error": "Missing 'param1' parameter"
        }, status_code=400)

    try:
        # Create any resources needed (e.g., Google Sheet)
        sheet_url = create_google_sheet(...)
        sheet_id = extract_sheet_id(sheet_url)

        # Notify that we're starting
        slack_notify(f"Workflow started: {param1}")

        # Spawn background task (non-blocking!)
        my_workflow_background.spawn(param1, param2, sheet_id, sheet_url)

        # Return 201 immediately
        return JSONResponse({
            "status": "accepted",
            "message": "Workflow started. Monitor Slack for progress.",
            "sheet_url": sheet_url,
            "workflow": [
                "1. First step description",
                "2. Second step description",
                "3. Third step description"
            ]
        }, status_code=201)

    except Exception as e:
        slack_error(f"Init failed: {str(e)}")
        return JSONResponse({
            "status": "error",
            "error": str(e)
        }, status_code=500)
```

---

## Setting Up Modal Secrets

Secrets are stored in the Modal dashboard, not in code. Create them via CLI:

```bash
# Anthropic API Key
modal secret create anthropic-secret ANTHROPIC_API_KEY=sk-ant-...

# Google OAuth Token (JSON blob)
modal secret create google-token GOOGLE_TOKEN_JSON='{"token":"...","refresh_token":"...","token_uri":"...","client_id":"...","client_secret":"...","scopes":[...]}'

# Slack Webhook for notifications
modal secret create slack-webhook SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...

# Apify API Token
modal secret create apify-secret APIFY_API_TOKEN=apify_api_...

# AnyMailFinder API Key
modal secret create anymailfinder-secret ANYMAILFINDER_API_KEY=...

# PandaDoc API Key
modal secret create pandadoc-secret PANDADOC_API_KEY=...
```

### Using Secrets in Code

```python
# Secrets are exposed as environment variables
api_key = os.getenv("ANTHROPIC_API_KEY")
token_data = json.loads(os.getenv("GOOGLE_TOKEN_JSON"))
```

---

## Slack Notifications

Every workflow should send Slack notifications for visibility:

```python
def slack_notify(message: str):
    """Send notification to Slack."""
    import requests
    webhook_url = os.getenv("SLACK_WEBHOOK_URL")
    if webhook_url:
        try:
            requests.post(webhook_url, json={"text": message}, timeout=5)
        except:
            pass  # Don't fail workflow if Slack fails

def slack_error(error: str):
    """Send error notification to Slack."""
    slack_notify(f"❌ *Error*: {error}")
```

Use notifications at key points:
- Workflow started
- Each major step completed
- Errors encountered
- Workflow completed

---

## Identifying Required Secrets

Every workflow has dependencies on external services. Before deploying, audit your workflow to identify what secrets are needed.

### Reading the Code for Secret Dependencies

Look for these patterns in the execution code:

```python
# Direct environment variable access
os.getenv("SOME_API_KEY")
os.environ["SOME_TOKEN"]

# Anthropic client initialization
anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

# Requests with Authorization headers
headers={"Authorization": f"Bearer {os.getenv('SERVICE_API_KEY')}"}

# Client libraries with tokens
SomeClient(token=os.getenv("SERVICE_TOKEN"))
```

### Common Secret Categories

| Category | Secret Name | Environment Variable | Where to Get |
|----------|-------------|---------------------|--------------|
| **LLM** | anthropic-secret | ANTHROPIC_API_KEY | console.anthropic.com |
| **LLM** | openai-secret | OPENAI_API_KEY | platform.openai.com |
| **Google** | google-token | GOOGLE_TOKEN_JSON | Google Cloud Console + OAuth flow |
| **Notifications** | slack-webhook | SLACK_WEBHOOK_URL | Slack App settings |
| **Scraping** | apify-secret | APIFY_API_TOKEN | console.apify.com |
| **Email Enrichment** | anymailfinder-secret | ANYMAILFINDER_API_KEY | anymailfinder.com dashboard |
| **Documents** | pandadoc-secret | PANDADOC_API_KEY | PandaDoc developer settings |
| **Email Sending** | sendgrid-secret | SENDGRID_API_KEY | sendgrid.com |
| **CRM** | hubspot-secret | HUBSPOT_API_KEY | HubSpot developer account |
| **General** | env-vars | (multiple) | Your `.env` file |

### Workflow to Secret Mapping

For each workflow step, identify the service:

```
Workflow Step              → Service       → Secret Needed
─────────────────────────────────────────────────────────
Scrape leads              → Apify         → apify-secret
Upload to Google Sheet    → Google APIs   → google-token
Enrich emails             → AnyMailFinder → anymailfinder-secret
AI text processing        → Anthropic     → anthropic-secret
Send Slack notification   → Slack         → slack-webhook
Create document           → PandaDoc      → pandadoc-secret
Send email                → SendGrid      → sendgrid-secret
```

---

## Handling Missing Secrets

When you don't have all the required credentials, follow this decision tree.

### Step 1: List What's Missing

```bash
# See what you have
modal secret list

# Compare against what the workflow needs (from code audit)
# Missing = Required - Available
```

### Step 2: Categorize the Missing Secrets

**Critical (workflow won't run):**
- Primary data source (e.g., Apify for scraping)
- Primary output destination (e.g., Google Sheets)

**Optional (workflow runs with reduced functionality):**
- Slack notifications (workflow still completes, just no visibility)
- Email enrichment (data still collected, just fewer emails)

**Alternative available:**
- Multiple LLM providers (OpenAI vs Anthropic)
- Multiple scraping services (Apify vs custom)

### Step 3: Actions for Missing Secrets

#### If the user doesn't have the credential:

1. **Ask them to create an account** with the service
2. **Provide direct link** to the API key page
3. **Explain what the service does** and why it's needed
4. **Offer alternatives** if they exist

Example prompt to user:
```
The workflow requires an Apify API token for lead scraping.

To get one:
1. Create account at https://console.apify.com
2. Go to Settings → Integrations → API
3. Copy your Personal API Token
4. Run: modal secret create apify-secret APIFY_API_TOKEN=your_token_here

Alternative: If you have leads in another format (CSV, JSON), I can skip the scrape step and process those directly.
```

#### If the credential exists but isn't in Modal:

```bash
# Check local .env
grep "SOME_API_KEY" .env

# If found, create the Modal secret
modal secret create service-secret SOME_API_KEY=$(grep SOME_API_KEY .env | cut -d= -f2)
```

#### If the service is optional:

Modify the workflow to gracefully skip that step:

```python
# In the background function
amf_key = os.getenv("ANYMAILFINDER_API_KEY")
if amf_key:
    slack_notify("📧 Enriching emails...")
    # ... enrichment code ...
else:
    slack_notify("⚠️ Skipping email enrichment (no API key)")
```

### Step 4: Deploy with Available Secrets

Update the `ALL_SECRETS` list to only include secrets that exist:

```python
# Build secrets list dynamically or comment out missing ones
ALL_SECRETS = [
    modal.Secret.from_name("anthropic-secret"),
    modal.Secret.from_name("google-token"),
    modal.Secret.from_name("slack-webhook"),
    # modal.Secret.from_name("apify-secret"),  # TODO: User needs to add
]
```

### Common "Missing Secret" Scenarios

| Scenario | Solution |
|----------|----------|
| No Google token | Run OAuth flow locally, create secret from token.json |
| No Slack webhook | Workflow works, just no notifications (use logging) |
| No scraping API | User provides data manually (CSV/JSON upload) |
| No email enrichment | Skip step, user enriches manually or uses another tool |
| No LLM API | Cannot process text; must get key or use alternative |

---

## Generic Integration Patterns

These patterns work with ANY external service, not just the specific examples shown elsewhere.

### Pattern 1: REST API with Bearer Token

Most APIs use this pattern:

```python
def call_any_api(endpoint: str, method: str = "GET", data: dict = None) -> dict:
    """Generic pattern for Bearer token APIs."""
    import requests

    api_key = os.getenv("SERVICE_API_KEY")
    base_url = "https://api.service.com/v1"

    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }

    if method == "GET":
        response = requests.get(f"{base_url}/{endpoint}", headers=headers, timeout=30)
    elif method == "POST":
        response = requests.post(f"{base_url}/{endpoint}", headers=headers, json=data, timeout=30)

    response.raise_for_status()
    return response.json()
```

**Services using this pattern:** OpenAI, Anthropic, SendGrid, HubSpot, Notion, Airtable, etc.

### Pattern 2: API Key in Header

Some APIs use custom header names:

```python
def call_custom_header_api(endpoint: str, data: dict = None) -> dict:
    """Pattern for APIs with custom auth headers."""
    import requests

    api_key = os.getenv("SERVICE_API_KEY")

    headers = {
        "X-API-Key": api_key,  # or "Api-Key", "apikey", etc.
        "Content-Type": "application/json"
    }

    response = requests.post(
        f"https://api.service.com/{endpoint}",
        headers=headers,
        json=data,
        timeout=30
    )
    response.raise_for_status()
    return response.json()
```

**Services using this pattern:** AnyMailFinder, some legacy APIs

### Pattern 3: OAuth 2.0 with Refresh Token

For services requiring user authorization:

```python
def get_oauth_client():
    """Pattern for OAuth 2.0 services."""
    from google.oauth2.credentials import Credentials
    from google.auth.transport.requests import Request

    token_data = json.loads(os.getenv("SERVICE_TOKEN_JSON"))

    creds = Credentials(
        token=token_data["token"],
        refresh_token=token_data["refresh_token"],
        token_uri=token_data["token_uri"],
        client_id=token_data["client_id"],
        client_secret=token_data["client_secret"],
        scopes=token_data["scopes"]
    )

    # Auto-refresh if expired
    if creds.expired and creds.refresh_token:
        creds.refresh(Request())

    return creds
```

**Services using this pattern:** Google APIs, Microsoft Graph, Salesforce, etc.

### Pattern 4: SDK Client Libraries

Many services provide Python SDKs:

```python
def use_sdk_client():
    """Pattern for services with official SDKs."""
    from service_sdk import ServiceClient

    # Most SDKs accept API key in constructor
    client = ServiceClient(api_key=os.getenv("SERVICE_API_KEY"))

    # Or via environment variable automatically
    # client = ServiceClient()  # Reads SERVICE_API_KEY from env

    return client
```

**Examples:**
- `ApifyClient(token=os.getenv("APIFY_API_TOKEN"))`
- `anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))`
- `openai.OpenAI(api_key=os.getenv("OPENAI_API_KEY"))`
- `stripe.api_key = os.getenv("STRIPE_API_KEY")`

### Pattern 5: Webhook Callbacks

For services that push data to you:

```python
@app.function(image=image, secrets=ALL_SECRETS)
@modal.fastapi_endpoint(method="POST")
def webhook_receiver(request_data: dict):
    """Pattern for receiving webhooks from external services."""
    from fastapi.responses import JSONResponse

    # Verify webhook signature (most services provide this)
    signature = request_data.get("signature")
    secret = os.getenv("WEBHOOK_SECRET")

    if not verify_signature(signature, secret, request_data):
        return JSONResponse({"error": "Invalid signature"}, status_code=401)

    # Process the webhook payload
    event_type = request_data.get("event_type")

    if event_type == "order.created":
        handle_order(request_data)
    elif event_type == "payment.completed":
        handle_payment(request_data)

    return JSONResponse({"status": "ok"}, status_code=200)
```

### Adding a New Service Integration

When you need to integrate a new service:

1. **Find the API documentation**
2. **Identify the auth pattern** (Bearer? API Key? OAuth? SDK?)
3. **Create the secret:**
   ```bash
   modal secret create newservice-secret NEW_SERVICE_API_KEY=xxx
   ```
4. **Add to ALL_SECRETS:**
   ```python
   ALL_SECRETS = [
       # ... existing secrets ...
       modal.Secret.from_name("newservice-secret"),
   ]
   ```
5. **Write the integration function** using the appropriate pattern above
6. **Add error handling and retries** for production use

---

## Google Sheets Integration

### Creating a Sheet

```python
import gspread
from google.oauth2.credentials import Credentials as UserCredentials
from google.auth.transport.requests import Request

def create_sheet(name: str) -> tuple[str, str]:
    """Create a Google Sheet, return (sheet_id, sheet_url)."""
    token_data = json.loads(os.getenv("GOOGLE_TOKEN_JSON"))
    creds = UserCredentials(
        token=token_data["token"],
        refresh_token=token_data["refresh_token"],
        token_uri=token_data["token_uri"],
        client_id=token_data["client_id"],
        client_secret=token_data["client_secret"],
        scopes=token_data["scopes"]
    )
    if creds.expired:
        creds.refresh(Request())

    gc = gspread.authorize(creds)
    sh = gc.create(name)
    return sh.id, sh.url
```

### Uploading Data with Pandas

```python
import pandas as pd

def upload_to_sheet(sheet_id: str, data: list[dict]):
    """Upload JSON data to sheet using pandas for proper flattening."""
    # ... auth code ...

    gc = gspread.authorize(creds)
    sh = gc.open_by_key(sheet_id)
    worksheet = sh.get_worksheet(0)

    # Flatten nested JSON
    df = pd.json_normalize(data)
    df = df.fillna("")  # Convert NaN to empty strings

    # Prepare rows
    headers = df.columns.tolist()
    rows = [headers] + df.values.tolist()

    # Resize if needed
    if len(rows) > worksheet.row_count or len(headers) > worksheet.col_count:
        worksheet.resize(rows=len(rows), cols=len(headers))

    # Upload
    worksheet.update(values=rows, range_name="A1")
```

---

## Example Integrations

The following are concrete examples of the generic patterns above. Use these as templates when integrating similar services.

### Apify (Lead Scraping) - SDK Pattern

```python
from apify_client import ApifyClient

def scrape_leads(query: str, location: str, limit: int) -> list[dict]:
    client = ApifyClient(os.getenv("APIFY_API_TOKEN"))

    run_input = {
        "fetch_count": limit,
        "contact_job_title": [query],
        "company_keywords": [query],
        "contact_location": [location.lower()],
        "language": "en",
    }

    run = client.actor("code_crafter/leads-finder").call(run_input=run_input)

    results = []
    for item in client.dataset(run["defaultDatasetId"]).iterate_items():
        results.append(item)

    return results
```

### AnyMailFinder (Email Enrichment) - Custom Header Pattern

```python
import requests

def enrich_email(first_name: str, last_name: str, domain: str, company_name: str) -> str | None:
    api_key = os.getenv("ANYMAILFINDER_API_KEY")

    response = requests.post(
        "https://api.anymailfinder.com/v5.1/find-email/person",
        headers={"Authorization": api_key, "Content-Type": "application/json"},
        json={
            "first_name": first_name,
            "last_name": last_name,
            "domain": domain,
            "company_name": company_name
        },
        timeout=30
    )

    if response.status_code == 200:
        return response.json().get("email")
    return None
```

### Claude (Haiku for Batch Processing) - SDK Pattern

```python
import anthropic

def casualize_batch(records: list[dict]) -> list[dict]:
    """Use Claude Haiku to casualize names in batch."""
    client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

    records_json = json.dumps(records)

    prompt = f"""Convert to casual forms for cold emails. Return ONLY valid JSON array.

Rules:
- first_name: Common nicknames (William→Will, Jennifer→Jen)
- company_name: Remove legal suffixes, generic words
- city: Local nicknames (San Francisco→SF)

Input: {records_json}

Output JSON only:"""

    message = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=6000,
        messages=[{"role": "user", "content": prompt}]
    )

    response_text = message.content[0].text.strip()
    if response_text.startswith("```"):
        lines = response_text.split('\n')
        response_text = '\n'.join(lines[1:-1])

    return json.loads(response_text)
```

### PandaDoc (Proposal Generation) - API Key Header Pattern

```python
import requests

def create_proposal(client_info: dict, tokens: list[dict]) -> str:
    """Create a PandaDoc document from template."""
    api_key = os.getenv("PANDADOC_API_KEY")
    template_id = "YOUR_TEMPLATE_UUID"

    payload = {
        "name": f"Proposal - {client_info['company']}",
        "template_uuid": template_id,
        "recipients": [{
            "email": client_info["email"],
            "first_name": client_info["first_name"],
            "last_name": client_info["last_name"],
            "role": "Client"
        }],
        "tokens": tokens
    }

    response = requests.post(
        "https://api.pandadoc.com/public/v1/documents",
        headers={
            "Authorization": f"API-Key {api_key}",
            "Content-Type": "application/json"
        },
        json=payload
    )
    response.raise_for_status()

    doc_id = response.json()["id"]
    return f"https://app.pandadoc.com/a/#/documents/{doc_id}"
```

---

## Deploying

```bash
# From the workspace root
modal deploy execution/modal_webhook.py

# View logs
modal logs claude-orchestrator

# Run a specific function locally for testing
modal run execution/modal_webhook.py::my_function
```

---

## Complete Example: Lead Scraping Workflow

Here's the full pattern we established for the lead scraping workflow:

### Directive: `directives/scrape_leads.md`

The directive describes:
1. Scrape leads via Apify
2. Upload to Google Sheet (preserving all fields)
3. Enrich missing emails with AnyMailFinder
4. Casualize first_name, company_name, city

### Modal Implementation

```python
# Background task - does all the work
@app.function(image=image, secrets=ALL_SECRETS, timeout=1800)
def scrape_leads_background(query: str, location: str, limit: int, sheet_id: str, sheet_url: str):
    import pandas as pd
    from apify_client import ApifyClient
    import anthropic
    import gspread

    try:
        # Step 1: Scrape with Apify
        slack_notify(f"📥 Step 1/4: Scraping {limit} leads...")
        apify_client = ApifyClient(os.getenv("APIFY_API_TOKEN"))
        run = apify_client.actor("code_crafter/leads-finder").call(run_input={
            "fetch_count": limit,
            "contact_job_title": [query],
            "company_keywords": [query],
            "contact_location": [location.lower()],
            "language": "en",
        })
        results = list(apify_client.dataset(run["defaultDatasetId"]).iterate_items())

        if not results:
            slack_notify("⚠️ No leads found")
            return {"status": "no_results"}

        # Step 2: Upload to Sheet
        slack_notify(f"📤 Step 2/4: Uploading {len(results)} leads...")
        # ... auth and upload with pandas ...
        df = pd.json_normalize(results)
        # Add casual columns
        if "first_name" in df.columns:
            df.insert(df.columns.get_loc("first_name") + 1, "casual_first_name", "")
        if "company_name" in df.columns:
            df.insert(df.columns.get_loc("company_name") + 1, "casual_company_name", "")
        if "city" in df.columns:
            df.insert(df.columns.get_loc("city") + 1, "casual_city_name", "")
        # ... upload to sheet ...

        # Step 3: Enrich emails
        slack_notify(f"📧 Step 3/4: Enriching emails...")
        # ... call AnyMailFinder for rows missing email ...

        # Step 4: Casualize
        slack_notify(f"✨ Step 4/4: Casualizing names...")
        # ... batch process with Claude Haiku ...

        slack_notify(f"✅ Complete! {len(results)} leads → {sheet_url}")
        return {"status": "success", "leads_found": len(results)}

    except Exception as e:
        slack_error(f"Failed: {str(e)}")
        return {"status": "error", "error": str(e)}


# Endpoint - returns 201 immediately
@app.function(image=image, secrets=ALL_SECRETS, timeout=60)
@modal.fastapi_endpoint(method="GET")
def scrape_leads(query: str = "", location: str = "United States", limit: int = 100):
    from fastapi.responses import JSONResponse

    if not query:
        return JSONResponse({"status": "error", "error": "Missing query"}, status_code=400)

    # Create sheet
    sheet_id, sheet_url = create_sheet(f"Leads - {query} - {datetime.utcnow()}")

    slack_notify(f"🚀 Started: {query}, {location}, {limit} leads")

    # Spawn background (non-blocking)
    scrape_leads_background.spawn(query, location, limit, sheet_id, sheet_url)

    # Return immediately
    return JSONResponse({
        "status": "accepted",
        "sheet_url": sheet_url,
        "workflow": ["1. Scraping", "2. Uploading", "3. Enriching", "4. Casualizing"]
    }, status_code=201)
```

### Calling from Local Agent

```bash
curl "https://your-app--claude-orchestrator-scrape-leads.modal.run?query=dentists&location=United+States&limit=100"
```

Response (instant):
```json
{
  "status": "accepted",
  "sheet_url": "https://docs.google.com/spreadsheets/d/...",
  "workflow": ["1. Scraping", "2. Uploading", "3. Enriching", "4. Casualizing"]
}
```

Monitor progress via Slack notifications.

---

## Converting a Directive to Modal Webhook

### Step-by-Step Process

1. **Read the directive** to understand the workflow steps
2. **Identify inputs** (what parameters does the endpoint need?)
3. **Identify outputs** (Google Sheet? PandaDoc? Email?)
4. **List API dependencies** (Apify? AMF? Claude?)
5. **Create the background function** with all workflow steps
6. **Create the endpoint** that spawns it
7. **Add secrets** if new APIs are needed
8. **Deploy and test**

### Checklist

- [ ] Background function has appropriate timeout (1800s for long workflows)
- [ ] Endpoint returns 201 with useful info (URLs, workflow steps)
- [ ] Each step sends Slack notification
- [ ] Errors are caught and reported via Slack
- [ ] All secrets are created in Modal dashboard
- [ ] Secrets are added to `ALL_SECRETS` list

---

## Troubleshooting

### Deployment Errors

**"Secret not found: xyz-secret"**
```bash
# List available secrets
modal secret list

# Create the missing secret
modal secret create xyz-secret XYZ_API_KEY=your_key_here
```

**"Module not found: xyz"**
Add the missing package to the image definition:
```python
image = (
    modal.Image.debian_slim(python_version="3.11")
    .pip_install(
        # ... existing packages ...
        "xyz",  # Add missing package
    )
)
```

**"Function was terminated by signal"**
- Increase timeout: `@app.function(timeout=1800)`
- Check for infinite loops in your code
- Ensure you're not using streaming APIs (use direct calls instead)

### Runtime Errors

**Slow responses from agent endpoint**
- Don't run Claude orchestration on Modal—it's too slow
- Use execution-only webhooks instead
- Let local Claude agent orchestrate, Modal just executes

**Google Sheets auth errors**
- Ensure GOOGLE_TOKEN_JSON includes `refresh_token`
- Check scopes include:
  - `https://www.googleapis.com/auth/spreadsheets`
  - `https://www.googleapis.com/auth/drive`
- If token expired, re-run OAuth flow locally and update the secret

**Rate limits / 429 errors**
Add exponential backoff:
```python
import time

def with_retry(func, max_retries=3):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if "429" in str(e) and attempt < max_retries - 1:
                time.sleep(2 ** attempt)
                continue
            raise
```

**Background task not running**
- Check Modal dashboard for function logs
- Ensure `.spawn()` is called (not `.call()`)
- Verify function has correct secrets attached

### Debugging Workflow Steps

**No Slack notifications**
1. Check if `SLACK_WEBHOOK_URL` is set correctly
2. Test webhook directly:
   ```bash
   curl -X POST -H 'Content-type: application/json' \
     --data '{"text":"Test"}' \
     $SLACK_WEBHOOK_URL
   ```

**Data not appearing in Google Sheet**
1. Check if sheet was created (look for sheet_url in response)
2. Verify sheet permissions (should be editable by service account)
3. Check for errors in background task logs

**Empty results from scraping**
1. Test the query locally first:
   ```bash
   python3 execution/scrape_apify.py --query "test" --location "US" --max_items 5
   ```
2. Verify Apify API token is valid
3. Check Apify dashboard for run status

### Viewing Logs

```bash
# View all logs for the app
modal logs claude-orchestrator

# View logs for a specific function (last 100 lines)
modal logs claude-orchestrator --function scrape_leads_background

# Follow logs in real-time
modal logs claude-orchestrator --follow
```

### Testing Locally Before Deploying

```bash
# Run a function locally (uses Modal's remote execution)
modal run execution/modal_webhook.py::function_name

# Test with specific arguments
modal run execution/modal_webhook.py::scrape_leads_background \
  --query "dentists" --location "US" --limit 10
```

---

## File Locations

```
execution/modal_webhook.py    # All Modal webhooks
directives/*.md               # SOPs describing workflows
.env                          # Local environment variables (not deployed)
```

### Migration Checklist

When moving from Modal to another platform:

1. [ ] Replace `@app.function()` with platform's function decorator
2. [ ] Replace `modal.Secret.from_name()` with platform's secret access
3. [ ] Replace `.spawn()` with platform's async job system
4. [ ] Replace `@modal.fastapi_endpoint()` with platform's HTTP trigger
5. [ ] Update image/dependencies to platform's format
6. [ ] Configure appropriate timeouts for long-running tasks

---

## Summary

1. **Don't run Claude on the server** - orchestration is too slow remotely
2. **Use execution-only webhooks** - fast, deterministic, testable
3. **Return 201/202 immediately** - spawn background task for long work
4. **Send notifications** - visibility into progress (Slack, email, etc.)
5. **Use pandas for data** - proper JSON flattening to sheets
6. **Match directive structure** - same steps, same outputs as local scripts
7. **Audit secrets before deploying** - know what credentials you need
8. **Use generic patterns** - Bearer tokens, OAuth, SDKs work everywhere
9. **Handle missing credentials gracefully** - skip optional steps, guide users



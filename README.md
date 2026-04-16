# MarkApp Automation Hub

> GitHub Actions workflows for Pantheon (SSP), Harion (DSP), and MarkDash (Reporting & Ops).

---

## Workflows

| Workflow | File | Trigger | Purpose |
|---|---|---|---|
| Slack Notifications | `slack-notifications.yml` | Schedule / Manual | Daily summaries, anomaly alerts, performance alerts |
| Daily Reporting | `daily-reporting.yml` | Schedule / Manual | Pull MarkDash data → Google Sheets → Slack summary |
| CI/CD Deploy | `ci-cd-deploy.yml` | Push / Manual | Test → Build → Stage → Production deploy |
| Doc Generation | `doc-generation.yml` | Webhook / Manual | Generate proposals, contracts, media plans, invoices |

---

## Setup

### 1. Required Secrets

Go to **Settings → Secrets and variables → Actions → New repository secret** and add:

#### Slack
```
SLACK_WEBHOOK_URL          Incoming webhook URL from Slack App
SLACK_CHANNEL_ALERTS       Channel ID for alerts (e.g. C01234ABCDE)
SLACK_CHANNEL_REPORTS      Channel ID for reports
```

#### MarkDash
```
MARKDASH_API_KEY           Your MarkDash API key
MARKDASH_API_URL           MarkDash base API URL
MARKDASH_DASHBOARD_URL     Dashboard URL for Slack deep links
```

#### Google
```
GOOGLE_SERVICE_ACCOUNT     JSON string of service account credentials
GOOGLE_SHEETS_ID           Target Google Sheet ID
GOOGLE_DRIVE_FOLDER_ID     Target Drive folder ID for docs
```

#### Deploy (CI/CD)
```
STAGING_DEPLOY_KEY         SSH key or deploy token for staging
STAGING_HOST               Staging server hostname/IP
STAGING_URL                Staging base URL (for smoke tests)
PRODUCTION_DEPLOY_KEY      SSH key or deploy token for production
PRODUCTION_HOST            Production server hostname/IP
PRODUCTION_URL             Production base URL
```

#### Doc Generation
```
SENDGRID_API_KEY           SendGrid API key for email delivery
DOCUSIGN_API_KEY           DocuSign API key (optional, for e-signing)
```

---

## Usage

### Slack Notifications

**Auto:** Runs Mon–Fri at 08:00 UTC (daily summary) and every 2 hours (anomaly check).

**Manual:**
1. Go to **Actions → Slack Notifications → Run workflow**
2. Select alert type: `daily_summary`, `performance_alert`, `anomaly_alert`, or `custom`

**Anomaly thresholds** (set in workflow env):
- CTR drop > 20%
- Fill rate drop > 15%
- Spend spike > 30%

---

### Daily Reporting Pipeline

**Auto:** Weekdays at 07:00 UTC (daily) · Mondays at 06:00 UTC (weekly).

**Manual:**
1. Go to **Actions → Daily Reporting Pipeline → Run workflow**
2. Choose: `daily`, `weekly`, `monthly`, or `custom_range`
3. For `custom_range`: provide `date_from` and `date_to` (YYYY-MM-DD)

**What it does:**
- Fetches performance data from MarkDash API
- Processes & formats via Python pandas
- Pushes to Google Sheets (auto-updates existing sheet)
- Sends summary card to Slack with link to MarkDash dashboard
- Saves report artifact to GitHub for 30 days

---

### CI/CD Deploy Pipeline

**Auto:** Triggers on push to `main` or `release/**` branches.

**Manual:**
1. Go to **Actions → CI/CD Deploy Pipeline → Run workflow**
2. Select target: `staging`, `production`, `pantheon`, `harion`, `markdash`, or `all`
3. Optionally enable `force_deploy` to skip tests

**Pipeline stages:**
1. `test` — Lint + unit tests (Node + Python)
2. `build` — Docker image build & push to GHCR
3. `deploy-staging` — Auto-deploy + smoke test
4. `deploy-production` — Requires manual approval (GitHub Environment protection rule)

**To enable production approval gate:**
- Go to **Settings → Environments → production → Add protection rule → Required reviewers**

---

### Doc & Contract Generation

**Via webhook (Zapier / CRM / Form):**
```bash
curl -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer YOUR_PAT" \
  https://api.github.com/repos/markappmedia/markapp-automations/dispatches \
  -d '{
    "event_type": "generate_proposal",
    "client_payload": {
      "doc_type": "proposal",
      "client_name": "Acme Corp",
      "client_email": "contact@acme.com",
      "campaign_budget": "50000",
      "deal_id": "CRM-1234"
    }
  }'
```

**Supported event types:**
- `generate_proposal` — Client proposal PDF
- `generate_contract` — Signed contract (DocuSign-ready)
- `generate_media_plan` — Media plan with inventory breakdown
- `generate_invoice` — Invoice PDF
- `generate_deck_brief` — Campaign deck brief

**Manual:**
1. Go to **Actions → Doc & Contract Generation → Run workflow**
2. Fill in doc type, client name, client email, and optional budget/deal ID

**Output:**
- PDF/DOCX saved to Google Drive (auto-organized by client)
- Email sent to client via SendGrid
- Team notified on Slack with full deal details

---

## Required Scripts

These Python scripts are referenced by the workflows. Add them to `scripts/` in this repo:

| Script | Used by | Purpose |
|---|---|---|
| `scripts/daily_summary.py` | Slack workflow | Pull MarkDash data, format & post summary |
| `scripts/anomaly_check.py` | Slack workflow | Detect anomalies, post alerts |
| `scripts/performance_alert.py` | Slack workflow | Send performance alert for specific campaign |
| `scripts/generate_report.py` | Daily reporting | Fetch data, generate Excel/CSV report |
| `scripts/push_to_sheets.py` | Daily reporting | Push processed data to Google Sheets |
| `scripts/deploy.sh` | CI/CD | Deployment script (SSH, kubectl, or platform CLI) |
| `scripts/generate_doc.py` | Doc generation | Render Jinja2 templates to PDF/DOCX |
| `scripts/upload_to_drive.py` | Doc generation | Upload generated doc to Google Drive |
| `scripts/send_document.py` | Doc generation | Email doc to client via SendGrid |

---

## Stack Alignment

| Product | Workflow | Integration point |
|---|---|---|
| **Pantheon** (SSP) | CI/CD, Reporting | Deploy target, inventory data source |
| **Harion** (DSP) | Reporting, Slack alerts | Campaign performance, spend anomalies |
| **MarkDash** | All workflows | Primary data source + dashboard deep links |

---

*Maintained by markappmedia · Questions? Ping the ops channel on Slack.*—

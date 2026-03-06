# Gmail Auto-Reply Bot (GCP)

A HIPAA-compliant, serverless Gmail auto-reply bot deployed on Google Cloud Platform. It monitors a Gmail inbox for incoming website lead inquiries and sends personalized acknowledgment replies automatically.

## Features

- **Serverless** — runs on Cloud Functions (Gen 2) with zero infrastructure to manage
- **Real-time** — Gmail Pub/Sub push notifications trigger replies within seconds
- **Personalized** — parses lead name from structured form emails for a personal greeting
- **Safe** — daily send limit, deduplication, and sender filtering prevent runaway replies
- **HIPAA-conscious** — no PHI is stored; all processing happens in memory
- **Fully automated** — Gmail watch renewal handled by Cloud Scheduler

## Architecture

```
Website form submission --> Gmail inbox
                              |
                   Gmail Pub/Sub notification
                              |
                     Cloud Function triggered
                              |
              Parse lead name & email from form
                              |
               Send personalized auto-reply
                              |
              Record in Firestore (dedup + daily count)
```

## How It Works

1. A visitor fills out a contact form on the website
2. The CMS emails the form data to the monitored inbox
3. Gmail Pub/Sub detects the new message and triggers the Cloud Function
4. The function parses the lead's name and email from the structured form body
5. A personalized reply is sent and the message is labeled and marked as read
6. Firestore tracks message IDs (deduplication) and a daily send counter

## Project Structure

```
├── cloud_function/           # Deployed Cloud Function
│   ├── main.py               # Entry point (Pub/Sub trigger)
│   ├── config.py             # Settings & reply template
│   ├── gmail_client.py       # Gmail API wrapper
│   ├── email_parser.py       # Parses lead info from form emails
│   ├── reply_builder.py      # Constructs reply messages
│   ├── firestore_client.py   # Dedup & daily limit tracking
│   └── requirements.txt      # Python dependencies
├── setup/
│   ├── setup_pubsub_watch.py # One-time Gmail watch registration
│   ├── renew_watch.py        # Watch renewal (deployed as separate function)
│   └── migrate_tracking.py   # Import history from a previous bot
├── tests/                    # Unit & integration tests
├── deploy.sh                 # GCP deployment script
└── .env.yaml                 # Environment variables (not committed)
```

## Prerequisites

- Python 3.11+
- Google Cloud SDK (`gcloud`)
- A GCP project with billing enabled
- A Google Workspace domain (for domain-wide delegation)

## Setup & Configuration

### Environment Variables

Create a `.env.yaml` file in the project root:

```yaml
IMPERSONATE_USER: "your-monitored-inbox@yourdomain.com"
GCP_PROJECT: "your-gcp-project-id"
DAILY_SEND_LIMIT: "50"
```

### Service Account Key

A service account key file is required at `cloud_function/service_account.json`. This file contains credentials and **must not be committed to version control**.

Generate it after deployment:

```bash
gcloud iam service-accounts keys create cloud_function/service_account.json \
  --iam-account=dental-autoreply@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### Email Settings

Edit `cloud_function/config.py` to customize:

- `SENDER_EMAIL` — the "From" address on form notification emails
- `SUBJECT_LINE` — subject line to match
- `EMAIL_REPLY_TEMPLATE` — the reply body template (supports `{firstname}` placeholder)
- `DAILY_SEND_LIMIT` — max auto-replies per day (default: 50)

Example reply template:

```python
EMAIL_REPLY_TEMPLATE = """Hi {firstname},

Thank you for reaching out! Our team has received your inquiry and will
get back to you soon. You can also reply to this email directly.

All the best,

Your Practice Name
555-555-5555
123 Main St, Suite 100
City, ST 00000"""
```

## Deployment

The `deploy.sh` script provisions all GCP resources:

```bash
chmod +x deploy.sh
./deploy.sh
```

This will:

1. Set the active GCP project
2. Enable required APIs (Cloud Functions, Pub/Sub, Firestore, Gmail, etc.)
3. Create Firestore database
4. Create a service account with least-privilege roles
5. Create the Pub/Sub topic with Gmail push permissions
6. Deploy the main Cloud Function (Pub/Sub triggered)
7. Deploy the watch renewal function (HTTP triggered)
8. Create a Cloud Scheduler job to renew the Gmail watch every 6 days

### Post-Deployment Steps

1. **Domain-wide delegation** — In Google Workspace Admin Console, authorize the service account with the `https://www.googleapis.com/auth/gmail.modify` scope
2. **Download service account key** — See [Service Account Key](#service-account-key) above
3. **Register Gmail watch** — `python setup/setup_pubsub_watch.py`
4. **Migrate history** (optional) — `python setup/migrate_tracking.py`
5. **Send a test email** to verify the bot responds

## GCP Resources

| Resource | Type | Details |
|---|---|---|
| `dental-autoreply` | Cloud Function (Gen 2) | Pub/Sub triggered, 256 MB, 60s timeout |
| `renew-gmail-watch` | Cloud Function (Gen 2) | HTTP triggered, 128 MB, 30s timeout |
| `renew-gmail-watch` | Cloud Scheduler | Runs every 6 days at 3 AM CST |
| `gmail-notifications` | Pub/Sub Topic | Receives Gmail push notifications |
| Firestore | Database | 3 collections: tracking, daily counts, config |

## Testing

```bash
pip install pytest
pytest tests/
```

30+ tests covering email parsing, reply construction, deduplication, daily limits, and sender filtering.

## HIPAA Considerations

- **No PHI stored** — Firestore only holds opaque message IDs and timestamps
- **In-memory processing** — email content is never persisted
- **Encryption** — at rest (AES-256) and in transit (TLS) via GCP defaults
- **Audit trail** — Cloud Logging captures actions without PHI
- **Safety limits** — daily send cap and deduplication prevent runaway sending
- **Least privilege** — service account has only `datastore.user` and `logging.logWriter` roles

> For full HIPAA production compliance, ensure a Business Associate Agreement (BAA) is in place with both Google Cloud and Google Workspace.

## Security Notes

- **Never commit** `service_account.json`, `.env.yaml`, or any credential files
- The `.gitignore` excludes these files
- Service account keys should be rotated periodically
- Domain-wide delegation grants access to the specified Gmail account — scope it narrowly

## License

This project is provided as-is for reference and adaptation. Add a `LICENSE` file to specify terms if publishing publicly.

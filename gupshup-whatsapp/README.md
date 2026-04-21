# gupshup-whatsapp

**Stack:** Gupshup · WhatsApp Business API · Node.js · GCP Cloud Functions · Cloud Run · BigQuery

## Description

Full integration lifecycle for sending and receiving WhatsApp messages via Gupshup as your WABA provider. Covers account setup, Meta verification, template creation and UUID resolution, API send patterns, PubSub architecture, inbound webhook parsing, BigQuery logging, and environment variable management on GCP.

## Key topics covered

- Account setup and Meta Embedded Signup flow
- Template naming, variable rules, and submission
- **Critical:** Template UUID vs template name (most common integration mistake)
- Template registry pattern for PubSub publisher/subscriber architecture
- Node.js `sendGupshupTemplate` — no external dependencies
- Inbound webhook parsing (Gupshup v2 format)
- Session message replies
- BigQuery streaming insert for message logs
- Secret Manager setup for API key
- Common errors and fixes

## How to use

1. Download `gupshup-whatsapp.skill`
2. Go to Claude.ai → Settings → Skills → Upload
3. The skill is now active in your Claude conversations

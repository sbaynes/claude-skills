# Twilio Bulk Export → GCP Cloud Functions

Automate daily Twilio WhatsApp/SMS message log exports into 
Google Cloud Storage and BigQuery using Cloud Functions Gen 2.

## What this solves
Twilio's bulk export signed URLs expire within minutes of the 
notification email being sent — making manual downloads unreliable.
This skill sets up a fully automated pipeline that runs daily 
with no manual steps.

## What's included
- Cloud Function 1: triggers Twilio export job daily
- Cloud Function 2: webhook handler that downloads, decompresses,
  and loads data into BigQuery
- Cloud Scheduler config
- BigQuery schema for Twilio messages
- Full troubleshooting guide for non-obvious Twilio API behaviours

## Stack
Google Cloud Functions (Gen 2) · Cloud Run · Cloud Storage · 
BigQuery · Cloud Scheduler · Twilio Bulk Export API · Node.js 20

## How to use
Upload `twilio-gcp-cloud-functions.skill` to Claude.ai → Settings → Skills

## Tested on
- Twilio standard account (non-Editions)
- WhatsApp Business messaging via Twilio
- GCP Node.js 20 runtime

---
name: gupshup-whatsapp
description: "Use this skill when setting up Gupshup as a WhatsApp Business API provider, creating and submitting templates, wiring templates to Cloud Functions or Cloud Run services, and handling inbound messages via webhook. Covers account setup, Meta verification, template management, UUID resolution, API send patterns, BigQuery logging, and inbound message parsing."
license: MIT
---

# Gupshup WhatsApp Business API Integration

## Overview

Gupshup is a WhatsApp Business API (WABA) partner that lets you send template messages and session messages via REST API. This skill covers the full integration lifecycle for a Node.js / Firebase Cloud Functions stack.

---

## Critical: Template ID vs Template Name

**This is the most common integration mistake and Gupshup's documentation does not make it clear.**

The send API requires the **Gupshup internal UUID** in the `id` field — NOT the template name.

```json
// WRONG — returns status:submitted but message will NOT deliver
{"id": "class_booked_student_v1", "params": [...]}

// CORRECT
{"id": "df628aab-cefb-4d83-898e-590c266f4e41", "params": [...]}
```

The API returns `{"status": "submitted"}` even when the name is used instead of the UUID. The failure is only visible via the webhook callback as Meta error code 4003 `"template did not match"`. This makes the bug invisible at integration time.

**How to find the UUID:**
1. Gupshup dashboard → your app → Templates
2. Click on the template
3. Click the curl / API example button
4. Extract the `id` value — format: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

The template name (e.g. `class_booked_student_v1`) is what shows in the UI. The UUID is what the API requires. They are different values.

---

## Account Setup

### Prerequisites
- Meta Business Manager account (business.facebook.com)
- Meta Business Verification completed (GST number or business registration)
- A phone number NOT currently registered on WhatsApp

### Steps
1. Create account at gupshup.io
2. Create a WhatsApp app — name it clearly (e.g. `MyAppWhatsappAlerts`)
3. Go to your app → WhatsApp → Connect
4. Complete Meta Embedded Signup flow — links your Meta Business Manager
5. Register your phone number via OTP (SMS or voice call)
6. Set business profile: display name, description, category
7. Submit display name for Meta approval — **outbound template messages are blocked until approved**

### Display Name Approval
- Meta approval typically takes 2–24 hours for verified businesses
- Do NOT resubmit or change the name while review is pending — resets the clock
- Session messages (replies within 24hr window) work even while name is pending
- Template messages are silently dropped until the name is approved

### Phone Number Verification
After Meta approves your number, Gupshup sends an email asking you to complete OTP verification. If you cannot find the OTP button in the dashboard, contact support@gupshup.io — they can trigger it manually.

---

## Template Creation

### Naming Convention
Use a consistent naming pattern:
```
{org}_{description}_{recipient}_{version}
# Example: fsg_class_booked_student_v1
```

### Template Categories
- **Utility** — transactional (booking confirmations, cancellations, reminders) — cheapest
- **Marketing** — promotional — more expensive, subject to user opt-out
- **Authentication** — OTP only

Use Utility for all operational alerts.

### Variable Rules
WhatsApp requires variables to be numbered sequentially `{{1}}`, `{{2}}`, `{{3}}` and appear in that order in the body. You cannot have `{{3}}` appear before `{{1}}` in the text.

Variables must be sequential with no gaps — if you declare 5 variables, all of `{{1}}` through `{{5}}` must appear in the body.

### Submitting Templates
1. Dashboard → your app → Templates → Create Template
2. Enter template name (lowercase, underscores, no spaces)
3. Select category: Utility / Marketing / Authentication
4. Write body with `{{1}}`, `{{2}}` etc. for dynamic values
5. Add sample variable values — Meta requires these for approval
6. Submit — approval typically takes minutes to a few hours for Utility

### Sample Variables
Always provide realistic sample values. Meta uses these to evaluate the template:
```
{{1}} = Aryan Sharma        (not "name" or "{{name}}")
{{2}} = Piano               (not "subject")
{{3}} = 14/4/2026           (not "date")
```

### Example Templates

**Class booked — student notification**
```
*Class confirmed.*

Hi *{{1}}*,

Your *{{2}}* class with *{{3}}* has been scheduled for *{{4}}*, *{{5}}* at *{{6}}* for {{7}} minutes in {{8}}.

If you are unable to attend, please cancel in advance.

Ref: {{9}}
```
Variables: `1:studentName 2:subject 3:teacherName 4:dayOfWeek 5:date 6:time 7:duration 8:room 9:scheduleId`

**Class booked — teacher notification**
```
*Class booked.*

Hi *{{1}}*,

You have a class with *{{2}}* for *{{3}}* on *{{4}}*, *{{5}}* at *{{6}}* for {{7}} minutes in {{8}}.

If you are unable to take this session, please cancel at least a day in advance.

Ref: {{9}}
```
Variables: `1:teacherName 2:studentName 3:subject 4:dayOfWeek 5:date 6:time 7:duration 8:room 9:scheduleId`

**Class cancelled — student notification**
```
*Your class has been cancelled.*

Hi *{{1}}*,

Your class with *{{2}}* on *{{4}}*, *{{3}}* at *{{5}}* for {{6}} minutes — *{{7}}* in {{8}} — has been cancelled. Please rebook at the earliest.

Ref: {{9}}
```
Variables: `1:studentName 2:teacherName 3:date 4:dayOfWeek 5:time 6:duration 7:subject 8:room 9:scheduleId`

---

## API: Sending Template Messages

### Endpoint
```
POST https://api.gupshup.io/wa/api/v1/template/msg
Content-Type: application/x-www-form-urlencoded
apikey: YOUR_API_KEY
```

### Required Fields
```
channel=whatsapp
source=91XXXXXXXXXX          (your registered number, digits only, no +)
destination=91XXXXXXXXXX     (recipient number, digits only)
src.name=YourAppName         (exact app name from Gupshup dashboard)
template={"id":"UUID","params":["val1","val2","val3"]}
```

### Node.js Implementation (no external dependencies)
```javascript
const https = require('https');

async function sendGupshupTemplate(to, templateUUID, params) {
  // Normalise to digits only — strip whatsapp: prefix and leading +
  const destination = to.replace('whatsapp:', '').replace(/^\+/, '');

  const payload = new URLSearchParams({
    channel:     'whatsapp',
    source:      process.env.GUPSHUP_FROM_NUMBER,
    destination: destination,
    'src.name':  process.env.GUPSHUP_APP_NAME,
    template:    JSON.stringify({ id: templateUUID, params }),
  }).toString();

  return new Promise((resolve, reject) => {
    const options = {
      hostname: 'api.gupshup.io',
      path:     '/wa/api/v1/template/msg',
      method:   'POST',
      headers:  {
        'Content-Type':   'application/x-www-form-urlencoded',
        'apikey':         process.env.GUPSHUP_API_KEY,
        'Content-Length': Buffer.byteLength(payload),
      },
    };

    const req = https.request(options, (res) => {
      let data = '';
      res.on('data', chunk => { data += chunk; });
      res.on('end', () => {
        try {
          const parsed = JSON.parse(data);
          if (parsed.status === 'submitted') {
            resolve(parsed); // parsed.messageId for delivery tracking
          } else {
            reject(new Error('Gupshup response: ' + data));
          }
        } catch (e) {
          reject(new Error('Parse error: ' + data));
        }
      });
    });

    req.on('error', reject);
    req.setTimeout(8000, () => req.destroy(new Error('Gupshup timeout')));
    req.write(payload);
    req.end();
  });
}
```

### Important: `status: submitted` does NOT mean delivered
Gupshup returns `submitted` when it accepts the request. Actual delivery happens asynchronously and is reported via webhook. A `submitted` response with an incorrect UUID will never deliver — see the UUID section above.

---

## Template Registry Pattern

Maintain a UUID registry in your subscriber function so publishers can use human-readable names:

```javascript
// Maps template name (shown in Gupshup UI) → UUID (required by API)
// To update: open template in Gupshup → copy curl → extract 'id' field
const GUPSHUP_TEMPLATES = {
  'fsg_class_booked_student_v1': {
    uuid:        'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',
    name:        'fsg_class_booked_student_v1',
    description: 'Class booking confirmation to student',
    recipient:   'student',
    trigger:     'onDailyScheduleUpdated — student booked',
    params:      '1:studentName 2:subject 3:teacherName 4:dayOfWeek 5:date 6:time 7:duration 8:room 9:scheduleId',
  },
  'fsg_class_booked_teacher_v1': {
    uuid:        'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',
    name:        'fsg_class_booked_teacher_v1',
    description: 'Class booking notification to teacher',
    recipient:   'teacher',
    trigger:     'onDailyScheduleUpdated — student booked',
    params:      '1:teacherName 2:studentName 3:subject 4:dayOfWeek 5:date 6:time 7:duration 8:room 9:scheduleId',
  },
};

function resolveGupshupTemplateId(templateName) {
  const entry = GUPSHUP_TEMPLATES[templateName];
  if (!entry) {
    console.warn('No UUID mapping for template: ' + templateName
      + ' — sending name directly. Add it to GUPSHUP_TEMPLATES.');
    return templateName; // will fail at Meta — fix the registry
  }
  return entry.uuid;
}
```

Publishers pass `gupshupTemplateName` (the human-readable name) in their PubSub payload. The subscriber resolves it to a UUID before calling the API.

---

## PubSub Architecture

### Publisher (Cloud Function)
```javascript
await topic.publish(Buffer.from(JSON.stringify({
  messageDocumentID:   docId,
  subscriberEmail:     userEmail,   // subscriber resolves phone from email
  gupshupTemplateName: 'fsg_class_booked_student_v1',
  contentVariables:    { 1: studentName, 2: subject, 3: teacherName },
}), 'utf8'));
```

### Subscriber (Cloud Run)
```javascript
const { subscriberEmail, gupshupTemplateName, contentVariables } = payload;

// Resolve phone from your users collection by email
const snap = await db.collection('UsersCollection')
  .where('UserLoginEmail', '==', subscriberEmail)
  .limit(1).get();
const phone = snap.docs[0].data().UserContactPhone;

// Build ordered params array from variables object
const variables = typeof contentVariables === 'string'
  ? JSON.parse(contentVariables) : (contentVariables || {});
const params = Object.keys(variables)
  .sort((a, b) => Number(a) - Number(b))
  .map(k => variables[k]);

// Resolve UUID and send
const uuid = resolveGupshupTemplateId(gupshupTemplateName);
await sendGupshupTemplate(phone, uuid, params);
```

---

## Inbound Messages (Webhook)

### Webhook Setup
1. Gupshup dashboard → your app → Webhooks
2. Set callback URL to your Cloud Run / Cloud Function endpoint
3. Select **Gupshup format (v2)** — NOT Meta format (v3)

### Gupshup v2 Payload Shape
```json
{
  "app":     "YourAppName",
  "type":    "message",
  "payload": {
    "source":  "919999999999",
    "type":    "text",
    "payload": {
      "text": "Hello"
    }
  }
}
```

### Parsing Inbound Messages
```javascript
function parseInbound(req) {
  // Gupshup v2 — JSON with nested payload.source
  if (req.body?.payload?.source) {
    const source = req.body.payload.source;
    const type   = req.body.payload.type;
    let   text   = '';

    if (type === 'text') {
      text = req.body.payload.payload?.text || '';
    } else if (type === 'interactive') {
      // Button / list replies
      text = req.body.payload.payload?.title
          || req.body.payload.payload?.text || '';
    }

    return {
      messageText: text.trim(),
      fromNumber:  'whatsapp:+' + source,
      provider:    'gupshup',
    };
  }

  console.warn('Unknown webhook format — body keys:', Object.keys(req.body || {}));
  return { messageText: '', fromNumber: '', provider: 'unknown' };
}
```

### HTTP Response
Gupshup expects a plain `200 OK` with empty body:
```javascript
res.status(200).send('');
```

### Sending Session Messages (Replies)
Session messages are free-text replies within the 24hr window after a user messages you.

```javascript
async function sendSessionMessage(to, body) {
  const destination = to.replace('whatsapp:', '').replace(/^\+/, '');

  const payload = new URLSearchParams({
    channel:     'whatsapp',
    source:      process.env.GUPSHUP_FROM_NUMBER,
    destination: destination,
    'src.name':  process.env.GUPSHUP_APP_NAME,
    message:     JSON.stringify({ type: 'text', text: body }),
  }).toString();

  return new Promise((resolve, reject) => {
    const options = {
      hostname: 'api.gupshup.io',
      path:     '/wa/api/v1/msg',
      method:   'POST',
      headers:  {
        'Content-Type':   'application/x-www-form-urlencoded',
        'apikey':         process.env.GUPSHUP_API_KEY,
        'Content-Length': Buffer.byteLength(payload),
      },
    };
    const req = https.request(options, (res) => {
      let data = '';
      res.on('data', chunk => { data += chunk; });
      res.on('end', () => resolve(JSON.parse(data)));
    });
    req.on('error', reject);
    req.setTimeout(8000, () => req.destroy(new Error('Gupshup timeout')));
    req.write(payload);
    req.end();
  });
}
```

**Important:** Reply on the same channel the message came in on. If Gupshup sent the inbound, reply via Gupshup. Replying from a different number confuses the user.

---

## Environment Variables

| Variable | Storage | Notes |
|---|---|---|
| `GUPSHUP_API_KEY` | Secret Manager | Sensitive — never store as plaintext |
| `GUPSHUP_APP_NAME` | Plain env var | Exact app name from Gupshup dashboard |
| `GUPSHUP_FROM_NUMBER` | Plain env var | Digits only, no + (e.g. `917975342881`) |

### GCP Secret Manager Setup
```bash
# Create secret
echo -n "YOUR_API_KEY" | gcloud secrets create gupshup-api-key \
  --project=YOUR_PROJECT \
  --replication-policy=automatic \
  --data-file=-

# Grant access to service account
gcloud secrets add-iam-policy-binding gupshup-api-key \
  --project=YOUR_PROJECT \
  --member="serviceAccount:YOUR_SA@appspot.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Mount on Cloud Run service
gcloud run services update YOUR_SERVICE \
  --set-secrets="GUPSHUP_API_KEY=gupshup-api-key:1" \
  --set-env-vars="GUPSHUP_APP_NAME=YourAppName,GUPSHUP_FROM_NUMBER=91XXXXXXXXXX"

# Mount on Cloud Function
gcloud functions deploy YOUR_FUNCTION \
  --set-secrets="GUPSHUP_API_KEY=gupshup-api-key:1" \
  --set-env-vars="GUPSHUP_APP_NAME=YourAppName,GUPSHUP_FROM_NUMBER=91XXXXXXXXXX"
```

---

## BigQuery Logging

Log every send attempt for observability:

```bash
bq mk --table YOUR_PROJECT:gupshup_logs.message_log \
  log_id:STRING,timestamp:TIMESTAMP,direction:STRING,\
  status:STRING,template_name:STRING,template_uuid:STRING,\
  to_phone:STRING,to_email:STRING,from_phone:STRING,\
  message_body:STRING,doc_id:STRING,app_name:STRING,\
  error_message:STRING,elapsed_ms:INTEGER,\
  params_count:INTEGER,gupshup_response:STRING
```

Write via BigQuery streaming insert using the GCP metadata server token:
```javascript
async function logToBigQuery(row) {
  try {
    const tokenRes = await fetch(
      'http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token',
      { headers: { 'Metadata-Flavor': 'Google' } }
    );
    const { access_token } = await tokenRes.json();

    await fetch(
      `https://bigquery.googleapis.com/bigquery/v2/projects/${PROJECT}/datasets/gupshup_logs/tables/message_log/insertAll`,
      {
        method:  'POST',
        headers: {
          'Authorization': 'Bearer ' + access_token,
          'Content-Type':  'application/json',
        },
        body: JSON.stringify({
          rows: [{ insertId: row.log_id, json: row }]
        }),
      }
    );
  } catch (err) {
    // Never let logging failure break message delivery
    console.error('BQ log failed (non-fatal):', err.message);
  }
}
```

Log at minimum two points: send success and send failure.

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `status: submitted` but no delivery | Template name sent instead of UUID | Use UUID from template curl example in dashboard |
| Error 4003 `template did not match` via webhook | Wrong UUID or template not yet approved | Verify approval status; check UUID |
| Regex `/^+/` syntax error | Unescaped `+` in regex | Use `/^\+/` |
| Phone number not resolving | Inconsistent storage format | Normalise on write — store digits only without country code |
| `status: submitted` but delivery delayed | Gupshup→Meta sync lag after display name approval | Wait up to 2 hours after approval |
| Warning log: `No UUID mapping for template` | Template added to publisher but not to registry | Add entry to `GUPSHUP_TEMPLATES` map and redeploy subscriber |

---

## Opt-out / STOP Handling

When a user replies STOP, Meta automatically blocks further **Marketing** template messages from your WABA to that number. Utility templates are not affected.

To build full opt-out compliance:
1. Webhook receives inbound STOP message
2. Write opt-out record to your database
3. Check opt-out before sending any Marketing template
4. Remove from ad retargeting lists if applicable

---

## Go-Live Checklist

- [ ] Meta Business Verification complete
- [ ] Phone number registered and OTP verified in Gupshup
- [ ] Display name approved by Meta
- [ ] All templates approved (Utility category recommended for transactional)
- [ ] UUID registry built — never send template name as `id`
- [ ] Gupshup API key stored in Secret Manager, not plaintext
- [ ] Webhook URL set with format = Gupshup v2
- [ ] BigQuery logging enabled
- [ ] API key rotated if it was ever logged or pasted in plaintext

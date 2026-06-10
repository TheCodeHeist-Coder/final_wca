# WhatsApp Webhooks — Developer Integration Guide

How to receive real-time WhatsApp events (incoming messages, delivery/read status, account updates) in **any** codebase, with working code.

Sources:
- [Webhooks Overview](https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/overview)
- [Create a Webhook Endpoint](https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/create-webhook-endpoint)
- [Set up a WhatsApp Echo Bot](https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/set-up-whatsapp-echo-bot)
- [Override the Webhook](https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/override)
- [Generic Meta Webhooks](https://developers.facebook.com/docs/graph-api/webhooks/getting-started)

---

## 1. Overview — how WhatsApp webhooks work

A **webhook** is an HTTPS endpoint *you* host. Meta sends an HTTP request to it whenever a subscribed event happens (someone messages your number, a message is delivered/read, a flow is completed, etc.). You don't poll — Meta pushes.

```
 WhatsApp user ──▶ Meta (Cloud API) ──▶ POST https://your-server/webhook ──▶ your app
                                    ◀── HTTP 200 OK (within seconds)
```

**Two kinds of requests hit your single endpoint:**

| Method | Purpose | When |
|--------|---------|------|
| `GET /webhook` | **Verification** — proves you own the URL | Once, when you save the callback URL in the App Dashboard |
| `POST /webhook` | **Event notification** — the actual data | Every time a subscribed event fires |

**Hard requirements:**
- **HTTPS** with a valid (non‑self‑signed) TLS certificate.
- Respond **`200 OK`** fast. Do heavy work async — Meta retries failed deliveries for up to **36 hours** (so deduplicate by message `id`).
- Payloads can be up to **3 MB**; a single notification's `entry` array may batch many events.

---

## 2. Subscribable webhook fields

In the App Dashboard → WhatsApp → Configuration → Webhook fields, subscribe to what you need. The most common:

| Field | Fires when |
|-------|-----------|
| `messages` | Incoming messages **and** message status updates (sent/delivered/read/failed) — this is the main one |
| `message_template_status_update` | A message template is approved/rejected/paused |
| `flows` | Flow status changes & endpoint health alerts |
| `account_update`, `phone_number_name_update`, `phone_number_quality_update` | Account/number administrative changes |
| `account_review_update`, `business_capability_update` | Review & capability changes |

> For most chatbots you only need **`messages`**.

---

## 3. The notification payload structure

All WhatsApp notifications share this envelope (`object` is always `whatsapp_business_account`):

```json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "<WABA_ID>",
      "changes": [
        {
          "field": "messages",
          "value": {
            "messaging_product": "whatsapp",
            "metadata": {
              "display_phone_number": "15551234567",
              "phone_number_id": "<PHONE_NUMBER_ID>"
            }
            /* + contacts / messages / statuses (see below) */
          }
        }
      ]
    }
  ]
}
```

Navigation: `entry[]` → `changes[]` → `value`. The shape of `value` depends on the event.

### 3.1 Incoming text message
```json
"value": {
  "messaging_product": "whatsapp",
  "metadata": { "display_phone_number": "15551234567", "phone_number_id": "106540352242922" },
  "contacts": [
    { "profile": { "name": "Raj" }, "wa_id": "919876543210" }
  ],
  "messages": [
    {
      "from": "919876543210",
      "id": "wamid.HBgL...==",
      "timestamp": "1718000000",
      "type": "text",
      "text": { "body": "Hello!" }
    }
  ]
}
```

### 3.2 Other incoming message types (the `messages[].type` switch)

| `type` | Key payload |
|--------|-------------|
| `text` | `text.body` |
| `image` / `video` / `audio` / `document` / `sticker` | `<type>.id` (media id to download), `mime_type`, `sha256`, optional `caption` |
| `location` | `location.latitude`, `longitude`, `name`, `address` |
| `contacts` | `contacts[]` |
| `button` | `button.text`, `button.payload` (template quick-reply) |
| `interactive` | `interactive.type` = `button_reply` (`id`,`title`) or `list_reply` (`id`,`title`,`description`) |
| `interactive` → `nfm_reply` | Flow completion — `nfm_reply.response_json` |
| `reaction` | `reaction.message_id`, `reaction.emoji` |
| `order` | `order.catalog_id`, `order.product_items[]` |

A `context` object appears when the message is a reply/reaction: `context.from`, `context.id`.

### 3.3 Status update (sent / delivered / read / failed)
```json
"value": {
  "messaging_product": "whatsapp",
  "metadata": { "display_phone_number": "15551234567", "phone_number_id": "106540352242922" },
  "statuses": [
    {
      "id": "wamid.HBgL...==",
      "status": "delivered",
      "timestamp": "1718000005",
      "recipient_id": "919876543210",
      "conversation": { "id": "...", "origin": { "type": "marketing" } },
      "pricing": { "billable": true, "pricing_model": "CBP", "category": "marketing" }
    }
  ]
}
```
`status` ∈ `sent | delivered | read | failed`. On `failed`, an `errors[]` array (`code`, `title`, `message`, `error_data`) is included.

---

## 4. Creating the webhook endpoint

### 4.1 GET — verification handshake
When you save the callback URL, Meta sends:
```
GET /webhook?hub.mode=subscribe&hub.challenge=1158201444&hub.verify_token=YOUR_TOKEN
```
You must check `hub.verify_token` against the token you typed in the dashboard, then **echo back `hub.challenge` as plain text with status 200**. Otherwise return `403`.

### 4.2 POST — receive events
Return `200` immediately; parse `entry[].changes[].value` for `messages` / `statuses`.

### 4.3 Configure in the App Dashboard
1. App Dashboard → **WhatsApp → Configuration**.
2. **Callback URL**: `https://your-domain/webhook`.
3. **Verify token**: any string you choose (must match your code's `VERIFY_TOKEN`).
4. Click **Verify and save** (triggers the GET handshake).
5. Under **Webhook fields**, click **Manage** → subscribe to `messages`.

---

## 5. Security — validate the payload signature

Every `POST` includes header `X-Hub-Signature-256: sha256=<hmac>`. Compute HMAC‑SHA256 of the **raw request body** with your **App Secret** and compare. Reject mismatches — this stops spoofed requests.

```js
const crypto = require('crypto');

function verifySignature(rawBody, header, appSecret) {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', appSecret)
    .update(rawBody)            // RAW bytes, before JSON.parse
    .digest('hex');
  // constant-time compare
  return crypto.timingSafeEqual(Buffer.from(header || ''), Buffer.from(expected));
}
```

> To get the raw body in Express, capture it in the JSON parser:
> ```js
> app.use(express.json({ verify: (req, _res, buf) => { req.rawBody = buf; } }));
> ```

---

## 6. Code demos

### 6.1 Node.js / Express — verification + receiver + signature check
```js
require('dotenv').config();
const express = require('express');
const crypto = require('crypto');

const app = express();
// keep the raw body so we can validate X-Hub-Signature-256
app.use(express.json({ verify: (req, _res, buf) => { req.rawBody = buf; } }));

const VERIFY_TOKEN = process.env.VERIFY_TOKEN;
const APP_SECRET   = process.env.APP_SECRET;

// 1) Verification handshake
app.get('/webhook', (req, res) => {
  const mode      = req.query['hub.mode'];
  const token     = req.query['hub.verify_token'];
  const challenge = req.query['hub.challenge'];

  if (mode === 'subscribe' && token === VERIFY_TOKEN) {
    return res.status(200).send(challenge);   // echo challenge as plain text
  }
  return res.sendStatus(403);
});

// 2) Event notifications
app.post('/webhook', (req, res) => {
  // validate signature (optional but recommended)
  const sig = req.get('X-Hub-Signature-256');
  const expected = 'sha256=' + crypto.createHmac('sha256', APP_SECRET)
    .update(req.rawBody).digest('hex');
  if (!sig || sig !== expected) return res.sendStatus(401);

  // ALWAYS ack quickly
  res.sendStatus(200);

  const value = req.body?.entry?.[0]?.changes?.[0]?.value;
  if (!value) return;

  if (value.messages) {
    for (const msg of value.messages) {
      console.log(`Message from ${msg.from} (${msg.type}):`,
        msg.type === 'text' ? msg.text.body : msg);
      // ... your business logic (queue it, don't block the response)
    }
  }
  if (value.statuses) {
    for (const st of value.statuses) {
      console.log(`Message ${st.id} is now ${st.status}`);
    }
  }
});

app.listen(process.env.PORT || 4000, () => console.log('Webhook listening'));
```

`.env`:
```
VERIFY_TOKEN=my_verify_token
APP_SECRET=your_app_secret_from_dashboard
GRAPH_TOKEN=your_cloud_api_access_token
```

### 6.2 Echo bot — reply to whatever the user sends

Receives a text message and sends the same text back (plus marks it read + shows a typing indicator).

```js
const axios = require('axios');
const GRAPH = 'https://graph.facebook.com/v22.0';
const TOKEN = process.env.GRAPH_TOKEN;

async function callGraph(phoneNumberId, data) {
  return axios.post(`${GRAPH}/${phoneNumberId}/messages`, data, {
    headers: { Authorization: `Bearer ${TOKEN}`, 'Content-Type': 'application/json' },
  });
}

app.post('/webhook', async (req, res) => {
  res.sendStatus(200);   // ack first

  const value = req.body?.entry?.[0]?.changes?.[0]?.value;
  const msg = value?.messages?.[0];
  if (!msg || msg.type !== 'text') return;

  const phoneNumberId = value.metadata.phone_number_id;

  // (a) mark the incoming message as read + show typing indicator
  await callGraph(phoneNumberId, {
    messaging_product: 'whatsapp',
    status: 'read',
    message_id: msg.id,
    typing_indicator: { type: 'text' },
  });

  // (b) echo the text back
  await callGraph(phoneNumberId, {
    messaging_product: 'whatsapp',
    recipient_type: 'individual',
    to: msg.from,
    type: 'text',
    text: { preview_url: false, body: `You said: ${msg.text.body}` },
  });
});
```

> **Why mark-as-read matters:** the `to` number can only be messaged freely inside the 24‑hour customer service window opened by their inbound message. Outside it, you must use an approved template.

### 6.3 Python / Flask — equivalent
```python
import os, hmac, hashlib
from flask import Flask, request, abort

app = Flask(__name__)
VERIFY_TOKEN = os.environ["VERIFY_TOKEN"]
APP_SECRET   = os.environ["APP_SECRET"].encode()

@app.get("/webhook")
def verify():
    if (request.args.get("hub.mode") == "subscribe"
            and request.args.get("hub.verify_token") == VERIFY_TOKEN):
        return request.args.get("hub.challenge"), 200
    return "Forbidden", 403

@app.post("/webhook")
def receive():
    sig = request.headers.get("X-Hub-Signature-256", "")
    expected = "sha256=" + hmac.new(APP_SECRET, request.data, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(sig, expected):
        abort(401)

    value = (request.json.get("entry", [{}])[0]
                          .get("changes", [{}])[0]
                          .get("value", {}))
    for m in value.get("messages", []):
        print("from", m["from"], m.get("text", {}).get("body"))
    return "", 200
```

### 6.4 PHP — verification snippet
```php
<?php
// GET verification
if ($_SERVER['REQUEST_METHOD'] === 'GET') {
  if ($_GET['hub_mode'] === 'subscribe'
      && $_GET['hub_verify_token'] === getenv('VERIFY_TOKEN')) {
    http_response_code(200);
    echo $_GET['hub_challenge'];           // echo challenge
  } else { http_response_code(403); }
  exit;
}
// POST notifications
$body = file_get_contents('php://input');
$sig  = 'sha256=' . hash_hmac('sha256', $body, getenv('APP_SECRET'));
if (!hash_equals($sig, $_SERVER['HTTP_X_HUB_SIGNATURE_256'] ?? '')) {
  http_response_code(401); exit;
}
http_response_code(200);
$data = json_decode($body, true);
// ... handle $data['entry'][0]['changes'][0]['value']
```

---

## 7. Overriding the webhook (per‑phone‑number)

By default all phone numbers in a WABA send events to the WABA‑level callback URL configured in the App Dashboard. **Webhook override** lets you route a *specific phone number's* events to a *different* URL — useful for multi‑tenant platforms, partner solutions (BSPs), or staged migration.

### 7.1 Set an override for one phone number
`POST {BASE-URL}/{PHONE-NUMBER-ID}/subscribed_apps`

| Param | Notes |
|-------|-------|
| `override_callback_uri` | The HTTPS URL that should receive **this number's** webhooks |
| `verify_token` | Token used for that override URL's GET handshake |

```bash
curl -X POST 'https://graph.facebook.com/v22.0/{PHONE-NUMBER-ID}/subscribed_apps' \
  -H 'Authorization: Bearer {ACCESS-TOKEN}' \
  -H 'Content-Type: application/json' \
  -d '{
        "override_callback_uri": "https://tenant-a.example.com/webhook",
        "verify_token": "tenant_a_token"
      }'
```
```json
{ "success": true }
```
Meta immediately sends the GET verification handshake to `override_callback_uri` (same `hub.mode`/`hub.challenge`/`hub.verify_token` flow as §4.1) — your override URL must echo the challenge.

### 7.2 Inspect current subscriptions / overrides
`GET {BASE-URL}/{PHONE-NUMBER-ID}/subscribed_apps`

Returns the subscribed app(s) and any `override_callback_uri` in effect.

### 7.3 Remove the override
`DELETE {BASE-URL}/{PHONE-NUMBER-ID}/subscribed_apps`

Removes the app subscription / override; the number falls back to the WABA‑level webhook.

**Precedence:** phone‑number override URL → else WABA‑level App Dashboard URL.

---

## 8. Testing & troubleshooting

- **Local dev:** expose your localhost with `ngrok http 4000` and use the `https://*.ngrok.io/webhook` URL as the callback.
- **Test button:** App Dashboard → Webhooks → **Test** sends a sample payload per field.
- **Verification fails (403):** `hub.verify_token` in code ≠ dashboard token, or you're not echoing `hub.challenge` as plain text.
- **No events arrive:** subscribe to the `messages` field; confirm the number is registered; check the WABA is subscribed to your app.
- **Duplicate events:** Meta retries on non‑200 — always return 200 and deduplicate on `messages[].id` / `statuses[].id`.
- **401 on your side:** signature mismatch — make sure you HMAC the **raw** body, not the re‑serialized JSON.

---

## 9. Notes for *this* project

This repo's [index.js](index.js) already implements `GET /webhook` (verification) and `POST /webhook` (receiver). To harden it against this guide:

1. The POST handler reads `...value.message[0]` / `messages[0].text.body` — guard for non‑text types (§3.2) before accessing `.text.body`.
2. Add **signature validation** (§5) using your App Secret.
3. Return `200` **before** doing the outbound `axios` send so slow Graph calls can't cause retries.
4. The outbound payload is double‑nested (`data: { data: {...} }`) — the Graph body should be the message object directly (see §6.2).
5. Align the Graph API version (`v25.0` in index.js vs `v22.0` here) across the codebase.

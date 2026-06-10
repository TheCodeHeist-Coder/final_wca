# WhatsApp Flows — Developer Implementation Guide

A complete reference of the WhatsApp Flows endpoints for implementing Flows in this project.
Source: [Meta — WhatsApp Flows documentation](https://developers.facebook.com/documentation/business-messaging/whatsapp/flows)

WhatsApp Flows let you build structured, interactive form-like experiences (multi-screen questionnaires, appointment booking, sign-ups, etc.) inside WhatsApp. There are two integration styles:

- **Static flows** — all screens/data defined in *Flow JSON*; no backend needed.
- **Dynamic flows (with endpoint)** — your server (a "Flow Endpoint") receives encrypted requests and returns the next screen's data at runtime.

---

## 0. Conventions

| Token | Meaning |
|-------|---------|
| `BASE-URL` | `https://graph.facebook.com/v22.0` (use a recent Graph API version) |
| `WABA-ID` | WhatsApp Business Account ID |
| `FLOW-ID` | The Flow's ID returned by *Create Flow* |
| `PHONE-NUMBER-ID` | Phone number ID used to send messages / set encryption keys |
| `ACCESS-TOKEN` | A System User or app access token with `whatsapp_business_management` + `whatsapp_business_messaging` |

**Auth (all requests):**
```
Authorization: Bearer {ACCESS-TOKEN}
Content-Type: application/json     # except asset upload (multipart/form-data)
```

---

## 1. Flows Management API

These endpoints create and manage the lifecycle of a Flow. Lifecycle: `DRAFT → PUBLISHED → DEPRECATED` (or `THROTTLED` / `BLOCKED`).

### 1.1 Create a Flow
`POST {BASE-URL}/{WABA-ID}/flows`

| Param | Type | Req | Notes |
|-------|------|-----|-------|
| `name` | string | ✅ | Flow name |
| `categories` | array | ✅ | One+ of: `SIGN_UP`, `SIGN_IN`, `APPOINTMENT_BOOKING`, `LEAD_GENERATION`, `CONTACT_US`, `CUSTOMER_SUPPORT`, `SURVEY`, `OTHER` |
| `flow_json` | string | — | JSON-encoded Flow definition (create with content in one call) |
| `publish` | boolean | — | Publish immediately on creation |
| `clone_flow_id` | string | — | Clone an existing flow |
| `endpoint_uri` | string | — | Flow Endpoint URL (replaces deprecated `data_channel_uri`) |

```bash
curl -X POST '{BASE-URL}/{WABA-ID}/flows' \
  -H 'Authorization: Bearer {ACCESS-TOKEN}' \
  -H 'Content-Type: application/json' \
  -d '{"name":"Appointment Booking","categories":["APPOINTMENT_BOOKING"]}'
```
```json
{ "id": "<FLOW-ID>", "success": true, "validation_errors": [] }
```

### 1.2 Update Flow Metadata
`POST {BASE-URL}/{FLOW-ID}`

Optional: `name`, `categories`, `endpoint_uri`, `application_id` (Meta app connected to the endpoint).
```json
{ "success": true }
```

### 1.3 Upload / Update Flow JSON (Assets)
`POST {BASE-URL}/{FLOW-ID}/assets` — `multipart/form-data`

| Field | Value |
|-------|-------|
| `file` | The Flow JSON file (max **10 MB**) |
| `name` | `"flow.json"` |
| `asset_type` | `"FLOW_JSON"` |

```bash
curl -X POST '{BASE-URL}/{FLOW-ID}/assets' \
  -H 'Authorization: Bearer {ACCESS-TOKEN}' \
  -F 'file=@flow.json;type=application/json' \
  -F 'name=flow.json' \
  -F 'asset_type=FLOW_JSON'
```
```json
{ "success": true, "validation_errors": [] }
```

### 1.4 List Flows
`GET {BASE-URL}/{WABA-ID}/flows`
```json
{
  "data": [
    { "id": "flow-1", "name": "flow 1", "status": "DRAFT",
      "categories": ["CONTACT_US"], "validation_errors": [] }
  ],
  "paging": { "cursors": { "before": "...", "after": "..." } }
}
```

### 1.5 Get Flow Details
`GET {BASE-URL}/{FLOW-ID}?fields=id,name,categories,preview,status,validation_errors,json_version,data_api_version,endpoint_uri,whatsapp_business_account,application,health_status`

Add per-phone health: `...&fields=health_status.phone_number({PHONE-NUMBER-ID})`. `health_status` contains `can_send_message`.

### 1.6 Get Flow Assets
`GET {BASE-URL}/{FLOW-ID}/assets`
```json
{ "data": [ { "name": "flow.json", "asset_type": "FLOW_JSON",
              "download_url": "https://scontent.xx.fbcdn.net/..." } ] }
```

### 1.7 Web Preview
`GET {BASE-URL}/{FLOW-ID}?fields=preview.invalidate(false)`

Optional query params for an interactive preview: `interactive`, `flow_token`, `flow_action` (`navigate`|`data_exchange`), `flow_action_payload` (URI-encoded JSON), `phone_number`, `debug`.
```json
{ "preview": { "preview_url": "https://business.facebook.com/wa/manage/flows/...",
               "expires_at": "2023-05-21T11:18:09+0000" }, "id": "flow-1" }
```

### 1.8 Publish a Flow
`POST {BASE-URL}/{FLOW-ID}/publish`

Preconditions: verified business, **zero** validation errors, complies with WhatsApp ToS/policies. After publishing, Flow JSON becomes immutable (create a new flow to change it).
```json
{ "success": true }
```

### 1.9 Deprecate a Flow
`POST {BASE-URL}/{FLOW-ID}/deprecate`

Use this to retire a **published** flow (you cannot delete published flows).
```json
{ "success": true }
```

### 1.10 Delete a Flow
`DELETE {BASE-URL}/{FLOW-ID}` — only allowed while `status == DRAFT`.
```json
{ "success": true }
```

### 1.11 Migrate Flows (between WABAs)
`POST {BASE-URL}/{DEST-WABA-ID}/migrate_flows?source_waba_id={SRC-WABA-ID}&source_flow_names={NAMES}`

Both WABAs must belong to the same Meta business. Flows with name collisions are skipped. Omit `source_flow_names` to migrate all.

---

## 2. Sending a Flow Message

### 2.1 Interactive Flow message (user-initiated session)
`POST {BASE-URL}/{PHONE-NUMBER-ID}/messages`
```json
{
  "recipient_type": "individual",
  "messaging_product": "whatsapp",
  "to": "{RECIPIENT-WA-ID}",
  "type": "interactive",
  "interactive": {
    "type": "flow",
    "header": { "type": "text", "text": "Flow header" },
    "body":   { "text": "Flow body" },
    "footer": { "text": "Flow footer" },
    "action": {
      "name": "flow",
      "parameters": {
        "flow_message_version": "3",
        "flow_id": "{FLOW-ID}",
        "flow_cta": "Book!",
        "flow_token": "AQAAAAACS5FpgQ_cAAAAAD0QI3s",
        "flow_action": "navigate",
        "flow_action_payload": {
          "screen": "SCREEN_NAME",
          "data": { "product_name": "name", "product_price": 100 }
        },
        "mode": "published"
      }
    }
  }
}
```

| Parameter | Notes |
|-----------|-------|
| `flow_message_version` | `"3"` (required) |
| `flow_id` *or* `flow_name` | Pick one |
| `flow_cta` | Button label (≤30 chars) |
| `flow_token` | Your session/correlation id (default `"unused"`) — echoed back in webhooks & endpoint requests |
| `flow_action` | `navigate` (open at a screen) or `data_exchange` (call your endpoint for the first screen) |
| `flow_action_payload` | `{ screen, data }` — required when `flow_action=navigate` |
| `mode` | `published` (default) or `draft` (for testing) |

### 2.2 Flow via a Message Template (business-initiated)

**Create template** with a FLOW button:
```json
{
  "name": "example_template_name", "language": "en_US", "category": "MARKETING",
  "components": [
    { "type": "body", "text": "This is a flow-as-template demo" },
    { "type": "BUTTONS",
      "buttons": [ { "type": "FLOW", "text": "Open flow!", "flow_id": "{FLOW-ID}" } ] }
  ]
}
```

**Send template** (must be `APPROVED`):
```json
{
  "messaging_product": "whatsapp", "recipient_type": "individual",
  "to": "{PHONE-NUMBER}", "type": "template",
  "template": {
    "name": "{TEMPLATE-NAME}", "language": { "code": "{LANGUAGE-CODE}" },
    "components": [
      { "type": "button", "sub_type": "flow", "index": "0",
        "parameters": [
          { "type": "action",
            "action": { "flow_token": "{FLOW-TOKEN}", "flow_action_data": {} } }
        ] }
    ]
  }
}
```

---

## 3. Flow JSON (screen definition)

Top-level object:
```json
{
  "version": "7.0",
  "data_api_version": "3.0",
  "routing_model": { "SCREEN_ONE": ["SCREEN_TWO"], "SCREEN_TWO": [] },
  "screens": [ /* screen objects */ ]
}
```

| Field | Notes |
|-------|-------|
| `version` | Flow JSON version |
| `data_api_version` | Required only for endpoint-backed (dynamic) flows; `"3.0"` |
| `routing_model` | Allowed screen→screen transitions (dynamic flows) |
| `screens` | Array of screen objects |

**Screen object:**
```json
{
  "id": "SCREEN_NAME",
  "title": "Title",
  "terminal": true,
  "success": true,
  "data": {},
  "layout": {
    "type": "SingleColumnLayout",
    "children": [ /* components */ ]
  }
}
```
- `terminal: true` marks the last screen. `SUCCESS` is a reserved terminal screen name.

**Action types** (used in components' `on-click-action` / `on-select-action`):

| `name` | Purpose |
|--------|---------|
| `navigate` | Go to another screen (static) |
| `data_exchange` | Call your Flow Endpoint, server returns next screen |
| `update_data` | Update screen data client-side |
| `complete` | Finish the flow and send the response back to the business |

```json
"on-click-action": {
  "name": "data_exchange",
  "payload": { "field": "${form.field}" }
}
```

**Common component types:** `TextHeading`, `TextSubheading`, `TextBody`, `TextCaption`,
`TextInput`, `TextArea`, `CheckboxGroup`, `RadioButtonsGroup`, `Dropdown`, `DatePicker`,
`CalendarPicker`, `OptIn`, `Image`, `PhotoPicker`, `DocumentPicker` (media upload), `Footer`, `EmbeddedLink`.

---

## 4. Flow Endpoint — Data Exchange Contract (dynamic flows)

Set the Flow's `endpoint_uri` (§1.1/§1.2). Meta sends **encrypted** POSTs to that URL. You decrypt, process, and return an **encrypted** response.

### 4.1 Decrypted request payload
```json
{
  "version": "3.0",
  "action": "INIT | BACK | data_exchange | ping",
  "screen": "SCREEN_NAME",
  "data": { "prop_1": "value_1" },
  "flow_token": "TOKEN_VALUE",
  "flow_token_signature": "JWT_TOKEN"
}
```

| Field | Notes |
|-------|-------|
| `version` | Data API version, `"3.0"` |
| `action` | See table below |
| `screen` | Current screen (absent for `INIT`/`ping`) |
| `data` | Submitted form values (absent for `INIT`/`BACK`) |
| `flow_token` | The token you set when sending the flow |
| `flow_token_signature` | JWT for token authenticity (Flows ≥ v7.3, data API ≥ v4.0) |

| `action` | When |
|----------|------|
| `INIT` | Flow opened with `flow_action=data_exchange` — return the first screen |
| `BACK` | Back pressed and screen has `refresh_on_back: true` |
| `data_exchange` | A screen was submitted — process & return next screen |
| `ping` | Health check — confirm endpoint is alive |

### 4.2 Responses (encrypt before returning)

**Next screen:**
```json
{ "screen": "NEXT_SCREEN_NAME",
  "data": { "property_1": "value_1", "error_message": "optional" } }
```

**Completion:**
```json
{ "screen": "SUCCESS",
  "data": { "extension_message_response": {
    "params": { "flow_token": "TOKEN_VALUE", "optional_param1": "value1" } } } }
```

**Health check (`ping`):**
```json
{ "data": { "status": "active" } }
```

**Error acknowledgement** (Meta notifies you of a client error):
```json
{ "data": { "acknowledged": true } }
```
Incoming error notification looks like:
```json
{ "version": "3.0", "flow_token": "...", "action": "data_exchange|INIT",
  "data": { "error": "ERROR_KEY", "error_message": "Description" } }
```

### 4.3 Status codes
- Return `200` with an encrypted body on success.
- Return `421 Misdirected Request` if you cannot decrypt the request (signals a key mismatch so WhatsApp refreshes the public key).

---

## 5. Encryption (required for endpoint flows)

### 5.1 Upload your business public key
`POST {BASE-URL}/{PHONE-NUMBER-ID}/whatsapp_business_encryption`

Body param: `business_public_key` (a 2048-bit RSA public key in PEM).
```bash
curl -X POST '{BASE-URL}/{PHONE-NUMBER-ID}/whatsapp_business_encryption' \
  -H 'Authorization: Bearer {ACCESS-TOKEN}' \
  --data-urlencode 'business_public_key=-----BEGIN PUBLIC KEY-----...'
```

### 5.2 Get / verify the key
`GET {BASE-URL}/{PHONE-NUMBER-ID}/whatsapp_business_encryption`
```json
{ "data": [ { "business_public_key": "...",
              "business_public_key_signature_status": "VALID|MISMATCH" } ] }
```

### 5.3 Scheme (per request)

The encrypted request body contains three base64 fields:

| Field | Meaning |
|-------|---------|
| `encrypted_flow_data` | The flow payload, AES-128-**GCM** encrypted (128-bit auth tag appended) |
| `encrypted_aes_key` | The AES key, RSA-encrypted with **RSA-OAEP / SHA-256** using your public key |
| `initial_vector` | The IV/nonce for AES-GCM |

**Decrypt (your endpoint):**
1. RSA-decrypt `encrypted_aes_key` with your **private** key → 16-byte AES key.
2. AES-128-GCM decrypt `encrypted_flow_data` using the AES key + `initial_vector` (the last 16 bytes are the GCM tag) → the JSON request from §4.1.

**Encrypt the response:**
1. Reuse the same AES key.
2. **Flip every bit (invert) of the `initial_vector`** and use that as the IV for the response.
3. AES-128-GCM encrypt your JSON response and return it **base64-encoded as the raw HTTP body** (`Content-Type: text/plain`).

> Validate the request signature using the SHA-256 HMAC with your **app secret** (and the `flow_token_signature` JWT where available) before trusting any payload.

---

## 6. Webhooks

Subscribe the WABA to the `flows` and `messages` fields.

### 6.1 Flow completion (user finished the flow)
Arrives on the **messages** webhook as an `interactive` / `nfm_reply`:
```json
{
  "messages": [{
    "from": "<USER-WA-NUMBER>", "id": "<MESSAGE-ID>", "type": "interactive",
    "interactive": {
      "type": "nfm_reply",
      "nfm_reply": {
        "name": "flow",
        "body": "Sent",
        "response_json": "{\"flow_token\":\"<TOKEN>\", ...}"
      }
    },
    "timestamp": "<TIMESTAMP>"
  }]
}
```
Parse `response_json` (a JSON string) for the data the user submitted; structure is defined by your Flow JSON or returned by your endpoint.

### 6.2 Flow status change (`field: "flows"`)
```json
{
  "object": "whatsapp_business_account",
  "entry": [{ "id": "<WABA-ID>", "changes": [{
    "field": "flows",
    "value": {
      "event": "FLOW_STATUS_CHANGE",
      "message": "Flow changed status from DRAFT to PUBLISHED",
      "flow_id": "<FLOW-ID>",
      "old_status": "DRAFT", "new_status": "PUBLISHED"
    }
  }]}]
}
```
Statuses: `DRAFT`, `PUBLISHED`, `DEPRECATED`, `BLOCKED`, `THROTTLED`.

### 6.3 Endpoint health alerts (`field: "flows"`)
Each carries `error_rate`/`threshold`/`alert_state` (`ACTIVATED`/`DEACTIVATED`):

| Event | Thresholds | Window |
|-------|-----------|--------|
| `CLIENT_ERROR_RATE` | 5% / 10% / 50% | 60 min |
| `ENDPOINT_ERROR_RATE` | 5% / 10% / 50% | 30 min |
| `ENDPOINT_LATENCY` | P90 1s / 5s / 7s | — |
| `ENDPOINT_AVAILABILITY` | < 90% | — |

---

## 7. Suggested implementation order for this project

This repo currently has a basic webhook in [index.js](index.js). To add Flows:

1. **Generate RSA keypair**, upload public key (§5.1), keep the private key in `.env`.
2. **Author Flow JSON** (§3); **Create Flow** (§1.1) and **upload assets** (§1.3).
3. **Publish** (§1.8) once validation passes.
4. **Send** the flow via interactive message or template (§2).
5. Add a **`POST /flow-endpoint`** route that: decrypts (§5.3), routes on `action` (§4.1), returns encrypted next-screen JSON (§4.2).
6. Handle the **`nfm_reply`** completion on your existing `/webhook` (§6.1) to capture submitted data.
7. Monitor via **health webhooks** (§6.3) and `health_status` (§1.5).

### Minimal endpoint skeleton (Express, Node `crypto`)
```js
const crypto = require('crypto');

function decryptRequest(body, privatePem) {
  const { encrypted_flow_data, encrypted_aes_key, initial_vector } = body;
  const aesKey = crypto.privateDecrypt(
    { key: privatePem, padding: crypto.constants.RSA_PKCS1_OAEP_PADDING, oaepHash: 'sha256' },
    Buffer.from(encrypted_aes_key, 'base64')
  );
  const flowData = Buffer.from(encrypted_flow_data, 'base64');
  const iv = Buffer.from(initial_vector, 'base64');
  const tag = flowData.subarray(-16);
  const body_ct = flowData.subarray(0, -16);
  const decipher = crypto.createDecipheriv('aes-128-gcm', aesKey, iv);
  decipher.setAuthTag(tag);
  const json = Buffer.concat([decipher.update(body_ct), decipher.final()]);
  return { decrypted: JSON.parse(json.toString('utf-8')), aesKey, iv };
}

function encryptResponse(responseObj, aesKey, iv) {
  const flippedIv = Buffer.from(iv.map(b => b ^ 0xff));   // invert all bits
  const cipher = crypto.createCipheriv('aes-128-gcm', aesKey, flippedIv);
  const ct = Buffer.concat([cipher.update(JSON.stringify(responseObj), 'utf-8'), cipher.final()]);
  return Buffer.concat([ct, cipher.getAuthTag()]).toString('base64');
}

app.post('/flow-endpoint', (req, res) => {
  const { decrypted, aesKey, iv } = decryptRequest(req.body, process.env.PRIVATE_KEY);
  let response;
  switch (decrypted.action) {
    case 'ping':          response = { data: { status: 'active' } }; break;
    case 'INIT':          response = { screen: 'FIRST_SCREEN', data: {} }; break;
    case 'data_exchange': response = { screen: 'NEXT_SCREEN', data: {} }; break;
    default:              response = { data: { acknowledged: true } };
  }
  res.send(encryptResponse(response, aesKey, iv));   // base64, text/plain
});
```
> Note: encryption requires the **raw request body**. Use `express.json()` for `/flow-endpoint` so the three base64 fields parse correctly, and validate the `x-hub-signature-256` header against your app secret.

---

## 8. Reference links
- [Flows overview](https://developers.facebook.com/documentation/business-messaging/whatsapp/flows)
- [Flows API reference](https://developers.facebook.com/docs/whatsapp/flows/reference/flowsapi/)
- [Implementing the Flow Endpoint](https://developers.facebook.com/docs/whatsapp/flows/guides/implementingyourflowendpoint/)
- [Flow JSON](https://developers.facebook.com/docs/whatsapp/flows/reference/flowjson/)
- [Reference index](https://developers.facebook.com/docs/whatsapp/flows/reference/)
- [Changelog](https://developers.facebook.com/documentation/business-messaging/whatsapp/changelog)

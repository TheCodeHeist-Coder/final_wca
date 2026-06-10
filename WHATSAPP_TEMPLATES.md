# WhatsApp Message Templates — Developer Implementation Guide

Everything needed to **create, send, and manage** WhatsApp message templates with the Cloud API, plus categorization, languages, and messaging limits.

Sources (Meta — Templates docs):
- [Overview](https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/overview)
- [Components](https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/components)
- [Supported Languages](https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/supported-languages)
- [Template Categorization](https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/template-categorization)
- [Template Comparison](https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/template-comparison)
- [Template Library](https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/template-library)
- [Messaging Limits](https://developers.facebook.com/documentation/business-messaging/whatsapp/messaging-limits)

---

## 0. Conventions

| Token | Meaning |
|-------|---------|
| `BASE-URL` | `https://graph.facebook.com/v22.0` |
| `WABA-ID` | WhatsApp Business Account ID |
| `PHONE-NUMBER-ID` | Phone number ID used to send messages |
| `ACCESS-TOKEN` | Token with `whatsapp_business_management` + `whatsapp_business_messaging` |

**Auth (all requests):** `Authorization: Bearer {ACCESS-TOKEN}`, `Content-Type: application/json`.

---

## 1. What templates are & why

Outside the **24‑hour customer service window** (opened when a user messages you), you can **only** send a pre‑approved **message template**. Templates are reusable, Meta‑reviewed messages with placeholders you fill at send time. Free‑form text/media only works inside the open window.

Two‑step model:
1. **Create** a template under your WABA → Meta reviews it → status becomes `APPROVED`.
2. **Send** it by name to a recipient, passing parameter values for its placeholders.

---

## 2. Categories (set at creation)

| Category | Use for | Notes |
|----------|---------|-------|
| `MARKETING` | Promotions, offers, announcements, re‑engagement | Highest cost; opt‑in required |
| `UTILITY` | Transaction/account updates tied to a specific action: order confirmations, shipping, appointment reminders, OTP‑adjacent receipts | Cheaper; free if sent inside an open service window |
| `AUTHENTICATION` | One‑time passcodes, 2FA, account verification | Uses special OTP button/format; volume‑based discounts |

> **Categorization enforcement:** Meta auto‑classifies and may **re‑categorize** a template if its content doesn't match the chosen category (e.g., a "utility" template containing a promo). Since **April 16, 2025**, abusive miscategorization can be re‑categorized **without the prior 24‑hour notice**. Pricing (effective **July 1, 2025**) is **per delivered message**, priced by category + recipient country code.

---

## 3. Create a template

`POST {BASE-URL}/{WABA-ID}/message_templates`

| Field | Req | Notes |
|-------|-----|-------|
| `name` | ✅ | lowercase, digits & underscores only (e.g. `order_confirmation`) |
| `language` | ✅ | BCP‑47‑style code, e.g. `en_US` (see §7) |
| `category` | ✅ | `MARKETING` \| `UTILITY` \| `AUTHENTICATION` |
| `components` | ✅ | Array of HEADER / BODY / FOOTER / BUTTONS objects (§4) |
| `parameter_format` | — | `POSITIONAL` (`{{1}}`) or `NAMED` (`{{order_id}}`) |

### Example — marketing template with header, body, footer, buttons
```bash
curl -X POST '{BASE-URL}/{WABA-ID}/message_templates' \
  -H 'Authorization: Bearer {ACCESS-TOKEN}' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "seasonal_promotion",
    "language": "en_US",
    "category": "MARKETING",
    "components": [
      {
        "type": "HEADER",
        "format": "TEXT",
        "text": "Our {{1}} is on!",
        "example": { "header_text": ["Summer Sale"] }
      },
      {
        "type": "BODY",
        "text": "Hi {{1}}, get {{2}} off all items until {{3}}. Use code {{4}}.",
        "example": { "body_text": [["Raj", "30%", "June 30", "SUMMER30"]] }
      },
      { "type": "FOOTER", "text": "Reply STOP to opt out" },
      {
        "type": "BUTTONS",
        "buttons": [
          { "type": "QUICK_REPLY", "text": "Shop now" },
          { "type": "URL", "text": "View offer",
            "url": "https://example.com/sale/{{1}}",
            "example": ["https://example.com/sale/summer"] },
          { "type": "PHONE_NUMBER", "text": "Call us", "phone_number": "+15551234567" }
        ]
      }
    ]
  }'
```
```json
{ "id": "<TEMPLATE-ID>", "status": "PENDING", "category": "MARKETING" }
```

> Every placeholder **must** have a matching `example` value or Meta rejects the template.

---

## 4. Components reference

A template is an ordered array of component objects. Order on screen: **HEADER → BODY → FOOTER → BUTTONS**.

### 4.1 HEADER (optional, one per template)
`"type": "HEADER"` + a `format`:

| `format` | Extra fields |
|----------|--------------|
| `TEXT` | `text` (≤60 chars, supports **one** `{{1}}`), `example.header_text` |
| `IMAGE` | `example.header_handle: ["<media-handle-or-URL>"]` |
| `VIDEO` | `example.header_handle: [...]` |
| `DOCUMENT` | `example.header_handle: [...]` |
| `LOCATION` | (no example; lat/long passed at send time) |

### 4.2 BODY (required)
```json
{ "type": "BODY",
  "text": "Hello {{1}}, your order {{2}} ships on {{3}}.",
  "example": { "body_text": [["Raj", "#1234", "Monday"]] } }
```
- Up to **1024 chars**. Supports `*bold*`, `_italic_`, `~strike~`, ` ```mono``` `.
- Placeholders: positional `{{1}}` or named `{{order_id}}` (set `parameter_format`).

### 4.3 FOOTER (optional)
```json
{ "type": "FOOTER", "text": "Not interested? Tap Stop promotions." }
```
≤60 chars, no placeholders.

### 4.4 BUTTONS (optional, up to 10, with type limits)
```json
{ "type": "BUTTONS", "buttons": [ /* button objects */ ] }
```

| Button `type` | Fields | Limits / use |
|---------------|--------|--------------|
| `QUICK_REPLY` | `text` | Sends a webhook payload back; max 10 (combined with others) |
| `URL` | `text`, `url`, optional `{{1}}` in url + `example` | Static or dynamic link; max 2 |
| `PHONE_NUMBER` | `text`, `phone_number` | Click‑to‑call; max 1 |
| `COPY_CODE` | `example` (coupon code) | Copy‑to‑clipboard coupon |
| `OTP` | `otp_type`: `COPY_CODE` \| `ONE_TAP` \| `ZERO_TAP` | Authentication templates only |
| `FLOW` | `text`, `flow_id` (or `flow_name`), `navigate_screen` | Launches a WhatsApp Flow |
| `CATALOG` / `MPM` | `text` | Opens catalog / multi‑product message |

---

## 5. Manage templates

| Action | Method & URL |
|--------|--------------|
| **List** all | `GET {BASE-URL}/{WABA-ID}/message_templates` |
| **Filter** | `GET {BASE-URL}/{WABA-ID}/message_templates?name=order_confirmation&status=APPROVED&language=en_US` |
| **Get** fields | `GET {BASE-URL}/{WABA-ID}/message_templates?fields=name,status,category,components,language` |
| **Edit** (by id) | `POST {BASE-URL}/{TEMPLATE-ID}` with new `components`/`category` (only `APPROVED`/`REJECTED`/`PAUSED` editable; resets to `PENDING`) |
| **Delete by name** | `DELETE {BASE-URL}/{WABA-ID}/message_templates?name={NAME}` (deletes all languages) |
| **Delete one language** | `DELETE {BASE-URL}/{WABA-ID}/message_templates?hsm_id={TEMPLATE-ID}&name={NAME}` |

**Status values:** `PENDING` → `APPROVED` \| `REJECTED`; also `PAUSED`, `DISABLED`, `IN_APPEAL`, `PENDING_DELETION`. Status changes arrive on the `message_template_status_update` webhook field.

---

## 6. Sending a template message

`POST {BASE-URL}/{PHONE-NUMBER-ID}/messages`

### 6.1 Minimal (no variables)
```json
{
  "messaging_product": "whatsapp",
  "to": "{RECIPIENT-WA-ID}",
  "type": "template",
  "template": { "name": "hello_world", "language": { "code": "en_US" } }
}
```

### 6.2 With header media + body variables + a dynamic URL button
```json
{
  "messaging_product": "whatsapp",
  "recipient_type": "individual",
  "to": "{RECIPIENT-WA-ID}",
  "type": "template",
  "template": {
    "name": "seasonal_promotion",
    "language": { "code": "en_US" },
    "components": [
      { "type": "header",
        "parameters": [ { "type": "image", "image": { "link": "https://example.com/banner.jpg" } } ] },
      { "type": "body",
        "parameters": [
          { "type": "text", "text": "Raj" },
          { "type": "text", "text": "30%" },
          { "type": "text", "text": "June 30" },
          { "type": "text", "text": "SUMMER30" }
        ] },
      { "type": "button", "sub_type": "url", "index": "0",
        "parameters": [ { "type": "text", "text": "summer" } ] }
    ]
  }
}
```

**Parameter `type` values (body/header):** `text`, `currency` (`{ fallback_value, code, amount_1000 }`), `date_time` (`{ fallback_value }`), `image`/`video`/`document` (`{ link }` or `{ id }`), `location` (`{ latitude, longitude, name, address }`).

**Button components:** `type:"button"`, `sub_type` ∈ `quick_reply | url | flow | copy_code | catalog`, `index` (string, "0"‑based), and `parameters` (e.g. quick‑reply `{ type:"payload", payload:"..." }`).

### 6.3 Named parameters variant
If the template used `parameter_format: NAMED`:
```json
"parameters": [ { "type": "text", "parameter_name": "customer_name", "text": "Raj" } ]
```

---

## 7. Supported languages

`language.code` uses Meta's locale codes — e.g. `en_US`, `en_GB`, `es_ES`, `es_MX`, `pt_BR`, `fr_FR`, `de_DE`, `hi`, `id`, `ar`, `zh_CN`, `zh_HK`, `zh_TW`, `ja`, `ko`. Some languages are bare (`hi`, `ar`, `ja`), others locale‑qualified (`en_US`). The code in your **send** request must exactly match the template's created language. Full list: [Supported Languages](https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/supported-languages).

---

## 8. Template Library

Meta provides a **Template Library** — a catalog of pre‑written, pre‑categorized templates (common utility/auth use cases). Creating from a library entry typically yields **faster/auto‑approval** since wording is vetted. Reference the library entry name when creating instead of authoring raw text. See [Template Library](https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/template-library).

---

## 9. Template comparison (which to use)

| Need | Category | Why |
|------|----------|-----|
| Promo / re‑engagement / newsletter | `MARKETING` | Only category allowed to promote |
| Order/shipping/appointment/account update | `UTILITY` | Cheaper; free inside open service window |
| OTP / 2FA / verification | `AUTHENTICATION` | Required OTP button formats, anti‑fraud handling |

Inside an **open 24‑hour service window** you usually don't need a template at all — send free‑form messages. Templates are for **re‑opening** a conversation. See [Template Comparison](https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/template-comparison).

---

## 10. Messaging limits

Limits cap how many **unique customers you can start business‑initiated conversations with** in a rolling 24‑hour window. They do **not** restrict replies inside an open service window.

| Tier | Unique customers / 24 h |
|------|------------------------|
| Starting (unverified) | **250** |
| Tier 1 | **1,000** |
| Tier 2 | **10,000** |
| Tier 3 | **100,000** |
| Tier 4 | **Unlimited** |

- Upgrades are **automatic** based on **quality rating** + sending volume (and Business Verification to pass 250).
- **Since October 2025**, limits apply at the **Business Portfolio** level, not per phone number.
- Exceeding the limit returns an error (e.g. code `131056` / rate‑limit) — back off and retry later.
- Watch **quality rating** (Green/Yellow/Red) in the dashboard; Red can throttle or downgrade your tier.

See [Messaging Limits](https://developers.facebook.com/documentation/business-messaging/whatsapp/messaging-limits).

---

## 11. End‑to‑end demo (Node.js)

```js
require('dotenv').config();
const axios = require('axios');

const BASE = 'https://graph.facebook.com/v22.0';
const TOKEN = process.env.GRAPH_TOKEN;
const WABA_ID = process.env.WABA_ID;
const PHONE_NUMBER_ID = process.env.PHONE_NUMBER_ID;
const h = { Authorization: `Bearer ${TOKEN}`, 'Content-Type': 'application/json' };

// 1) Create a utility template
async function createTemplate() {
  const { data } = await axios.post(`${BASE}/${WABA_ID}/message_templates`, {
    name: 'order_confirmation',
    language: 'en_US',
    category: 'UTILITY',
    components: [
      { type: 'BODY',
        text: 'Hi {{1}}, your order {{2}} is confirmed and ships on {{3}}.',
        example: { body_text: [['Raj', '#1234', 'Monday']] } },
      { type: 'FOOTER', text: 'Thank you for shopping with us' }
    ]
  }, { headers: h });
  console.log('Created:', data);              // { id, status: 'PENDING', category }
  return data.id;
}

// 2) Poll status
async function getStatus(name) {
  const { data } = await axios.get(
    `${BASE}/${WABA_ID}/message_templates?name=${name}&fields=name,status`,
    { headers: h });
  return data.data?.[0]?.status;              // PENDING | APPROVED | REJECTED
}

// 3) Send it (once APPROVED)
async function sendTemplate(to) {
  const { data } = await axios.post(`${BASE}/${PHONE_NUMBER_ID}/messages`, {
    messaging_product: 'whatsapp',
    to,
    type: 'template',
    template: {
      name: 'order_confirmation',
      language: { code: 'en_US' },
      components: [
        { type: 'body', parameters: [
          { type: 'text', text: 'Raj' },
          { type: 'text', text: '#1234' },
          { type: 'text', text: 'Monday' }
        ] }
      ]
    }
  }, { headers: h });
  console.log('Sent:', data);                 // { messages: [{ id: 'wamid...' }] }
}
```

`.env`:
```
GRAPH_TOKEN=...
WABA_ID=...
PHONE_NUMBER_ID=...
```

---

## 12. Notes for *this* project

[index.js](index.js) currently only auto‑replies with free‑form text inside the service window. To message users proactively (outside 24 h), add a template flow:
1. Create + get approval for a `UTILITY` template (§3, §11).
2. Send via `type: "template"` (§6) — **not** the `type: "text"` path used today.
3. Track approval via the `message_template_status_update` webhook (add it to your subscribed fields — see [WHATSAPP_WEBHOOKS.md](WHATSAPP_WEBHOOKS.md) §2).
4. Respect messaging limits (§10) and check the `to` number isn't over your tier before bulk sends.
5. Keep the Graph version consistent (`v25.0` in index.js vs `v22.0` here).

# Messenger Platform Message Templates — Developer Implementation Guide

How to send rich **Messenger** (Facebook Page) message templates with the Send API, with working code for every template type.

> ⚠️ **This is the Messenger Platform, not WhatsApp.** It uses a **Page Access Token**, addresses users by **PSID** (Page‑Scoped ID), and posts to the **Send API**. It's a different product from the WhatsApp Cloud API covered in the other guides in this repo.

Sources (Meta — Messenger Platform docs):
- [Templates overview](https://developers.facebook.com/documentation/business-messaging/messenger-platform/send-messages/templates)
- [Button](https://developers.facebook.com/documentation/business-messaging/messenger-platform/send-messages/template/button)
- [Coupon](https://developers.facebook.com/documentation/business-messaging/messenger-platform/send-messages/template/coupon)
- [Customer Feedback](https://developers.facebook.com/documentation/business-messaging/messenger-platform/send-messages/templates/customer-feedback-template)
- [Generic](https://developers.facebook.com/documentation/business-messaging/messenger-platform/send-messages/template/generic)
- [Instant Form](https://developers.facebook.com/documentation/business-messaging/messenger-platform/send-messages/templates/instant-form-template)
- [Media](https://developers.facebook.com/documentation/business-messaging/messenger-platform/send-messages/template/media)
- [Product](https://developers.facebook.com/documentation/business-messaging/messenger-platform/send-messages/template/product)

---

## 0. Conventions & setup

| Token | Meaning |
|-------|---------|
| `BASE-URL` | `https://graph.facebook.com/v22.0` |
| `PAGE-ACCESS-TOKEN` | Token for your Facebook Page (scopes: `pages_messaging`) |
| `PSID` | Page‑Scoped ID of the recipient (from the `messages` webhook `sender.id`) |

**Send API endpoint (all templates):**
```
POST {BASE-URL}/me/messages?access_token={PAGE-ACCESS-TOKEN}
```
(You can also POST to `{BASE-URL}/{PAGE-ID}/messages`.)

**Universal request envelope:**
```json
{
  "recipient": { "id": "{PSID}" },
  "messaging_type": "RESPONSE",
  "message": {
    "attachment": {
      "type": "template",
      "payload": {
        "template_type": "<TYPE>"
        /* template-specific fields */
      }
    }
  }
}
```

`messaging_type` ∈ `RESPONSE` (replying within 24 h) · `UPDATE` · `MESSAGE_TAG` (with a `tag`, for sending outside the 24‑hour window).

**Success response (all sends):**
```json
{ "recipient_id": "1254477...", "message_id": "AG5Hz2U..." }
```

---

## 1. Template types at a glance

| `template_type` | What it renders | Key payload |
|-----------------|-----------------|-------------|
| `button` | Text + up to 3 buttons | `text`, `buttons[]` |
| `generic` | Carousel of up to 10 cards | `elements[]` (title/subtitle/image/buttons) |
| `media` | Playable image/video + buttons | `elements[]` (media_type, attachment_id/url) |
| `coupon` | Reveal‑code promo card | `title`, `coupon_code`, … |
| `customer_feedback` | Native CSAT/NPS/CES survey | `feedback_screens[]` |
| `instant_form` | Lead‑gen form in thread | `form_id` |
| `product` | Catalog product card(s) | `elements[].id` |
| `receipt` | Order/receipt summary | `elements[]`, `summary` (not detailed here) |

---

## 2. Buttons (shared by button/generic/media)

| Button `type` | Fields | Notes |
|---------------|--------|-------|
| `web_url` | `url`, `title`, `webview_height_ratio` (`compact`/`tall`/`full`), `messenger_extensions`, `fallback_url` | Opens URL in webview |
| `postback` | `title`, `payload` | Fires a `messaging_postbacks` webhook |
| `phone_number` | `title`, `payload` (E.164 phone) | Click‑to‑call |
| `account_link` | `account_linking_url` | Start account linking |
| `account_unlink` | — | Unlink |
| `game_play` | `title`, `payload` | Launch Instant Game |

Button title: keep **≤ 20 characters**. Max **3** buttons per message/element.

```json
{ "type": "web_url", "url": "https://example.com", "title": "Visit",
  "webview_height_ratio": "full", "messenger_extensions": false,
  "fallback_url": "https://example.com" }
```
```json
{ "type": "postback", "title": "Start", "payload": "START_PAYLOAD" }
```
```json
{ "type": "phone_number", "title": "Call us", "payload": "+15551234567" }
```

---

## 3. Button template

Text message with up to 3 call‑to‑action buttons. `text` ≤ **640 chars**.

```json
{
  "recipient": { "id": "{PSID}" },
  "message": {
    "attachment": {
      "type": "template",
      "payload": {
        "template_type": "button",
        "text": "What do you want to do next?",
        "buttons": [
          { "type": "web_url", "url": "https://www.messenger.com", "title": "Visit Messenger" },
          { "type": "postback", "title": "Start Chatting", "payload": "START" }
        ]
      }
    }
  }
}
```

---

## 4. Generic template (carousel)

Up to **10** horizontally scrollable cards.

| Element field | Req | Notes |
|---------------|-----|-------|
| `title` | ✅ | ≤ 80 chars |
| `subtitle` | — | ≤ 80 chars |
| `image_url` | — | 1.91:1 aspect ratio |
| `default_action` | — | Tap behavior; same props as a `web_url` button **without** `title` |
| `buttons` | — | up to 3 |

```json
{
  "recipient": { "id": "{PSID}" },
  "message": {
    "attachment": {
      "type": "template",
      "payload": {
        "template_type": "generic",
        "sharable": true,
        "elements": [
          {
            "title": "Welcome!",
            "image_url": "https://example.com/hat.jpg",
            "subtitle": "We have the right hat for everyone.",
            "default_action": {
              "type": "web_url",
              "url": "https://www.example.com/",
              "webview_height_ratio": "tall"
            },
            "buttons": [
              { "type": "web_url", "url": "https://www.example.com/", "title": "View Website" },
              { "type": "postback", "title": "Start Chatting", "payload": "DEVELOPER_DEFINED_PAYLOAD" }
            ]
          }
        ]
      }
    }
  }
}
```

---

## 5. Media template

Plays an image/GIF/video with optional buttons. **Single element only.** Audio not supported.

| Field | Notes |
|-------|-------|
| `media_type` | `image` or `video` |
| `attachment_id` | From the [Attachment Upload API]; **mutually exclusive** with `url` |
| `url` | **Facebook‑hosted** URLs only |
| `buttons` | Up to 3 (web_url / postback / phone_number) |

```json
{
  "recipient": { "id": "{PSID}" },
  "message": {
    "attachment": {
      "type": "template",
      "payload": {
        "template_type": "media",
        "elements": [
          {
            "media_type": "image",
            "attachment_id": "{ATTACHMENT_ID}",
            "buttons": [
              { "type": "web_url", "url": "https://example.com/buy", "title": "Buy now" }
            ]
          }
        ]
      }
    }
  }
}
```
> External (non‑Facebook) media must be uploaded first via the Attachment Upload API to get an `attachment_id`.

---

## 6. Coupon template

Promo card with a built‑in (non‑customizable) **"Reveal code"** button. A webhook fires when the code is revealed.

| Field | Req | Notes |
|-------|-----|-------|
| `title` | ✅ | Headline |
| `coupon_code` | ✅ | Code revealed on tap |
| `subtitle` | — | Terms / expiry |
| `coupon_pre_message` | — | Greeting before the card |
| `image_url` | — | Promo image |
| `coupon_url` | — | Secondary button destination |
| `coupon_url_button_title` | — | Secondary button label (default "Shop now") |
| `payload` | — | Custom metadata returned in webhook |

```json
{
  "recipient": { "id": "{PSID}" },
  "message": {
    "attachment": {
      "type": "template",
      "payload": {
        "template_type": "coupon",
        "title": "10% off everything",
        "subtitle": "Limit 1 per customer. Expires Oct 1, 2026",
        "coupon_code": "10PERCENT",
        "coupon_pre_message": "Here's a deal just for you!",
        "image_url": "https://www.myshop.com/sale.png",
        "coupon_url": "https://www.myshop.com/",
        "coupon_url_button_title": "Shop now",
        "payload": "promo_q3_2026"
      }
    }
  }
}
```

---

## 7. Customer feedback template

Native in‑thread survey aggregating **CSAT / NPS / CES** scores. Must follow a real interaction (post‑purchase, post‑support); promotional or unrelated surveys are disallowed.

| Field | Notes |
|-------|-------|
| `title` | Survey heading |
| `subtitle` | Sub‑heading |
| `button_title` | CTA that opens the survey |
| `feedback_screens[]` | One+ screens, each with `questions[]` |
| `business_privacy.url` | Privacy policy link |
| `expires_in_days` | Survey validity window |

Each **question**: `id`, `type` (`csat` / `nps` / `ces` / `free_form`), `title`, optional `score_label`, `score_option`, and a `follow_up` free‑form prompt.

```json
{
  "recipient": { "id": "{PSID}" },
  "message": {
    "attachment": {
      "type": "template",
      "payload": {
        "template_type": "customer_feedback",
        "title": "How did we do?",
        "subtitle": "Your feedback helps us improve",
        "button_title": "Start survey",
        "feedback_screens": [
          {
            "questions": [
              {
                "id": "csat_q1",
                "type": "csat",
                "title": "How satisfied were you with your purchase?",
                "score_label": "neg_pos",
                "score_option": "five_stars",
                "follow_up": { "type": "free_form", "title": "Tell us more" }
              }
            ]
          }
        ],
        "business_privacy": { "url": "https://www.myshop.com/privacy" },
        "expires_in_days": 7
      }
    }
  }
}
```
> Results are delivered via the feedback webhook; aggregate them as CSAT/NPS/CES scores.

---

## 8. Instant form template (lead generation)

Sends an eligible **lead‑gen form** directly in the thread.

| Field | Notes |
|-------|-------|
| `template_type` | `instant_form` |
| `form_id` | ID of an eligible lead form (questions limited to `CUSTOM`, `EMAIL`, `FIRST_NAME`, `FULL_NAME`, `LAST_NAME`, `PHONE`) |

```json
{
  "recipient": { "id": "{PSID}" },
  "message": {
    "attachment": {
      "type": "template",
      "payload": {
        "template_type": "instant_form",
        "form_id": "{INSTANT_FORM_ID}"
      }
    }
  }
}
```

- On submit, the **`messaging_in_thread_lead_form_submit`** webhook fires with `sender.id`, `recipient.id`, `timestamp`, and the `form_id`.
- Error `2018382` = wrong/ineligible form ID.

---

## 9. Product template

Renders catalog products; details (image, title, price) auto‑populate from your catalog. Requires **Graph API v8.0+**. Up to **10** products → carousel.

| Field | Notes |
|-------|-------|
| `elements[].id` | `product_id` from the Catalog API / Commerce Manager, owned by the same Page |

```json
{
  "recipient": { "id": "{PSID}" },
  "message": {
    "attachment": {
      "type": "template",
      "payload": {
        "template_type": "product",
        "elements": [
          { "id": "{PRODUCT_ID_1}" },
          { "id": "{PRODUCT_ID_2}" }
        ]
      }
    }
  }
}
```
- Invalid product id → error `100 / 2018320`. API < v8.0 → error `100 / 2018328`.

---

## 10. Reusable Node.js sender

```js
require('dotenv').config();
const axios = require('axios');

const BASE = 'https://graph.facebook.com/v22.0';
const TOKEN = process.env.PAGE_ACCESS_TOKEN;

async function sendTemplate(psid, payload, messagingType = 'RESPONSE') {
  const { data } = await axios.post(
    `${BASE}/me/messages`,
    {
      recipient: { id: psid },
      messaging_type: messagingType,
      message: { attachment: { type: 'template', payload } },
    },
    { params: { access_token: TOKEN } }
  );
  return data;   // { recipient_id, message_id }
}

// examples ----------------------------------------------------------
const buttons = (psid) => sendTemplate(psid, {
  template_type: 'button',
  text: 'What next?',
  buttons: [
    { type: 'postback', title: 'Browse', payload: 'BROWSE' },
    { type: 'web_url', url: 'https://example.com', title: 'Website' },
  ],
});

const carousel = (psid, items) => sendTemplate(psid, {
  template_type: 'generic',
  elements: items.slice(0, 10).map((it) => ({
    title: it.title,
    subtitle: it.subtitle,
    image_url: it.image,
    buttons: [{ type: 'web_url', url: it.url, title: 'View' }],
  })),
});

const coupon = (psid) => sendTemplate(psid, {
  template_type: 'coupon',
  title: '10% off everything',
  coupon_code: '10PERCENT',
  coupon_pre_message: "Here's a deal just for you!",
});

module.exports = { sendTemplate, buttons, carousel, coupon };
```

`.env`:
```
PAGE_ACCESS_TOKEN=your_page_access_token
```

---

## 11. Receiving interactions (webhook)

Subscribe your Page to Messenger webhooks. Relevant events:

| Event | Field | Fires on |
|-------|-------|----------|
| Postback | `messaging_postbacks` | A `postback` button tap → `postback.payload` |
| Coupon reveal | `messaging_referrals` / coupon webhook | Code revealed → your `payload` |
| Lead form submit | `messaging_in_thread_lead_form_submit` | Instant form completed |
| Messages | `messages` | Inbound text/attachments; `sender.id` is the PSID to reply to |

The verification handshake (`hub.mode`/`hub.challenge`/`hub.verify_token`) and signature validation are identical to the pattern in [WHATSAPP_WEBHOOKS.md](WHATSAPP_WEBHOOKS.md) §4–5 — the same Meta webhook mechanism, just `object: "page"` instead of `whatsapp_business_account`.

---

## 12. Notes for *this* project

[index.js](index.js) is wired for the **WhatsApp** Cloud API (different token, recipient by phone number, `messaging_product: "whatsapp"`). Messenger templates need a **separate integration**:
1. A Facebook Page + **Page Access Token** (not the WhatsApp token in `.env`).
2. A distinct webhook subscription with `object: "page"`; you can reuse the same `/webhook` route but branch on `body.object`.
3. Address users by **PSID** from `sender.id`, post to `/me/messages` (§0), not `/{phone_number_id}/messages`.
4. Keep the Graph version consistent across the repo (`v22.0` here vs `v25.0` in index.js).

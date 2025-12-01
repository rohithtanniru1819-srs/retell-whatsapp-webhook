# Repository: retell-whatsapp-webhook

This repo contains a ready-to-deploy Node.js webhook that accepts Retell.ai order POSTs and forwards formatted messages to WhatsApp Cloud API (customer + owner). Deploy to Render / Railway / Heroku.

---

-- FILE: package.json
{
  "name": "retell-whatsapp-webhook",
  "version": "1.0.0",
  "description": "Webhook to receive Retell.ai order JSON and send WhatsApp messages using WhatsApp Cloud API",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "engines": {
    "node": ">=18"
  },
  "dependencies": {
    "body-parser": "^1.20.2",
    "dotenv": "^16.3.1",
    "express": "^4.18.2",
    "node-fetch": "^2.6.7"
  }
}

---

-- FILE: .gitignore
node_modules/
.env
npm-debug.log
.DS_Store

---

-- FILE: .env.example
# Copy to .env and fill values
PORT=3000
WHATSAPP_TOKEN=EAA...your_whatsapp_cloud_token_here...
PHONE_NUMBER_ID=10987654321
OWNER_WHATSAPP=+91XXXXXXXXXX
# Optional: secret to verify Retell webhook requests (set same value in Retell if you want verification)
RETELL_HMAC_SECRET=your_shared_secret_here

---

-- FILE: server.js
require('dotenv').config();
const express = require('express');
const bodyParser = require('body-parser');
const fetch = require('node-fetch');
const crypto = require('crypto');

const app = express();
app.use(bodyParser.json({ limit: '256kb' }));

const TOKEN = process.env.WHATSAPP_TOKEN;
const PHONE_ID = process.env.PHONE_NUMBER_ID;
const OWNER_WHATSAPP = process.env.OWNER_WHATSAPP;
const RETELL_HMAC_SECRET = process.env.RETELL_HMAC_SECRET || '';

if (!TOKEN || !PHONE_ID) {
  console.error('Missing WHATSAPP_TOKEN or PHONE_NUMBER_ID in environment');
  process.exit(1);
}

function formatOrderText(payload) {
  const name = payload.customer_name || payload.name || 'Customer';
  const phone = payload.phone || 'Unknown';
  const delivery = payload.delivery_type || 'Pickup';
  const address = payload.address || '';
  const notes = payload.notes || '';
  const items = (payload.order || []).map(i => `• ${i.qty || 1} x ${i.item || i.name || 'item'}`).join('\n') || 'No items';

  return `Tree House Restaurant - Order\n\nCustomer: ${name}\nPhone: ${phone}\nType: ${delivery}\nAddress: ${address}\n\nItems:\n${items}\n\nNotes: ${notes}\n\nThank you for ordering!`;
}

async function sendWhatsApp(to, bodyText) {
  const url = `https://graph.facebook.com/v17.0/${PHONE_ID}/messages`;
  const payload = {
    messaging_product: 'whatsapp',
    to: to.replace(/\s+/g, ''),
    type: 'text',
    text: { body: bodyText }
  };

  const res = await fetch(url, {
    method: 'POST',
    headers: { Authorization: `Bearer ${TOKEN}`, 'Content-Type': 'application/json' },
    body: JSON.stringify(payload)
  });

  const json = await res.json();
  if (!res.ok) {
    const err = new Error('WhatsApp send failed');
    err.response = json;
    throw err;
  }
  return json;
}

function verifyRetellSignature(req) {
  // Optional: verify HMAC if RETELL_HMAC_SECRET is set
  if (!RETELL_HMAC_SECRET) return true;
  const sig = req.headers['x-retell-signature'] || req.headers['x-hub-signature'] || '';
  if (!sig) return false;
  const raw = JSON.stringify(req.body);
  const hmac = crypto.createHmac('sha256', RETELL_HMAC_SECRET).update(raw).digest('hex');
  return hmac === sig || `sha256=${hmac}` === sig;
}

app.post('/retell-webhook', async (req, res) => {
  try {
    if (!verifyRetellSignature(req)) {
      return res.status(401).json({ ok: false, error: 'Invalid signature' });
    }

    const payload = req.body || {};
    // Map common keys
    const customerPhone = (payload.phone || payload.customer_phone || payload.customerPhone || '') + '';

    const msgToCustomer = formatOrderText(payload);
    const msgToOwner = `New order received:\n\n${msgToCustomer}`;

    const results = {};

    // Send to owner (always attempt)
    if (OWNER_WHATSAPP) {
      try {
        const r1 = await sendWhatsApp(OWNER_WHATSAPP, msgToOwner);
        results.owner = r1;
      } catch (err) {
        results.owner_error = err.response || err.message;
        console.error('Owner WhatsApp error', results.owner_error);
      }
    }

    // Send to customer if phone exists
    if (customerPhone && customerPhone.length > 5) {
      try {
        const r2 = await sendWhatsApp(customerPhone, msgToCustomer);
        results.customer = r2;
      } catch (err) {
        results.customer_error = err.response || err.message;
        console.error('Customer WhatsApp error', results.customer_error);
      }
    } else {
      results.customer = 'no-customer-number';
    }

    return res.status(200).json({ ok: true, results });
  } catch (err) {
    console.error('Webhook processing error', err);
    return res.status(500).json({ ok: false, error: err.message || err });
  }
});

const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`Webhook listening on http://0.0.0.0:${port}`));

---

-- FILE: README.md
# Retell → WhatsApp Webhook (Node.js)

This repository contains a simple Node.js webhook that accepts POST requests from Retell.ai (order data) and forwards formatted messages to WhatsApp Cloud API (customer + owner).

## Files included
- `server.js` - Express webhook server
- `package.json` - Node dependencies and scripts
- `.env.example` - Environment variables template
- `.gitignore`

## Quick start (local)
1. Copy `.env.example` to `.env` and fill in values (`WHATSAPP_TOKEN`, `PHONE_NUMBER_ID`, `OWNER_WHATSAPP`).
2. Install dependencies:
   ```bash
   npm install
   ```
3. Run the server:
   ```bash
   npm start
   ```
4. Test with curl:
   ```bash
   curl -X POST http://localhost:3000/retell-webhook \
     -H "Content-Type: application/json" \
     -d '{"customer_name":"Rohith","phone":"+918121223832","order":[{"item":"Chicken Biryani","qty":2}],"delivery_type":"Delivery","address":"Madhapur"}'
   ```

## Deploy to Render
1. Create a new **Web Service** on Render.
2. Connect your GitHub repo and select this repository.
3. Build command: `npm install`
4. Start command: `npm start`
5. Add environment variables on Render (WHATSAPP_TOKEN, PHONE_NUMBER_ID, OWNER_WHATSAPP, RETELL_HMAC_SECRET optional).
6. After deploy, copy the public URL and set the webhook URL in Retell: `https://<your-service>.onrender.com/retell-webhook`.

## Security
- Set `RETELL_HMAC_SECRET` and configure Retell to send a matching header for verification (optional).

## Notes
- WhatsApp Cloud API allows free-form messages within the 24-hour user interaction window. For messages outside that window, use approved templates.
- Monitor logs for error responses from Meta and retry with exponential backoff for 5xx errors.

---

-- End of repo

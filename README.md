# Cloudflare WhatsApp Agent (Golden Path)

A high-performance, low-latency WhatsApp Agent built on Cloudflare Workers, designed to handle reservations and menu recommendations using native "WhatsApp Flows".

## Architecture: The "Golden Path"

This project follows a strict **"Fast Path / Slow Path"** architecture to meet WhatsApp's 3-second timeout requirement while performing complex AI tasks.

### 1. The Fast Path (Webhook Response)
* **Goal**: Acknowledge receipt immediately (200 OK) to prevent WhatsApp retries.
* **Steps**:
  1. **Verify Signature**: Validate `X-Hub-Signature-256` to block spoofing.
  2. **Idempotency Check**: Check KV for `msg_{ID}`. If exists, ignore (duplicate).
  3. **Enqueue**: Push valid messages to a **Cloudflare Queue**.
  4. **Respond**: Return 200 OK.

### 2. The Slow Path (Queue Consumer)
* **Goal**: Perform heavy logic (AI, DB) and send a pro-active reply.
* **Steps**:
  1. **Fetch Context**: Retrieve state from KV and Inventory from D1.
  2. **AI Reasoning**: Call **Workers AI** (Llama 3) for menu recommendations.
  3. **Send Reply**: Use WhatsApp Graph API to send the response back to the user.

## Stack

* **Runtime**: Cloudflare Workers
* **Framework**: Hono
* **Database**: Cloudflare D1 (SQLite) - Reservations & Inventory
* **State/Cache**: Cloudflare KV - Conversation State & Idempotency
* **Async**: Cloudflare Queues
* **AI**: Cloudflare Workers AI (Llama 3)

## Setup

1. **Install Dependencies**
   ```bash
   npm install
   ```

2. **Configure Secrets**
   ```bash
   npx wrangler secret put WHATSAPP_TOKEN
   npx wrangler secret put WHATSAPP_SECRET
   ```

3. **Deploy**
   ```bash
   npx wrangler deploy
   ```

## Key Features (Tier 0 MVP)

* **Security**: `X-Hub-Signature-256` verification.
* **Reliability**: Queue-based processing for AI tasks.
* **Data**: Full SQLite schema for Menu and Reservations.
* **State**: Per-user conversation tracking.

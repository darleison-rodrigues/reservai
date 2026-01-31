# Implementation Plan - Cloudflare WhatsApp Agent

## Goal Description
Build a high-performance, low-cost WhatsApp Agent using Cloudflare-Native architecture. The agent will handle reservations, query context (mock Google Places/Weather), check inventory (D1), and use AI (Workers AI Llama 3) to recommend menu items via WhatsApp Flows.

## User Review Required
> [!IMPORTANT]
> **External API Dependencies**: This implementation relies on **Meta Business API** (for WhatsApp) and **Google Places API**.
> - I will set up the code structure and mock the API responses where real keys are missing.
> - You will need to provide `WHATSAPP_ACCESS_TOKEN`, `PHONE_NUMBER_ID`, and `GOOGLE_API_KEY` in your `.dev.vars` or Cloudflare secrets.

## Proposed Changes

## Proposed Changes

### 1. Architecture: The "Golden Path" (Tier 0 & 1)
- **Security (Tier 0)**: Middleware to validate `X-Hub-Signature-256`. Critical to prevent spoofing.
- **Async Processing (Tier 0)**: Use **Cloudflare Queues** to handle the AI Agent processing.
    - **Fast Path**: Webhook -> Verify Signature -> Check Idempotency -> Audit in KV -> Push to Queue -> Return 200 OK.
    - **Slow Path**: Queue Worker -> Fetch Context -> Check Inventory -> Run AI -> Send WhatsApp Reply.
- **Idempotency (Tier 0)**: KV check `msg_${msgId}` to prevent duplicate processing of retried webhooks.
- **State Management (Tier 0)**: KV-based Finite State Machine (FSM) to track conversation flow (e.g., `state:<phone>`).

### 2. Data Model (D1)
- **reservations**: `id` (PK), `phone`, `time`, `party_size`, `status`, `created_at`.
- **menu_items**: `id` (PK), `name`, `stock`, `tags`, `description`.

### 3. File Structure
- `wrangler.toml`: Configures D1, KV (`state`, `idempotency`), and Queues (`agent-processing-queue`).
- `src/index.ts`:
    - `POST /webhook`: The fast path handler.
    - `queue()`: The slow path consumer (AI logic).

### 4. Integration
- **WhatsApp**: Standard Graph API for sending messages. `verifyToken` for initial setup.
- **AI**: Workers AI (`@cf/meta/llama-3-8b-instruct`) called from the Queue consumer.

### 5. Marketing Site
- `public/index.html`: Static HTML mimicking Zapia.com.
- served via `serveStatic` or Cloudflare Pages.

## Verification Plan

### Automated Tests
- basic unit tests for Hono routes using `app.request()`.
- Mock D1 and AI bindings for testing logic.

### Manual Verification
- Deploy to Cloudflare Workers (`npx wrangler deploy`).
- Verify Webhook verification with a browser or curl.
- Test "Flow" trigger by sending a mock POST request to the webhook.

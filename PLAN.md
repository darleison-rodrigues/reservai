# Implementation Plan: Cloudflare WhatsApp Agent (The Golden Path)

This document integrates the refined architecture with the core implementation requirements from the "Brain" of the project. It outlines a high-performance, low-cost native Cloudflare solution for the Reserve It! WhatsApp Agent.

## 1. Goal Description
Build a production-ready WhatsApp Agent using Cloudflare-Native architecture. The agent handles reservations, inventory checks (D1), and provides AI-driven recommendations (Llama 3) via structured WhatsApp Flows.

## 2. Market Analysis & Strategy (Insights from Jota.ai & Zapia.com)
*   **The "Zapia" Effect**: We leverage a "GenAI-first" approach but target the **B2B2C** gap. Where Zapia acts for the user, **Reserve It!** provides the infrastructure for the business.
*   **The "Jota" Standard**: We adopt high-security standards (signature verification, idempotency) comparable to financial-grade AI agents.
*   **Edge Advantage**: By using **Cloudflare Workers AI**, we achieve lower latency and cost-per-token than the GCP/Gemini stacks used by competitors, making the model more sustainable for small businesses.

## 3. Critical Requirements (User Review Required)
> [!IMPORTANT]
> **External API Dependencies**: This implementation requires:
> - **Meta Business API**: `WHATSAPP_ACCESS_TOKEN`, `PHONE_NUMBER_ID`, `WHATSAPP_APP_SECRET`.
> - **Google Places API**: `GOOGLE_API_KEY` (for location/context).
> 
> *The initial code structure will mock responses where keys are missing to allow development to proceed.*

---

## 3. Architecture: The "Safe & Fast Path"

WhatsApp requires a response within **3 seconds**. We use a decoupled architecture to ensure 100% reliability.

### A. The Ingress Worker (Fast Path)
*   **Security**: Middleware verifies `X-Hub-Signature-256` using the App Secret.
*   **Webhook Handlers**: 
    *   `GET /webhook`: Verification challenge for Meta setup.
    *   `POST /webhook`: Inbound message processing.
*   **Idempotency & Audit**: Check/Set KV for `msg_${wamid}`. If skip, return 200 OK. Log audit in KV.
*   **Action**: Enqueue payload to **Cloudflare Queues** (`agent-processing-queue`) and return `200 OK`.

### B. The Logic Worker (Slow Path)
*   **Trigger**: Queue Consumer.
*   **Workflow**:
    1.  **State Management**: Load conversation state (FSM) from KV (e.g., `state:<phone>`).
    2.  **Context**: Fetch inventory from D1 and external context (Mock Google Places/Weather).
    3.  **AI Reasoning**: Call **Workers AI** (`@cf/meta/llama-3-8b-instruct`).
    4.  **Response**: Send formatted WhatsApp message or Flow via Graph API.

---

## 4. Technical Stack & Data Model

### Data Model (Cloudflare D1)
*   **`businesses`**: Config, WhatsApp IDs, working hours.
*   **`reservations`**: `id`, `phone`, `time`, `party_size`, `status`.
*   **`menu_items`**: `id`, `name`, `stock`, `tags`, `description`.

### State & Cache (Cloudflare KV)
*   `idempotency:{wamid}` -> TTL: 24h.
*   `state:{phone}` -> Finite State Machine node.

---

## 5. File Structure
*   `wrangler.toml`: Resources configuration (D1, KV, Queues).
*   `src/index.ts`: Unified Hono app with Webhook routing and `queue()` handler.
*   `public/index.html`: Marketing site mimicking Zapia.com (served via `serveStatic`).

---

## 6. Implementation Roadmap (Tier 0 MVP)

### Step 1: Foundation
- [ ] Implement signature verification and webhook challenge.
- [ ] Set up KV for idempotency and state.
- [ ] Initialize D1 schema and seed basic menu/business data.

### Step 2: Queue & AI
- [ ] Implement Queue consumer in `src/index.ts`.
- [ ] Integrate Workers AI with "Restaurant Assistant" prompt.
- [ ] Add basic reservation logic (D1 Insertion).

### Step 3: API & UI
- [ ] Implement WhatsApp outbound message helper.
- [ ] Build the Zapia-inspired marketing landing page.

---

## 7. Verification Plan

### Automated
- Hono route testing using `app.request()`.
- Mock D1/AI binding tests for state transitions.

### Manual
- `npx wrangler deploy` to a staging environment.
- Verify meta challenge with browser.
- Simulate incoming WhatsApp webhook via `curl`.

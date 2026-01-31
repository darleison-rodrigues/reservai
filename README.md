# Cloudflare WhatsApp Agent (Golden Path) - Refined Implementation Plan

This document outlines a robust, production-ready architecture for the Reserve It! WhatsApp Agent, optimized for the Cloudflare ecosystem. It addresses critical gaps in security, reliability, and state management.

## 1. Architecture: The "Safe & Fast Path"

WhatsApp requires a response to webhooks within **3 seconds**. To ensure we never time out while performing AI inference or database operations, we adopt a decoupled architecture.

### A. Phase 1: The Ingress Worker (Fast Path)
*   **Purpose**: Receive, verify, and acknowledge webhooks.
*   **Security**:
    *   **Signature Verification**: Validate `X-Hub-Signature-256` using the `WHATSAPP_APP_SECRET`. Reject any request that fails verification.
    *   **Webhook Verification**: Handle `GET` requests for the Hub Challenge during initial setup.
*   **Reliability**:
    *   **Idempotency Check**: Query **Cloudflare KV** for the unique `wamid` (WhatsApp Message ID). If it exists, return `200 OK` immediately and skip processing (this prevents duplicate processing on retries).
*   **Action**: Enqueue the valid payload into a **Cloudflare Queue** and return `200 OK` to WhatsApp.

### B. Phase 2: The Logic Worker (Slow Path)
*   **Purpose**: Process messages, interact with AI/DB, and reply to the user.
*   **Trigger**: Consumes messages from the Cloudflare Queue.
*   **Workflow**:
    1.  **Context Loading**: Fetch the user's `conversation_state` from KV and relevant restaurant data from **Cloudflare D1**.
    2.  **AI Reasoning**: Use **Workers AI (Llama 3.1)** to interpret the message and determine the intent (Reservation, Menu Inquiry, General Question).
    3.  **Action Execution**:
        *   **Reservations**: Insert/Update records in the D1 `reservations` table.
        *   **Menu/Info**: Query the D1 `inventory` table.
    4.  **Send Response**: Use the WhatsApp Graph API to send a message back to the user.
*   **Resilience**: If the Graph API or AI call fails, the Queue will automatically retry the message with exponential backoff.

---

## 2. Technical Stack

*   **Runtime**: Cloudflare Workers (Hono framework)
*   **Database**: Cloudflare D1 (Reservations, Inventory, Settings)
*   **State/Cache**: Cloudflare KV (Conversation state, Idempotency)
*   **Messaging**: Cloudflare Queues
*   **AI**: Cloudflare Workers AI (Llama 3.1)
*   **Validation**: Zod (for payload integrity)

---

## 3. Data Schema (Cloudflare D1)

### Table: `businesses`
*   `id`: UUID (PK)
*   `name`: TEXT
*   `whatsapp_phone_number_id`: TEXT
*   `location_id`: TEXT
*   `settings`: JSON (Working hours, timezone, reservation limits)

### Table: `reservations`
*   `id`: UUID (PK)
*   `business_id`: FK
*   `customer_id`: TEXT (WhatsApp ID)
*   `date_time`: TIMESTAMP
*   `party_size`: INTEGER
*   `status`: TEXT (confirmed, pending, cancelled)

### Table: `menu_items`
*   `id`: UUID
*   `business_id`: FK
*   `name`: TEXT
*   `description`: TEXT
*   `price`: REAL
*   `category`: TEXT

---

## 4. State & Idempotency (Cloudflare KV)

*   **Idempotency Store**: `idempotency:{wamid}` -> `processed_at` (TTL: 24h)
*   **Conversation State**: `session:{customer_id}` -> `{ last_node, history_token, metadata }` (TTL: 30m)

---

## 5. Implementation Roadmap (Tier 0 MVP)

### Step 1: Secure Webhook & Enqueueing
- [ ] Implement signature verification middleware.
- [ ] Implement KV-based idempotency.
- [ ] Set up Cloudflare Queue and the first Ingress Worker.

### Step 2: D1 & AI Integration
- [ ] Create D1 migrations for businesses, reservations, and menu.
- [ ] Implement the Queue Consumer.
- [ ] Integrate Workers AI with a specialized "Restaurant Assistant" system prompt.

### Step 3: Action Dispatcher & Response
- [ ] Implement WhatsApp Flow integration for structured data entry (e.g., date picker).
- [ ] Build the message sender utility for WhatsApp Graph API.

### Step 4: Observability
- [ ] Add error logging and monitoring for failed AI inferences.
- [ ] Implement a dry-run mode for testing without hitting live Graph API.

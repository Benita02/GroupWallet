# GroupWallet — System Architecture

**Purpose:** provide a developer-facing overview of stacks, component interactions, data model highlights, and technical tradeoffs.

---

## High-level architecture

<img width="300" height="799" alt="api services - GroupWallet" src="https://github.com/user-attachments/assets/4ae854cc-052a-46e9-ad75-c2c576a4165f" />


1. **Client (Mobile / Web)**

   - React Native for native mobile apps OR Next.js for mobile-optimized web.
   - Responsibilities: authentication, wallet UI, payment flows, loan creation, push notifications.

2. **API Gateway / Backend**

   - Node.js + Express or Django REST Framework
   - Responsibilities: business logic, authentication (JWT), rate limiting, orchestration of notifications, webhook endpoint handlers (PSP callbacks).

3. **Services**

   - **Payments Service**: interacts with PSP (Paystack/Flutterwave), handles payouts and verification.
   - **Notifications Service**: manages push, email, SMS, WhatsApp, and call escalations (integrates Twilio + WhatsApp Business API + SMTP).
   - **Ledger Service**: canonical source of truth for group balances, transactions, loans, approvals.
   - **Approval Engine**: enforces 51% rule for withdrawals and tracks voting state.

4. **Database**

   - PostgreSQL (ACID guarantees for ledger correctness).
   - Tables: `users`, `groups`, `wallets`, `transactions`, `loans`, `approvals`, `receipts`, `notifications_log`.

5. **Background jobs**

   - Queue (Redis + Bull or Celery) to handle async tasks: notification escalation, periodic reconciliation, webhook processing.

6. **Third-party integrations**
   - Payment provider (local PSP) for collections/payouts.
   - Twilio for SMS & calls.
   - WhatsApp Business API for messages.
   - Cloud storage (S3) for receipts.

---

## Component communication (sequence)

1. User initiates a contribution or loan in the app.
2. Client calls Backend API `POST /groups/:id/transactions`.
3. Backend persists the transaction in `transactions` (state = pending).
4. Payments Service requests checkout with PSP; PSP returns a callback to `POST /webhooks/payment`.
5. On successful callback: Backend marks transaction complete, updates ledger.
6. If it's a withdrawal request: Approval Engine checks group votes; if >=51% approval, Payment Service initiates payout.
7. For loans: if repayment due date passes and unpaid, Notification Service triggers escalation workflow (email → SMS → WhatsApp → Call) using queued jobs.


<img width="400" height="1536" alt="image" src="https://github.com/user-attachments/assets/baa284e3-71d9-410e-b363-2a37697af972" />


---

## Data integrity & concurrency

- Use DB transactions for multi-step operations (create transaction + ledger update).
- Optimistic locking on wallet balance updates (e.g., row versioning) to avoid race conditions.
- Audit log table for all money movements and approval votes.

---

## Security & compliance

- JWT access tokens and refresh tokens.
- Role-based access checks for group operations.
- Encrypt sensitive fields at rest where required (e.g., PSP metadata).
- Webhook signature verification for PSP and Twilio callbacks.

---

## Scalability & cost considerations

- Start monolith + modular services — cheapest to iterate. Split into microservices when traffic grows.
- Use managed DB with read replicas for scaling reads.
- Use provider-agnostic integration layer for payments to allow swapping PSPs per country.

---

## Why this is feasible

- Most components are standard, stable tech (React/Next, Node/Django, PostgreSQL).
- Payment and notification integrations are well-documented by providers (Paystack/Flutterwave/Twilio).
- The complex logic (approvals, loan escalation) is application logic — implementable with robust queues and DB transactions.

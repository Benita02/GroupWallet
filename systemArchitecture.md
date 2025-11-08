# GroupWallet — System Architecture

**Purpose:** provide a developer-facing overview of stacks, component interactions, data model highlights, and technical tradeoffs.

---

## High-level architecture

<img width="355" height="1536" alt="image" src="https://github.com/user-attachments/assets/1813d3ea-fd37-43f6-945e-43467a7513fe" />



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

<img width="400" height="1536" alt="image" src="https://github.com/user-attachments/assets/0e4cd777-3fc1-41dc-9557-5da4f90ca6ee">



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

## Why This Is Feasible

GroupWallet is technically feasible for several reasons:

1. **Mature, well-supported technology stack**  
   - Frontend: React Native or Next.js — widely used, stable, and mobile-first.  
   - Backend: Node.js/Express or Django REST Framework — reliable, scalable, and supported by large developer communities.  
   - Database: PostgreSQL — ACID-compliant for financial transactions and ledger integrity.

2. **Standard, well-documented integrations**  
   - Payment providers (Paystack, Flutterwave) provide APIs for secure collections and payouts.  
   - Notification providers (Twilio, WhatsApp Business API, SMTP) are proven and scalable.  
   - Cloud storage (S3 or equivalent) is widely used for receipt storage.

3. **Clear component boundaries and workflows**  
   - Services like Payments, Notifications, and Approval Engine are modular and communicate via API calls or queues.  
   - Asynchronous tasks (notifications, escalations) can be handled reliably using job queues (Redis + Bull / Celery).

4. **Data integrity and security can be ensured**  
   - DB transactions and optimistic locking prevent race conditions and maintain ledger correctness.  
   - Role-based access, JWT authentication, and encryption meet standard security practices.

5. **Scalable from day one**  
   - Start with a monolithic but modular architecture for rapid iteration.  
   - Can evolve into microservices as user growth demands.  
   - Managed services for DB and notifications reduce operational overhead.

6. **Business logic is implementable**  
   - Features like 51% approval for withdrawals, automated loan escalations, and receipt tracking are all software-driven and don’t rely on complex hardware or novel tech.  
   - The main challenge is careful orchestration of workflows, which can be reliably handled with queues, cron jobs, and API triggers.

> ✅ Overall, all components are based on mature technologies, well-documented APIs, and standard architectural patterns, making GroupWallet highly feasible to implement and scale.

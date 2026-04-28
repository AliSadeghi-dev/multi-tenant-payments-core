📘 Internal Multi-Project Payment Platform
System Design Document (Version 1.0)
1. Overview

This system is a centralized Payment Platform Service designed to manage all payment-related functionality across multiple independent client applications.

It supports:

One-time payments
Subscription billing
Multi-project architecture (multi-tenant)
Centralized plan/product management
Payment history tracking
Subscription lifecycle management
Integration with Stripe

The goal is to eliminate duplicated payment logic across projects and provide a reusable, scalable, and secure payment infrastructure.

2. Core Principles
2.1 Single Source of Truth

All financial and subscription states are stored in the Payment Service database.

2.2 Multi-Tenant Design

Each client application is treated as an isolated Project.

2.3 Backend-to-Backend Communication

Client applications never communicate directly with Stripe.

2.4 Event-Driven Sync

Stripe webhooks are used as the final authority for payment state updates.

3. System Architecture
3.1 High-Level Architecture

   [ Client Applications (Next.js / Web Apps) ]
                 |
                 v
     [ Client Backend (API Layer) ]
                 |
                 v
------------------------------------------------
|           Payment Service (Node.js)          |
|----------------------------------------------|
|  - Billing Logic                            |
|  - Subscription Management                  |
|  - Stripe Integration                      |
|  - Webhook Handler                         |
------------------------------------------------
                 |
                 v
         [ Stripe API Layer ]
                 |
                 v
------------------------------------------------
|             PostgreSQL Database             |
------------------------------------------------

                 ^
                 |
        [ React Admin Dashboard ]


3.2 Component Responsibilities
A. Client Applications
Display pricing UI
Trigger purchase actions
Request subscription status from backend
Enforce feature access based on subscription state
B. Client Backend (Mandatory Layer)

Acts as a secure proxy between frontend and Payment Service.

Responsibilities:

Authenticate users
Call Payment Service APIs
Apply business rules (feature gating)
Hide sensitive API keys
C. Payment Service (Core System)

Responsibilities:

Create checkout sessions
Manage products and pricing
Handle subscriptions
Process Stripe webhooks
Store all payment-related data
Provide APIs for clients and dashboard
D. Admin Dashboard (React)

Used for internal management:

Project creation
API key management
Product/plan creation
Subscription monitoring
Payment history analytics
E. Stripe

External payment provider handling:

Card processing
Billing cycles
Subscription lifecycle
Webhooks
F. Database (PostgreSQL)

Stores all business-critical data.

4. Data Model (Database Design)
4.1 Project

Represents each client application.

Project (
  id UUID PRIMARY KEY,
  name TEXT,
  apiKey TEXT UNIQUE,
  createdAt TIMESTAMP
)

4.2 Product (Plan)

Represents purchasable items.

Product (
  id UUID PRIMARY KEY,
  projectId UUID REFERENCES Project(id),
  name TEXT,
  type TEXT, -- "one_time" | "subscription"
  price INT,
  currency TEXT,
  interval TEXT NULL, -- monthly/yearly for subscriptions
  stripePriceId TEXT NULL,
  createdAt TIMESTAMP
)

4.3 Payment

Tracks all payment transactions.

Payment (
  id UUID PRIMARY KEY,
  projectId UUID,
  productId UUID,
  userId TEXT,
  amount INT,
  currency TEXT,
  status TEXT, -- pending | success | failed
  stripeSessionId TEXT UNIQUE,
  createdAt TIMESTAMP
)

4.4 Subscription

Tracks active recurring subscriptions.

Subscription (
  id UUID PRIMARY KEY,
  projectId UUID,
  productId UUID,
  userId TEXT,
  status TEXT, -- active | past_due | canceled
  currentPeriodEnd TIMESTAMP,
  stripeSubscriptionId TEXT UNIQUE,
  createdAt TIMESTAMP
)

5. Authentication & Authorization
5.1 API Key Authentication

Each project has a unique API key.

Used in:

Client → Payment Service communication

Header:
x-api-key: <project-api-key>

5.2 Admin Authentication

Dashboard uses:

JWT or external auth provider (e.g. Cognito)
5.3 Stripe Webhook Security

All webhook requests are verified using Stripe signature verification.

6. Payment Flow (One-Time & Subscription)
6.1 Step-by-Step Flow

1. User clicks "Buy"
2. Client Backend calls Payment Service
3. Payment Service validates API key
4. Product is fetched from DB
5. Stripe Checkout Session is created
6. Checkout URL is returned
7. User completes payment on Stripe
8. Stripe sends webhook event
9. Payment Service verifies webhook
10. Payment is stored in DB
11. Subscription (if applicable) is created/updated

6.2 Stripe Metadata Usage

Critical linking mechanism:

metadata:
{
  "projectId": "...",
  "productId": "...",
  "userId": "..."
}

7. Subscription Lifecycle Management

Handled via Stripe events:

Events:
checkout.session.completed
invoice.paid
invoice.payment_failed
customer.subscription.deleted

| Event                 | Action                |
| --------------------- | --------------------- |
| payment success       | activate subscription |
| payment failed        | mark as past_due      |
| subscription canceled | mark as canceled      |


8. Client Subscription Access Flow
8.1 Pull Model (MVP Recommended)
   Client App → Client Backend → Payment Service → Subscription Status

Used when:

user logs in
feature access is required
8.2 Push Model (Advanced)

Payment Service → Client Backend Webhook → Update local cache

Used for:

real-time updates
performance optimization
9. API Design (Payment Service)
9.1 Create Checkout Session

POST /checkout
Headers:
  x-api-key

Body:
{
  productId,
  userId
}

Response:
{
  "checkoutUrl": "https://stripe.com/..."
}

9.2 Webhook Endpoint
POST /webhook/stripe

Handles:

payment confirmation
subscription updates
failures
9.3 Subscription Fetch
GET /subscription/:userId

Returns:
{
  "status": "active",
  "plan": "Gold",
  "currentPeriodEnd": "..."
}

10. Security Considerations
10.1 No Direct Stripe Access from Clients

All Stripe interactions are isolated inside Payment Service.

10.2 API Key Isolation

Each project is fully isolated via API keys.

10.3 Idempotency Protection

Webhook events are deduplicated using:

stripe event id
stripeSessionId
10.4 Signature Verification

All Stripe webhooks are verified.

11. Error Handling Strategy
Retry-safe webhook processing
Graceful degradation for failed payments
Logging of all Stripe events
Idempotent database writes
12. Scalability Considerations
Stateless Payment Service
Horizontal scaling possible
DB indexing on:
userId
projectId
stripeSessionId
Webhook queue (optional future improvement)
13. Future Enhancements
Coupon / discount system
Usage-based billing
Stripe Connect (marketplace mode)
Analytics dashboard (MRR, churn, LTV)
Multi-currency support
Rate limiting per project

14. Final Summary

This system is designed as a:

Reusable, multi-tenant payment infrastructure acting as an internal Stripe-like service

It provides:

Centralized billing logic
Clean separation of concerns
Strong scalability foundation
Secure Stripe integration
Multi-project support


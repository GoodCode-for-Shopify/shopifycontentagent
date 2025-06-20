# Jules Authority Document: Shopify App Development

This document outlines the core technology stack, architectural decisions, and key directives for the ongoing Shopify app development project. Its purpose is to provide a centralized, concise reference to ensure consistency and alignment.

## 1. Core Technology Stack

The following technologies have been selected and are considered authoritative for this project:

*   **Frontend:**
    *   **Framework/Library:** React
    *   **UI Components:** Shopify Polaris (for UI embedded in Shopify Admin)
    *   **Communication with Backend:** Authenticated AJAX/fetch requests using Shopify App Bridge session tokens (JWTs).

*   **Backend:**
    *   **Runtime:** Node.js
    *   **Framework:** Express.js
    *   **Hosting Environment:** Cloud Functions for Firebase (2nd Generation preferred)
    *   **API Structure:** RESTful APIs

*   **Database:**
    *   **Type:** NoSQL Document Database
    *   **Service:** Firestore (via Firebase Admin SDK)
    *   **Data Model:** Multi-tenant, with data keyed by or associated with `shop_id`.

*   **Hosting:**
    *   **Frontend Hosting:** Firebase Hosting
    *   **Backend Hosting:** Cloud Functions for Firebase

*   **Authentication & Authorization:**
    *   **Shopify App to Backend:** Shopify session tokens (JWTs) obtained via App Bridge, verified by the backend.
    *   **Shopify Webhooks:** HMAC signature verification.
    *   **Admin Dashboard Users (if applicable, TBD):** To be determined, will align with Firebase capabilities (e.g., Firebase Authentication).

## 2. Key Directives & Architectural Choices

*   **Multi-Tenancy:** The application must support multiple Shopify stores (tenants) with strict data isolation enforced at the Firestore and backend logic levels, primarily using the `shop_id`.
*   **Credential Management (Third-Party APIs for Tenants):**
    *   **Storage:** Encrypted within Firestore, associated with the respective `shop_id`.
    *   **Encryption:** AES-256 encryption using Node.js `crypto` module on the backend.
    *   **Encryption Key Security:** The primary encryption key must be stored securely using Firebase environment configuration (`functions.config().secrets.encryption_key`) or Google Cloud Secret Manager. **Not to be hardcoded.**
    *   **OAuth Flows:** Backend to handle OAuth 2.0 flows for services like Google Ads, storing refresh tokens securely.
    *   Reference: `docs/jules/serverside-setup.md` for detailed implementation patterns.
*   **Development Priority (Admin Dashboard vs. Shopify App):**
    *   The Grok documents suggest building server-side functionality and an administrative dashboard *before* the end-user Shopify app. This directive is adopted. The admin dashboard will allow management of tenants, subscriptions (if applicable based on freemium model), and API credentials.
*   **Freemium Model & API Credential Options (as per Grok documents, adapted to our stack):**
    *   The app will support a freemium model (Free, Basic, Pro, Enterprise tiers).
    *   Tenants may have the option to use their own third-party API credentials or (for paid tiers, at a higher fee) use developer-provided/managed credentials for certain APIs (e.g., Google API Key, Gemini AI API Key).
    *   Google Ads will still require tenant-specific Refresh Tokens and Customer IDs due to its OAuth nature.
    *   Billing for subscriptions and differing credential usage options will be handled via Stripe.
*   **API Usage Tracking:**
    *   The system must track API usage for different tenants, especially if developer-provided credentials or usage quotas are involved. This will likely involve Firestore for logging usage counts against `shop_id` and API type.
*   **Modularity and Reusability:** Backend services built for the admin dashboard should be reusable for the end-user Shopify app where applicable.

## 3. Document Precedence

1.  **`docs/jules.authority.md` (this document):** For high-level technology choices and directives.
2.  **`docs/jules/serverside-setup.md`:** For detailed server-side setup and initial credential management implementation.
3.  **Newly generated `docs/jules/admin-dashboard/jules.*.md` documents:** For specific instructions on building the admin dashboard, adapted from Grok outlines.
4.  Grok-provided documents (`docs/grok/*`): Serve as the initial source/inspiration but are superseded by Jules-authored documents where adaptations for our tech stack are made.

This document is to be kept updated as major architectural decisions or directives evolve.

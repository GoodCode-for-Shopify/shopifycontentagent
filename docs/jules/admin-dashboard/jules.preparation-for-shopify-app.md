# Admin Dashboard: Preparation for End-User Shopify App

While the primary focus so far has been on building the server-side infrastructure and the Administrative Dashboard, these components are foundational for the eventual end-user Shopify application. This document outlines key considerations and preparatory steps for developing the Shopify app that tenants will install and use within their Shopify admin area.

## 1. Leveraging Existing Backend Infrastructure

(Derived from Grok Outline Section 9.1, adapted for our Firebase stack)

A significant advantage of building the admin dashboard and its backend first is that much of the server-side logic can be reused or extended for the end-user Shopify app.

*   **API Endpoints (Cloud Functions):**
    *   Many of the Express API routes created in Cloud Functions for managing credentials, handling Stripe subscriptions (if users manage their own subscriptions), and interacting with third-party APIs (Google Ads, Gemini AI) will be directly consumed by the end-user Shopify app.
    *   **Authentication:** The Shopify app frontend (React/Polaris embedded in Shopify admin) will authenticate its requests to your Cloud Functions backend using Shopify session tokens (JWTs obtained via App Bridge), as detailed in `docs/jules/serverside-setup.md` (Section: "Shopify App Backend Logic"). Ensure your backend has robust middleware for this.
    *   **Tenant Context:** All backend logic must operate strictly within the context of the authenticated `shop_id` derived from the Shopify session token.
*   **Database (Firestore):**
    *   The Firestore data structures defined for `shops`, `store_credentials`, `subscriptions`, and `api_usage_logs` will be the single source of truth, accessed by both the admin dashboard (via admin-authenticated APIs) and the Shopify app (via Shopify-app-authenticated APIs).
*   **Credential Management:**
    *   The secure storage, encryption, and decryption mechanisms for third-party API credentials (managed by Cloud Functions) will serve the Shopify app when it needs to make API calls on behalf of the tenant.

## 2. Shopify App Setup and Structure

*   **Shopify CLI:** Use the Shopify CLI to scaffold your new Shopify app project. This typically creates a Node.js backend (which you might use for Shopify-specific OAuth handling or webhook verification before passing logic to your main Firebase Cloud Functions) and a frontend structure.
    ```bash
    npm init @shopify/app@latest
    ```
    Choose Node.js and React (if prompted for frontend, though you might replace this with your more customized React/Polaris setup).
*   **Frontend (React/Polaris):**
    *   The Shopify app's user interface will be built with React and Shopify Polaris components to ensure it feels native within the Shopify admin.
    *   This UI will be embedded in an iframe in the Shopify admin.
    *   It will primarily interact with your existing Firebase Cloud Functions backend for data and operations.
*   **Firebase Hosting for Shopify App Frontend (Optional but Recommended):**
    *   You can host the static assets (HTML, CSS, JavaScript) of your Shopify app's frontend on Firebase Hosting, similar to the admin dashboard. This centralizes your hosting.
    *   The Shopify App setup in the Partner Dashboard will point to this Firebase Hosting URL.

## 3. Shopify OAuth and Access Token Management

(Derived from Grok Outline Section 9.2)

*   **App Installation Flow:**
    *   When a merchant installs your app, it must go through Shopify's OAuth 2.0 flow.
    *   Your backend (either a small Node.js app part of the Shopify CLI template or a dedicated Cloud Function) will handle:
        *   Redirecting the merchant to the Shopify authorization URL.
        *   Receiving the authorization `code` from Shopify after merchant approval.
        *   Exchanging the `code` for a Shopify access token (per-shop token).
*   **Storing Shopify Access Tokens:**
    *   The obtained Shopify access token for each shop **must** be stored securely.
    *   Encrypt this token using your standard encryption utility (Node.js `crypto` module with the key from Firebase config/Secret Manager) and store it in the corresponding `shops/{shop_id}` document in Firestore (e.g., in a field like `shopifyAccessToken: { iv: "...", encrypted_value: "..." }`).
    *   This token is essential for your backend to make authenticated calls to the Shopify Admin API on behalf of the merchant (e.g., to register webhooks, access shop data).
*   Refer to `docs/jules/serverside-setup.md` (Section: "Shopify App Backend Logic", subsection "Storing Shopify Access Token") for more details.

## 4. Implementing Freemium Features in the Shopify App

(Derived from Grok Outline Section 9.3)

The Shopify app's UI and functionality will need to adapt based on the tenant's current subscription plan.

*   **Plan-Based Feature Access:**
    *   The Shopify app's UI and functionality must dynamically adapt based on the tenant's current subscription plan and its specific, admin-configured features.
    *   **Fetching Dynamic Plan Configuration:** When a user interacts with the Shopify app, the frontend will make a request to your backend (Cloud Functions). This backend service will:
        1.  Identify the tenant's `shop_id`.
        2.  Retrieve the tenant's current `planId` from their `shops/{shop_id}` document in Firestore.
        3.  Fetch the detailed configuration for that `planId` from the `plans` collection in Firestore. This includes dynamically set attributes like `productProcessingLimit`, `keywordGenerationLimit`, `faqGeneration.enabled`, `faqGeneration.maxQuestions`, etc.
    *   **Controlling Frontend UI/UX:** The backend will return these specific plan feature values to the Shopify app frontend. The React/Polaris frontend will then use this information to:
        *   Enable or disable specific UI elements or entire sections.
        *   Adjust behavior, for example, limiting the number of items a user can select for processing based on `productProcessingLimit` (e.g., 1 for a basic dynamic plan, up to 5 for a pro dynamic plan).
        *   Display accurate information about usage limits or feature availability.
        *   Provide clear prompts to upgrade if a user attempts an action exceeding their current plan's dynamic limits.
    *   **Enforcing Limits on Backend:** Crucially, while the frontend adapts the UI, the backend Cloud Functions must also re-fetch and verify these dynamic plan limits before executing any resource-intensive or quota-limited operations (e.g., processing products, calling third-party AI services). This prevents users from bypassing frontend restrictions.
*   **Upgrade Prompts & Stripe Checkout:**
    *   Implement clear calls to action (CTAs) within the Shopify app for users to upgrade their plan.
    *   Clicking an "Upgrade" button should ideally initiate a Stripe Checkout session. This is typically done by:
        1.  Frontend calls a backend endpoint (e.g., `/api/subscriptions/create-checkout-session`).
        2.  Backend (Cloud Function) uses the Stripe SDK to create a Checkout session for the desired Stripe Price ID and the shop's Stripe Customer ID.
        3.  Backend returns the Checkout session ID to the frontend.
        4.  Frontend uses Stripe.js to redirect the user to Stripe Checkout.
        5.  Stripe webhooks handle the `checkout.session.completed` event to update the subscription status in Firestore.
*   **Credential Input and "Use Developer Credentials" Option:**
    *   The Shopify app's settings page will allow tenants to input their own API credentials (e.g., Google API Key, Gemini AI API Key) as per their plan's requirements.
    *   For paid plans offering the option to use developer-provided credentials (for certain APIs, at an extra cost):
        *   The UI will present a toggle or selection for this.
        *   Changing this setting will call a backend endpoint.
        *   The backend updates the `shops/{shop_id}.developerCredentialsOptIn` field in Firestore.
        *   If this change affects billing (i.e., moves them to a different Stripe Price ID associated with using developer keys), the backend must also update their Stripe subscription accordingly (e.g., by changing the subscribed price).

## 5. Design for Reusability
*   Strive to make backend Cloud Functions as generic as possible so they can serve both the admin dashboard (with admin-level authentication/authorization) and the end-user Shopify app (with Shopify session token authentication and tenant-specific logic).
*   Shared utility functions (encryption, API client wrappers) should be well-organized within the `functions/src/` directory.

By carefully planning these aspects, the development of the end-user Shopify app can build efficiently upon the robust backend and administrative systems already established.

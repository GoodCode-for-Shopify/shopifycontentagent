# Content Agent Shopify App: Authentication & Onboarding

This document details the authentication mechanisms for the "Content Agent" Shopify app and the user onboarding process, including terms acceptance, plan selection, Stripe integration for payments, and the initial API credential setup walkthrough.

## 1. Shopify App Authentication

The app, embedded within the Shopify admin, relies on Shopify to authenticate the merchant. Your app then needs to verify that requests to its backend are legitimate and originate from an authenticated Shopify session.

### 1.1. Shopify OAuth Flow (App Installation)
*   **Process:** As outlined in `docs/jules/serverside-setup.md` and `docs/jules/admin-dashboard/jules.preparation-for-shopify-app.md`, the initial app installation follows Shopify's OAuth 2.0 grant flow.
*   **Backend Handler:** A backend endpoint (typically part of the Shopify CLI's Node.js template, or a dedicated Cloud Function) handles the OAuth callback, exchanges the authorization code for a Shopify access token, and securely stores this encrypted token in Firestore against the `shop_id`.
*   **Scope:** Ensure your app requests the necessary scopes during installation (e.g., `read_products`, `write_products`, `read_content`, `write_content` for blog posts, potentially `read_themes` if interacting with theme templates/metafields for styling).

### 1.2. Authenticating Frontend Requests to Backend
*   **Shopify Session Tokens (JWTs):** The React/Polaris frontend, running embedded in the Shopify admin, uses Shopify App Bridge to obtain a session token (JWT).
    *   Install App Bridge if not already: `npm install @shopify/app-bridge @shopify/app-bridge-react` (in `web/frontend`).
    *   Wrap your React app in the App Bridge `Provider`.
*   **Sending Tokens:** Every API request from the React frontend to your Firebase Cloud Functions backend must include this session token in the `Authorization: Bearer <token>` header.
*   **Backend Verification:** Your Firebase Cloud Functions backend must have middleware to:
    1.  Verify the JWT signature using Shopify's JWKS (JSON Web Key Set).
    2.  Validate the token's claims (issuer `iss` should be `https://{shop_domain}/admin`, audience `aud` should be your Shopify App API Key).
    3.  Extract the shop domain (`dest` claim, which is your `shop_id`) to enforce tenant isolation.
    *   Refer to `docs/jules/serverside-setup.md` (Section: "Shopify App Backend Logic") for details on this JWT verification middleware.

## 2. User Onboarding Process

Once the app is installed and basic authentication is established, the user goes through an onboarding flow.

### 2.1. Terms Acceptance & User Data Collection
*   **UI (React/Polaris Page):**
    *   Display terms of service and privacy policy (links to full documents).
    *   Use a Polaris `Checkbox` for "I agree to the Terms of Service and Privacy Policy."
    *   Input fields for First Name, Last Name, and Email if not reliably obtainable or if a separate contact email is desired. Shopify often provides the store owner's email, which can be pre-filled.
*   **Backend Logic (Cloud Function):**
    *   Endpoint: e.g., `POST /api/onboarding/complete-profile`
    *   Receives: `firstName`, `lastName`, `email` (if manually entered), `termsAccepted: true`, `shop_id` (from verified session token).
    *   Shopify Data: The backend can use the shop's stored Shopify access token to fetch additional store details from the Shopify Admin API if needed (e.g., store name, primary contact email) to pre-populate or supplement.
    *   Storage: Update the `shops/{shop_id}` document in Firestore with this information (e.g., `contactName`, `contactEmail`, `termsAcceptedAt: timestamp`).
*   **Email Verification (Optional):**
    *   If the provided email needs verification (e.g., it's not the Shopify store owner's main email), trigger an email verification flow using Firebase Authentication's email verification or a custom solution with a transactional email service. This step might be skipped if relying on Shopify-provided emails.

### 2.2. Initial Plan Selection
*   **UI (React/Polaris Page):**
    *   Display available plans (Free, Basic, Pro, Enterprise features and pricing). This UI fetches the list of available plans (names, features, prices) by making an authenticated call to a backend endpoint such as `GET /api/plans-listing`.
    *   Clearly indicate the "default" plan (e.g., Free) and any "suggested" plan, based on flags set in the plan configuration by the admin team.
    *   Allow users to select a plan.
*   **Backend Logic (Cloud Function):**
    *   If Free plan is chosen: Update `shops/{shop_id}` in Firestore with `planId: "free"`. Proceed to credential walkthrough.
    *   If a Paid plan is chosen: Proceed to Stripe payment process.

### 2.3. Stripe Payment Process (for Paid Plans)
*   **UI (React/Polaris Component/Page):**
    *   After user selects a paid plan:
        *   The UI should display the selected plan's details and price.
        *   Include a Polaris `TextField` labeled "Discount Code" or "Promo Code" where the user can optionally enter a code.
    *   When the user clicks "Proceed to Payment" (or similar button):
        *   The React frontend captures the selected `planId` and the value from the discount code `TextField` (if any).
        *   It then calls a backend endpoint to initiate Stripe Checkout, including the discount code in the payload.
*   **Backend Logic (Cloud Function):**
    *   Endpoint: e.g., `POST /api/subscriptions/create-checkout-session`
    *   Receives: `{ planId: "pro_monthly_usd", discountCode: "USER_ENTERED_CODE" (optional), shop_id: "verified_shop_id" }`.
    *   Logic:
        1.  Retrieve the selected plan's details from Firestore, including its `stripePriceId`.
        2.  Retrieve or create a Stripe `Customer` for the `shop_id` (store `stripeCustomerId` in `shops/{shop_id}` in Firestore).
        3.  Prepare options for Stripe Checkout Session creation.
            *   If a `discountCode` is provided in the request and is valid (Stripe will validate this when `allow_promotion_codes` is enabled or if you pass specific coupon IDs), include it in the session creation parameters. Stripe Checkout can often handle `allow_promotion_codes=true` to let users enter codes on the Stripe-hosted page, or you can pass specific promotion codes directly. For passing a code from your UI to Stripe to apply, you'd typically use the `discounts` parameter with a `coupon` or `promotion_code` ID if your backend validates and fetches the ID first, or rely on `allow_promotion_codes`.
            *   For simplicity here, we'll assume the backend might pass a validated promotion code ID or use `allow_promotion_codes`. The example below shows `allow_promotion_codes`.
            ```javascript
            // const stripe = require('stripe')(functions.config().stripe.secret_key);
            // let checkoutOptions = {
            //   payment_method_types: ['card'],
            //   line_items: [{ price: stripePriceId, quantity: 1 }],
            //   mode: 'subscription',
            //   customer: stripeCustomerId,
            //   success_url: 'https://your-app-url/onboarding/payment-success?session_id={CHECKOUT_SESSION_ID}',
            //   cancel_url: 'https://your-app-url/onboarding/select-plan?payment_cancelled=true',
            //   metadata: { shop_id: shop_id, planId: planId } // Important for webhook
            // };
            //
            // // Option 1: If your UI collects the code and you want Stripe to apply it (simplest for UI)
            // if (discountCode) { // discountCode is the actual string like "SUMMER20"
            //   checkoutOptions.allow_promotion_codes = true;
            //   // Note: For Stripe to auto-apply a code passed this way, it's more complex.
            //   // Usually, you'd pass a specific promotion_code ID to the `discounts` array.
            //   // The example below assumes `allow_promotion_codes` is used, and user might re-enter on Stripe page,
            //   // OR your backend has logic to find a Promotion Code ID from `discountCode` string to pass to `discounts` array.
            //   // For a direct application of a known code string that maps to a Promotion Code ID:
            //   // if (validPromotionCodeId) { checkoutOptions.discounts = [{ promotion_code: validPromotionCodeId }]; }
            // }
            //
            // const session = await stripe.checkout.sessions.create(checkoutOptions);
            ```
             **Revised Backend Logic for `create-checkout-session` regarding `discountCode`:**
             The backend receives `discountCode` (string) from the frontend. It should attempt to find a corresponding active Promotion Code ID in Stripe. If found, it passes this ID to the `discounts` array in `stripe.checkout.sessions.create`. If not found, it could ignore it or return an error. Alternatively, set `allow_promotion_codes = true` on the Checkout session, and the user can enter the code on the Stripe page. The `allow_promotion_codes` approach is simpler if the user is okay re-entering or if the code is mainly for Stripe's UI.
             To directly apply the code from your UI:
             ```javascript
             // ...
             const checkoutSessionPayload = {
                 // ... other params ...
                 allow_promotion_codes: true, // Allows codes to be entered on Stripe's page
             };

             if (discountCode) { // discountCode is the string like "SUMMER10"
                // Optional: You could try to find a promotion code ID on Stripe based on the discountCode string
                // and add it to checkoutSessionPayload.discounts = [{promotion_code: 'promo_xxxx'}];
                // For simplicity, allow_promotion_codes=true is often sufficient.
                // If you want to ensure a specific code is applied, you'd need more logic here
                // to map the user-entered string to a Stripe Promotion Code ID.
             }
             const session = await stripe.checkout.sessions.create(checkoutSessionPayload);
             // ...
             ```
        4.  Return the `sessionId` from the Stripe Checkout Session to the frontend.
*   **Frontend Redirect to Stripe:**
    *   The React frontend uses Stripe.js to redirect the user to the Stripe Checkout page using the received `sessionId`.
    ```javascript
    // import { loadStripe } from '@stripe/stripe-js';
    // const stripe = await loadStripe('YOUR_STRIPE_PUBLISHABLE_KEY');
    // const { error } = await stripe.redirectToCheckout({ sessionId });
    ```
*   **Webhook Handling:** A separate Stripe webhook handler Cloud Function (e.g., `stripeWebhookHandler`) listens for `checkout.session.completed` and `customer.subscription.created/updated` events.
    *   Upon successful payment and subscription creation, the webhook updates the `shops/{shop_id}` document in Firestore with the `planId`, `stripeSubscriptionId`, and subscription `status: 'active'`.
*   **Post-Payment UI:**
    *   The `success_url` redirects the user back to a page in your app. This page should:
        *   Verify the status of the Checkout Session (by calling a backend endpoint that checks with Stripe using the session_id, or by relying on webhook to update Firestore and then checking Firestore).
        *   Confirm successful subscription.
        *   Proceed to credential walkthrough or app usage.

### 2.4. Credential Walkthrough (Multi-Step Form)
*   **UI (React/Polaris Multi-Step Form - e.g., using `Card` sections or `Layout`):**
    *   This is mandatory for Free plan users and for Paid plan users who opt to use their own credentials for certain services.
    *   Each step focuses on one API credential (e.g., Google Ads Refresh Token, Google API Key, Gemini AI API Key, WordPress credentials if chosen for export).
    *   **Content per step:**
        *   Clear text instructions on how to obtain the credential.
        *   Embedded instructional video (placeholder for where video URL would come from).
        *   Polaris `TextField` (or `Password` type where appropriate) for user to input the credential.
        *   "Save & Test Credential" button.
*   **Backend Logic (Cloud Function per credential type or a generic one):**
    *   Endpoint: e.g., `POST /api/credentials/save`
    *   Receives: `credentialType`, `value`, `shop_id` (from session).
    *   Logic:
        1.  **Test Credential:** Make a simple validation call to the respective third-party API using the provided credential (e.g., basic Google Ads API health check, simple Gemini AI query).
        2.  If test fails, return an error message to the UI.
        3.  If test succeeds:
            *   Encrypt the credential value using the Node.js `crypto` utility.
            *   Store it in Firestore under `shops/{shop_id}/credentials/{credentialType}`.
            *   Return success message to UI, allowing user to proceed to next step in the form.
*   **Bypass for Developer Credentials:** If a Paid plan uses developer-provided credentials for a service, that step in the walkthrough is skipped or shows an informational message.

Once onboarding (terms, plan, payment if any, credentials if any) is complete, the user is directed to the main part of the app (e.g., the "Product Selection" step for article generation).

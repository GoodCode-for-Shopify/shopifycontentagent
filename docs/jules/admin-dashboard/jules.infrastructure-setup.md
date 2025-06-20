# Admin Dashboard: Infrastructure Setup

This document details the setup of the necessary cloud infrastructure for the Shopify App's Administrative Dashboard, focusing on our chosen Firebase-centric stack. This includes database configuration (Firestore), other essential cloud services, and Stripe configuration for billing.

## 1. Database Configuration (Firestore)

(Derived from Grok Outline Section 2.1, adapted for Firestore)

Our primary database for storing all application data, including tenant information, subscriptions, credentials, and API usage logs, will be **Firestore**.

### 1.1. Enable Firestore
*   Ensure Firestore is enabled in your Firebase project, as outlined in `docs/jules/serverside-setup.md` (Section: "Firebase Project Setup").
*   Select **Production mode** for security rules.
*   Choose an appropriate **Firestore location (region)**. This cannot be changed later.

### 1.2. Data Schema Design for Admin Dashboard
While `docs/jules/serverside-setup.md` covers the basic `shops` collection for credential storage, the admin dashboard will require interaction with a slightly more comprehensive schema. All data will be organized with multi-tenancy in mind.

*   **`shops` Collection:**
    *   Document ID: `shop_id` (e.g., `your-store.myshopify.com`)
    *   Fields:
        *   `shopName`: (String) Store's display name.
        *   `installedAt`: (Timestamp) Date of app installation.
        *   `shopifyAccessToken`: (Map) `{ iv: "...", encrypted_value: "..." }` (for Shopify Admin API access, encrypted).
        *   `planId`: (String) Identifier for the current subscription plan (e.g., "free", "basic", "pro").
        *   `stripeCustomerId`: (String) Stripe Customer ID for this shop.
        *   `contactEmail`: (String, optional) Admin contact for the shop.
        *   `status`: (String) e.g., "active", "uninstalled", "frozen".
        *   `developerCredentialsOptIn`: (Map, optional) e.g., `{ gemini_ai: true, google_api_key: false }` indicating which developer-provided keys the tenant is using (and potentially being billed extra for).

*   **`plans` Collection:**
    *   Document ID: Auto-generated unique ID (or a human-readable slug like "pro-tier-q1-2024" if admins create these manually and slugs can be enforced as unique).
    *   Fields:
        *   `planName`: (String) User-friendly plan name (e.g., "Pro Tier", "Basic Monthly").
        *   `description`: (String, optional) Detailed description of the plan and its features.
        *   `price`: (Number) Monthly price in the smallest currency unit (e.g., cents for USD).
        *   `currency`: (String) ISO currency code (e.g., "USD", "EUR").
        *   `stripeProductId`: (String) Corresponding Product ID in Stripe (created via API).
        *   `stripePriceId`: (String) Corresponding Price ID in Stripe for the current price (created via API).
        *   `status`: (String) e.g., "active" (available for new subscriptions), "archived" (not available for new subscriptions, existing subscriptions may continue or be migrated), "draft".
        *   `isEditableByAdmin`: (Boolean) `true` if this plan was created by an admin and can be modified/archived; `false` for legacy or system-defined uneditable plans (if any).
        *   `createdAt`: (Timestamp) Creation date of the plan record.
        *   `updatedAt`: (Timestamp) Last modification date of the plan record.
        *   **`features` (Map):** Contains various configurable feature limits and flags for the plan.
            *   `productProcessingLimit`: (Number) Max number of products that can be processed simultaneously (e.g., 1 for Free/Basic, 5 for Pro).
            *   `keywordGenerationLimit`: (Number) Max number of keywords generated per product.
            *   `faqGeneration`: (Map)
                *   `enabled`: (Boolean) Whether FAQ generation is included.
                *   `maxQuestions`: (Number) Max number of FAQ questions generated per product.
            *   `apiCallLimit`: (Number, optional) Overall API call limit for a specific period for this plan, if applicable beyond individual feature limits.
            *   `usesDeveloperCredentials`: (Map, optional) e.g., `{ gemini_ai: true, google_api_key: false }` indicating if this plan inherently uses developer-provided keys for certain services. This might influence the Stripe Price ID selected or created.
            *   `customBooleanFeatureFlag1`: (Boolean, example) For future feature toggles.
            *   `customNumericFeatureValue1`: (Number, example) For future configurable numeric values.

*   **`subscriptions` Collection (if tracking more details than just `planId` on shop):**
    *   Document ID: Auto-generated or `shop_id` (if one subscription per shop).
    *   Fields:
        *   `shop_id`: (String) Foreign key to `shops` collection.
        *   `stripeSubscriptionId`: (String) Stripe Subscription ID.
        *   `stripePriceId`: (String) Stripe Price ID for the current subscription period.
        *   `planId`: (String) Current plan identifier.
        *   `status`: (String) e.g., "active", "past_due", "canceled".
        *   `startDate`: (Timestamp).
        *   `endDate`: (Timestamp, for canceled/trial subscriptions).
        *   `trialEndsAt`: (Timestamp, optional).
        *   `discountCodeApplied`: (String, optional) Stripe Coupon ID.

*   **`store_credentials` Subcollection within each `shops` Document:**
    *   Path: `shops/{shop_id}/credentials/{credential_type_slug}`
    *   This structure is detailed in `docs/jules/serverside-setup.md` (Section: "Database Design and Firestore Setup", Option B preferred for flexibility).
    *   Example `credential_type_slug`: `google_ads_refresh_token`, `google_api_key`, `gemini_ai_api_key`.
    *   Fields within each credential document:
        *   `iv`: (String) Initialization vector for encryption.
        *   `encrypted_value`: (String) Encrypted credential value.
        *   `updatedAt`: (Timestamp).
        *   `label`: (String, optional) User-friendly label if multiple credentials of the same type are allowed (e.g. "Main Google API Key").

*   **`api_usage_logs` Collection:**
    *   Document ID: Auto-generated.
    *   Fields:
        *   `shop_id`: (String) Identifies the tenant.
        *   `apiType`: (String) e.g., "google_ads", "gemini_ai".
        *   `timestamp`: (Timestamp) Time of the API call or log entry.
        *   `callCount`: (Number, optional, if batching logs) Number of calls made.
        *   `quotaUsed`: (Number, optional) How much quota this call consumed.
        *   `isDeveloperCredential`: (Boolean) Whether the call was made using developer-provided credentials.
        *   `endpoint`: (String, optional) Specific API endpoint called.
    *   Alternatively, monthly aggregated usage could be stored in the `shops` document or a dedicated `monthly_api_usage` collection to simplify quota checking.

### 1.3. Firestore Security Rules
*   Implement strict security rules for Firestore. Admin dashboard operations will primarily be performed by authenticated admin users (via Firebase Authentication).
*   Admin users should have broader access (read/write to most collections for management purposes).
*   Backend Cloud Functions using the Admin SDK will bypass these rules but should operate with the principle of least privilege programmatically.
*   Refer to `docs/jules/serverside-setup.md` for foundational rules, and extend them for admin access. Example for admin role:
    ```
    // Allow admin users (identified by a custom claim) to read/write all shop data
    match /shops/{shopId} {
      allow read, write: if request.auth.token.isAdmin == true;
      // Rules for subcollections like 'credentials' would inherit or have specific admin rules.
    }
    match /plans/{planId} {
      allow read: if true; // Plans might be public or readable by admins
      allow write: if request.auth.token.isAdmin == true; // Admins can manage plans
    }
    // Add rules for other collections like subscriptions, api_usage_logs
    ```

## 2. Other Cloud Services

(Derived from Grok Outline Section 2.2, adapted for Firebase/Google Cloud)

*   **Encryption Key Management:**
    *   As stated in `docs/jules.authority.md`, encryption keys for data at rest (e.g., in Firestore) will be managed using **Firebase environment configuration (`functions.config().secrets.encryption_key`) or Google Cloud Secret Manager**. This replaces AWS KMS from the Grok document.
*   **Caching & Rate Limiting Support:**
    *   The Grok document mentions Redis. For our Firebase stack:
        *   **Caching:** Firestore's persistence and offline capabilities, along with smart data structuring, can reduce the need for a separate caching layer for some use cases. For server-side caching of frequently accessed, slowly changing data, consider in-memory caching in Cloud Functions (with attention to instance lifecycle) or Firestore itself.
        *   **Rate Limiting:** For simple rate limiting (e.g., per `shop_id` per function), counters can be implemented in Firestore. For more complex needs, Google Cloud services like Cloud Memorystore (Redis) could be integrated, but the aim is to first leverage Firebase platform capabilities.
*   **Admin Dashboard Hosting:**
    *   The React/Polaris admin dashboard frontend will be hosted on **Firebase Hosting**. This replaces Vercel from the Grok document.
*   **Environment Variables:**
    *   All sensitive keys (Stripe keys, Google API client secrets for OAuth, our own encryption key if using `functions.config()`) will be stored as Firebase environment variables or in Google Cloud Secret Manager.

## 3. Stripe Configuration

(Derived from Grok Outline Section 2.3 - largely platform-agnostic)

Stripe will be used for handling all subscriptions, payments, and discount codes.

### 3.1. Stripe Account
*   Create or use an existing Stripe account.
*   Obtain your API keys:
    *   **Publishable Key:** For use in the frontend (e.g., with Stripe.js or React Stripe Elements) if doing client-side tokenization.
    *   **Secret Key:** For server-side API calls from your Cloud Functions. Store this securely in Firebase environment configuration.
    *   **Webhook Signing Secret:** To verify webhook events from Stripe. Store this securely.

### 3.2. Define Products and Prices (Price Tiers)
*   In the Stripe Dashboard, create Products for each of your service plans (e.g., "Basic Plan", "Pro Plan").
*   For each Product, create Prices corresponding to the different billing options:
    *   Free Plan: $0/month (Stripe might require a $0 price or just handle this without a Stripe subscription).
    *   Basic Plan: e.g., $29/month.
    *   Basic Plan (with Developer Credentials): e.g., $49/month (if this involves an extra charge reflected as a separate price).
    *   Pro Plan: e.g., $99/month.
    *   Pro Plan (with Developer Credentials): e.g., $149/month.
    *   Ensure to set recurring billing intervals (e.g., monthly).
    *   Note down the **Stripe Price IDs** for each of these; they will be used when creating subscriptions.

### 3.3. Discount Codes (Coupons)
*   In the Stripe Dashboard, create Coupons that your admin users or marketing team can distribute (e.g., "10PERCENTOFF" for 10% off for 3 months, "SAVE50" for a one-time $50 discount).
*   These coupon IDs can be applied when creating a subscription.

### 3.4. Webhooks
*   Configure Stripe webhooks to notify your backend (a specific HTTP Cloud Function) about important subscription events.
*   **Endpoint URL:** This will be an HTTP-triggered Cloud Function (e.g., `https://<region>-<project-id>.cloudfunctions.net/stripeWebhookHandler`).
*   **Events to listen for:**
    *   `customer.subscription.created`
    *   `customer.subscription.updated` (for plan changes, upgrades, downgrades, cancellations)
    *   `customer.subscription.deleted`
    *   `invoice.payment_succeeded` (to confirm payment and provision/continue service)
    *   `invoice.payment_failed` (to handle payment failures, potentially freeze accounts)
    *   `checkout.session.completed` (if using Stripe Checkout)
*   The webhook handler Cloud Function must:
    *   Verify the event signature using your Stripe Webhook Signing Secret.
    *   Process the event (e.g., update subscription status in Firestore, modify user plan).
    *   Return a `200 OK` response to Stripe quickly to acknowledge receipt.

This infrastructure setup provides a solid, Firebase-native foundation for your admin dashboard's backend services and billing integration.

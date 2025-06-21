# Technical Design Document: Admin Dashboard - Dynamic Plan Management & Stripe Integration

## 1. Introduction & Goals

### 1.1. Purpose of this Document
This Technical Design Document (TDD) outlines the architecture and flow for the Admin Dashboard feature that allows administrators to dynamically create, manage, and update subscription plans for the "Content Agent" Shopify app. It specifically details how these dynamic plans are integrated with Stripe for billing (Products and Prices) and how they are stored and utilized within the Firebase environment.

This TDD consolidates and expands upon plan management details previously mentioned in `jules.admin-dashboard.infrastructure-setup.md` and `jules.admin-dashboard.backend-development.md`.

### 1.2. The Dynamic Plan Challenge
The "Content Agent" app operates on a freemium model with multiple tiers (Free, Basic, Pro, Enterprise). To adapt to market needs and offer flexibility, administrators require the ability to:
*   Define and modify plan features, limits (e.g., `productProcessingLimit`, `keywordGenerationLimit`, `relatedQuestions.enabled`), and pricing without requiring new code deployments.
*   Have these changes automatically reflected in Stripe (for new subscriptions) and be available for the Shopify app to adjust user experience and feature access.
*   Manage the lifecycle of plans (e.g., creating new plans, updating existing ones, archiving old ones).

### 1.3. Key Design Goals
1.  **Dynamic Configuration:** Allow admins to define plan names, descriptions, features, limits, pricing (amount and currency), and billing intervals via the Admin Dashboard UI.
2.  **Stripe Product & Price Synchronization:** Ensure that for every active, billable plan, corresponding Product and Price objects are created and managed in Stripe.
3.  **Firestore Persistence:** Store plan configurations, including Stripe IDs, in Firestore for use by both the Admin Dashboard and the main Shopify app backend.
4.  **Plan Lifecycle Management:** Support creating, viewing, updating (including pricing changes), and archiving plans.
5.  **Clear Admin UX:** Provide an intuitive interface in the Admin Dashboard for managing these dynamic plans.
6.  **Robust Error Handling:** Implement error handling for Stripe API interactions and data consistency.
7.  **Impact on Existing Subscriptions:** Define how plan changes affect existing subscribers (typically, existing subscriptions continue on their original terms until modified by the subscriber or an admin intervention for special cases).

## 2. High-Level Admin Flow for Plan Management

1.  **Admin navigates** to "Plan Management" section in Admin Dashboard.
2.  **View Existing Plans:** A list of all plans (active, archived) is displayed, showing key details (Name, Price, Interval, Status, Stripe Product/Price IDs).
3.  **Create New Plan:**
    *   Admin clicks "Add New Plan."
    *   Fills a form with: Plan Name, Description, Tier (e.g., Basic, Pro), Price, Currency, Billing Interval (monthly/annually), Feature Flags (e.g., `relatedQuestions.enabled`), Usage Limits (e.g., `productProcessingLimit`, `keywordGenerationLimit`, `relatedQuestions.maxQuestionsToRetrieve`), `displayOrder`, `isPubliclyVisible`, `isDefaultPlan`.
    *   On submit, backend:
        *   Validates data.
        *   Creates a Stripe Product (if one for this conceptual plan doesn't exist) or uses existing.
        *   Creates a Stripe Price object associated with the Stripe Product.
        *   Saves plan details (including Stripe Product ID and Price ID) to Firestore.
4.  **Update Existing Plan:**
    *   Admin selects a plan and clicks "Edit."
    *   Modifies form fields.
    *   **If pricing changes:**
        *   Backend archives the old Stripe Price object (to prevent new subscriptions on this price).
        *   Backend creates a new Stripe Price object for the updated price.
        *   Firestore record is updated with the new Stripe Price ID and other plan details.
    *   **If only non-pricing details change:** Backend updates the plan details in Firestore. The Stripe Product might be updated if its description/name changed.
5.  **Archive Plan:**
    *   Admin selects a plan and clicks "Archive."
    *   Backend archives the Stripe Price object associated with the plan (prevents new subscriptions).
    *   Optionally, the Stripe Product can be archived if no other active Prices are using it.
    *   Plan status in Firestore is marked as "archived." The plan is no longer publicly listable for new subscriptions.
    *   Existing subscriptions on this plan continue until they naturally end or are actively changed.

## 3. Data Models

### 3.1. Firestore: `plans` Collection
*   Document ID: Auto-generated unique `planId`.
*   Fields:
    *   `name`: (String) e.g., "Pro Plan"
    *   `description`: (String) Marketing description.
    *   `tier`: (String) e.g., "pro" (used for programmatic checks if needed).
    *   `price`: (Number) e.g., 49.99
    *   `currency`: (String) e.g., "usd" (ISO currency code)
    *   `interval`: (String) e.g., "month", "year" (Stripe Price recurring interval)
    *   `stripeProductId`: (String) Stripe Product ID.
    *   `stripePriceId`: (String) Stripe Price ID for the current price.
    *   `features`: (Map)
        *   `productProcessingLimit`: (Number) Max products per generation job.
        *   `keywordGenerationLimit`: (Number) Max keywords to generate/validate.
        *   `relatedQuestions`: (Map)
            *   `enabled`: (Boolean)
            *   `maxQuestionsToRetrieve`: (Number)
        *   `aiImageGeneration`: (Map)
            *   `enabled`: (Boolean)
            *   `monthlyLimit`: (Number)
        *   `allowWordPressExport`: (Boolean)
        *   `allowCustomStyling`: (Boolean)
        *   `supportLevel`: (String) e.g., "standard", "priority"
    *   `status`: (String) e.g., "active", "archived"
    *   `isPubliclyVisible`: (Boolean) If true, show on Shopify app's plan selection page.
    *   `isDefaultPlan`: (Boolean) If true, this is the plan new free users are on (usually the "Free" plan).
    *   `displayOrder`: (Number) For ordering plans on selection pages.
    *   `createdAt`: (Timestamp)
    *   `lastUpdatedAt`: (Timestamp)
    *   `legacyPriceIds`: (Array of Strings, optional) Stores old Stripe Price IDs if pricing was changed, for historical reference.

### 3.2. Stripe: Products and Prices
*   **Stripe Product:** Represents the service plan itself (e.g., "Content Agent - Pro Plan").
    *   `id`: (e.g., `prod_XXXXXXXXXXXXXX`)
    *   `name`: (e.g., "Content Agent - Pro Plan")
    *   `description`: (Optional)
    *   `active`: (Boolean)
    *   `metadata`: (Should store `planId` from Firestore for cross-referencing, e.g., `{ firestorePlanId: 'planId_from_firestore' }`)
*   **Stripe Price:** Represents a specific price point for a Product (e.g., $49.99/month for the Pro Plan).
    *   `id`: (e.g., `price_YYYYYYYYYYYYYY`)
    *   `product`: (String - Stripe Product ID)
    *   `active`: (Boolean) - Setting to `false` archives the price.
    *   `unit_amount`: (Integer - price in cents, e.g., 4999)
    *   `currency`: (String - e.g., "usd")
    *   `recurring`: (Map - defines billing interval, e.g., `{ interval: "month", interval_count: 1 }`)
    *   `metadata`: (Should store `planId` from Firestore for cross-referencing, e.g., `{ firestorePlanId: 'planId_from_firestore' }`)

## 4. Admin Dashboard UI (Conceptual)

*   **Plan List View:**
    *   Table displaying columns: Name, Price, Interval, Currency, Status, Stripe Product ID, Stripe Price ID, Visible, Default.
    *   Actions: "Add New Plan," "Edit," "Archive."
*   **Plan Form (for Create/Edit):**
    *   Input fields for all `plans` collection fields described above.
    *   Polaris components: `TextField`, `Select` (for interval, currency, status), `Checkbox` (for booleans), `FormLayout`.
    *   Clear separation for "Pricing Details" vs. "Feature Limits."
    *   Validation messages.

## 5. Backend API Endpoints (Admin Dashboard Context)

Protected by admin authentication middleware (`adminAuthMiddleware`). All reside under `/api/admin/plans`.

*   **`POST /api/admin/plans` (Create Plan):**
    *   **Request Body:** Contains all necessary fields for a new plan (name, description, price, currency, interval, features, limits, etc.).
    *   **Logic:**
        1.  Validate input data.
        2.  Create a new Stripe Product. Store the Firestore `planId` in the Stripe Product's `metadata`.
        3.  Create a new Stripe Price with `unit_amount`, `currency`, `recurring: {interval, interval_count: 1}`, linked to the Stripe Product ID. Store the Firestore `planId` in the Stripe Price's `metadata`.
        4.  Save the new plan to Firestore `plans` collection, including the retrieved `stripeProductId` and `stripePriceId`.
    *   **Response:** The newly created plan object from Firestore.

*   **`GET /api/admin/plans` (List Plans):**
    *   **Logic:** Fetch all plans from Firestore `plans` collection.
    *   **Response:** Array of plan objects.

*   **`GET /api/admin/plans/:planId` (Get Specific Plan):**
    *   **Logic:** Fetch the plan with the given `planId` from Firestore.
    *   **Response:** Single plan object.

*   **`PUT /api/admin/plans/:planId` (Update Plan):**
    *   **Request Body:** Contains fields to be updated.
    *   **Logic:**
        1.  Validate input data.
        2.  Fetch the existing plan from Firestore.
        3.  **If price, currency, or interval is changed:**
            *   Archive the old Stripe Price object (`active: false`) using `stripe.prices.update(oldStripePriceId, {active: false})`. Add `oldStripePriceId` to `legacyPriceIds` in Firestore.
            *   Create a new Stripe Price object with the new details, linked to the same `stripeProductId`.
            *   Update the Firestore plan document with the new `stripePriceId` and other modified fields.
        4.  **If only non-pricing details (name, description, features, limits) are changed:**
            *   Update the Stripe Product's name/description if those changed using `stripe.products.update(stripeProductId, {name, description})`.
            *   Update the plan document in Firestore with the modified fields.
    *   **Response:** The updated plan object from Firestore.

*   **`DELETE /api/admin/plans/:planId` (Archive Plan):**
    *   **Logic:**
        1.  Fetch the plan from Firestore.
        2.  Archive its current Stripe Price: `stripe.prices.update(stripePriceId, {active: false})`.
        3.  (Optional: If no other active prices use the Stripe Product, it could also be archived: `stripe.products.update(stripeProductId, {active: false})`.)
        4.  Update the plan's `status` to "archived" in Firestore.
    *   **Response:** Success message.

## 6. Stripe Integration Details

*   **Stripe SDK:** Use the official `stripe` Node.js library in backend Cloud Functions.
*   **API Keys:** Store Stripe secret key securely (e.g., Firebase environment configuration or Google Secret Manager). Publishable key is used in frontend if Stripe Elements/Checkout is ever implemented in Admin (less likely for this flow).
*   **Error Handling:** Wrap all Stripe API calls in `try...catch` blocks. Handle Stripe-specific errors (e.g., `StripeCardError`, `StripeInvalidRequestError`) gracefully and return meaningful error messages to the admin UI.
*   **Idempotency:** For creation operations, consider using idempotency keys if needed, although simple create-and-store might be sufficient if UI prevents rapid duplicate submissions.
*   **Webhooks:** While Stripe webhooks are crucial for managing subscription lifecycle events initiated by users (e.g., `checkout.session.completed`, `customer.subscription.updated`), they are less critical for admin-initiated plan *definition* changes. However, logging Stripe events can be useful for auditing.

## 7. Impact on Existing Subscriptions

*   **General Rule:** Changes to a plan definition (e.g., price, features) in the Admin Dashboard and Stripe **do not automatically change existing active subscriptions** on that plan. Existing subscribers continue with the terms (Stripe Price and features) they signed up for.
*   **Price Changes:** When a plan's price is changed via the Admin Dashboard, a *new* Stripe Price object is created, and the old one is archived. New subscribers will use the new Price. Existing subscribers remain on the old Price.
    *   Migrating existing subscribers to a new price is a separate, complex process (often requiring consent) and is **out of scope for this V1 TDD**. It would involve using Stripe's subscription update APIs.
*   **Feature/Limit Changes:** When features or limits of a plan are changed in Firestore:
    *   The Shopify app backend, when checking a user's subscription against a plan, should ideally fetch the plan details using the `stripePriceId` stored in the user's `subscription` record in Firestore. This `stripePriceId` can then be used to look up the *specific version* of the plan details (features, limits) that were active when that Price was current. This requires storing historical plan versions or ensuring the current plan document accurately reflects what active subscriptions on a given `stripePriceId` should receive.
    *   **Simpler V1 Approach:** The Shopify app backend might just use the current plan details in Firestore based on the `planId` stored in the user's subscription. This means existing subscribers *would* see changes to features/limits if the plan they are on is updated. This is simpler to implement but might not always be desirable from a business perspective. **This TDD will assume this simpler V1 approach unless specified otherwise: Shopify app uses the current plan definition from Firestore.** Therefore, administrators should be aware that changes to features/limits for an active plan will affect existing subscribers on that plan, and such changes should be communicated appropriately. Communication to users about such changes would be essential.

## 8. Security Considerations

*   Admin Dashboard access is protected by Firebase Authentication and custom claims (`adminAuthMiddleware`).
*   Stripe API keys are stored securely.
*   Input validation is performed for all API requests.
*   Careful management of Stripe Product and Price objects is necessary to avoid billing errors. Archiving (setting `active: false`) is preferred over deletion for Prices associated with past or current subscriptions.

This TDD provides the framework for building a dynamic plan management system within the Admin Dashboard, enabling flexible control over the app's subscription offerings.

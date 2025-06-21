# Admin Dashboard: Backend Development

This document covers the backend development for the Shopify App's Administrative Dashboard using Cloud Functions for Firebase with Node.js/Express. It details project initialization, administrator authentication, API route creation, credential encryption, API usage tracking, and logging.

## 1. Backend Project Initialization (Cloud Functions)

(Derived from Grok Outline Section 3.1, adapted for Firebase Cloud Functions)

The backend logic will reside in Firebase Cloud Functions.

### 1.1. Initialize Firebase Functions
*   If not already done, initialize Firebase Functions in your project directory:
    ```bash
    firebase init functions
    ```
*   Choose **JavaScript** or **TypeScript** (examples here will use JavaScript).
*   Select ESLint for code linting.
*   Install dependencies with npm when prompted.
*   This creates a `functions` directory. Refer to `docs/jules/serverside-setup.md` (Section: "Backend Setup (Cloud Functions for Firebase)") for details on structure and initial Express app setup within `functions/src/index.js`.

### 1.2. Install Core Dependencies
Navigate to the `functions` directory and install essential packages:
```bash
cd functions
npm install express cors firebase-admin firebase-functions stripe
# firebase-admin and firebase-functions are likely already installed
# stripe is for Stripe API interaction
```
Other packages for specific API client libraries (e.g., Google APIs) will be added as needed.

## 2. Administrator Authentication

(Derived from Grok Outline Section 3.2, adapted for Firebase Authentication)

The admin dashboard needs to be accessible only to authorized administrators. We will use **Firebase Authentication** for this.

### 2.1. Enable Firebase Authentication
*   In the Firebase Console, navigate to "Authentication" and enable it.
*   Choose desired sign-in methods for administrators (e.g., **Email/Password**, **Google Sign-In**). Start with Email/Password for simplicity in creating initial admin users.

### 2.2. Create Admin Users
*   Manually create the first admin user(s) via the Firebase Console (Authentication > Users > Add user).
*   Note their UID.

### 2.3. Role-Based Access Control (RBAC) using Custom Claims
*   To distinguish admins (and potentially different admin roles like 'superadmin' or 'support'), use Firebase Auth custom claims.
*   Set custom claims for admin UIDs using the Firebase Admin SDK (this can be done via a secure, one-off script or a protected Cloud Function).
    ```javascript
    // Example: Utility function to set an admin claim (run this securely)
    // const admin = require('firebase-admin');
    // admin.auth().setCustomUserClaims(uid, { isAdmin: true })
    //   .then(() => { console.log(`Custom claim set for UID: ${uid}`); });
    ```

### 2.4. Authentication Middleware for Admin API Routes
Protect your Express API routes intended for admin dashboard consumption by verifying the Firebase ID token sent from the frontend and checking for the `isAdmin` custom claim.

```javascript
// functions/src/middleware/adminAuthMiddleware.js
const admin = require('firebase-admin');

async function adminAuthMiddleware(req, res, next) {
  const authorizationHeader = req.headers.authorization;
  if (!authorizationHeader || !authorizationHeader.startsWith('Bearer ')) {
    return res.status(403).send('Unauthorized: No token provided.');
  }

  const idToken = authorizationHeader.split('Bearer ')[1];
  try {
    const decodedToken = await admin.auth().verifyIdToken(idToken);
    if (decodedToken.isAdmin === true) {
      req.user = decodedToken; // Add user info to request object
      return next();
    } else {
      return res.status(403).send('Unauthorized: User is not an admin.');
    }
  } catch (error) {
    console.error('Error verifying Firebase ID token:', error);
    return res.status(403).send('Unauthorized: Invalid token.');
  }
}

module.exports = adminAuthMiddleware;
```
*   Apply this middleware to all admin-specific API routes in your Express app.

## 3. API Routes for Admin Dashboard

(Derived from Grok Outline Section 3.3, using Express within Cloud Functions)

Define RESTful API endpoints in your `functions/src/index.js` (or modularized route files) for the admin dashboard to interact with. All routes intended for admin use should be protected by the `adminAuthMiddleware`.

```javascript
// functions/src/index.js (simplified structure)
// ... (imports: functions, admin, express, cors, adminAuthMiddleware)
// admin.initializeApp(); // Ensure Firebase Admin is initialized

const app = express();
app.use(cors({ origin: true }));
app.use(express.json());

// --- Admin API Routes ---
const adminApi = express.Router();
adminApi.use(adminAuthMiddleware); // Protect all routes under /admin

// **Plan Management APIs**
// The Stripe SDK for Node.js (npm install stripe) would be initialized with your Stripe secret key.
// const stripe = require('stripe')(functions.config().stripe.secret_key);

adminApi.post('/plans', async (req, res) => {
  // Purpose: Create a new subscription plan.
  // Request Body: Contains all configurable plan attributes (planName, description, price, currency, feature configurations).
  try {
    // 1. Validate incoming plan data (ensure all required fields are present, types are correct, etc.)
    const { planName, description, price, currency, features } = req.body;
    if (!planName || !price || !currency || !features) {
      return res.status(400).send("Missing required plan attributes.");
    }

    // 2. Create a new Product in Stripe
    // const product = await stripe.products.create({ name: planName, description: description });
    const stripeProductId = "prod_mock_" + Date.now(); // Mock Stripe Product ID

    // 3. Create a new Price in Stripe associated with the Product ID
    // const stripePrice = await stripe.prices.create({
    //   unit_amount: price, // Price in smallest currency unit (e.g., cents)
    //   currency: currency,
    //   recurring: { interval: 'month' },
    //   product: product.id,
    // });
    const stripePriceId = "price_mock_" + Date.now(); // Mock Stripe Price ID

    // 4. Save the complete plan configuration to Firestore
    const newPlanRef = admin.firestore().collection('plans').doc();
    const newPlan = {
      id: newPlanRef.id,
      planName,
      description,
      price, // Store price in smallest currency unit
      currency,
      features, // e.g., { productProcessingLimit: 5, keywordGenerationLimit: 100, faqGeneration: { enabled: true, maxQuestions: 5 } }
      stripeProductId: stripeProductId,
      stripePriceId: stripePriceId,
      status: 'active', // Default status
      isEditableByAdmin: true,
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
      updatedAt: admin.firestore.FieldValue.serverTimestamp(),
    };
    await newPlanRef.set(newPlan);

    // 5. Return the newly created plan object
    res.status(201).json(newPlan);
  } catch (error) {
    console.error("Error creating plan:", error);
    // Handle Stripe API errors specifically if possible (e.g., error.type === 'StripeCardError')
    res.status(500).send("Failed to create plan.");
  }
});

adminApi.get('/plans', async (req, res) => {
  // Purpose: Retrieve a list of all subscription plans.
  try {
    const snapshot = await admin.firestore().collection('plans').orderBy('createdAt', 'desc').get();
    const plans = snapshot.docs.map(doc => doc.data());
    res.status(200).json(plans);
  } catch (error) {
    console.error("Error fetching plans:", error);
    res.status(500).send("Failed to fetch plans.");
  }
});

adminApi.get('/plans/:planId', async (req, res) => {
  // Purpose: Retrieve details of a specific subscription plan.
  try {
    const { planId } = req.params;
    const planRef = admin.firestore().collection('plans').doc(planId);
    const doc = await planRef.get();
    if (!doc.exists) {
      return res.status(404).send("Plan not found.");
    }
    res.status(200).json(doc.data());
  } catch (error) {
    console.error("Error fetching plan:", error);
    res.status(500).send("Failed to fetch plan.");
  }
});

adminApi.put('/plans/:planId', async (req, res) => {
  // Purpose: Update an existing subscription plan.
  // Request Body: Contains the plan attributes to be updated.
  try {
    const { planId } = req.params;
    const updates = req.body; // e.g., { description, price, currency, features, status }

    // 1. Validate incoming data
    if (Object.keys(updates).length === 0) {
      return res.status(400).send("No update data provided.");
    }

    const planRef = admin.firestore().collection('plans').doc(planId);
    const planDoc = await planRef.get();
    if (!planDoc.exists) {
      return res.status(404).send("Plan not found.");
    }
    const existingPlan = planDoc.data();

    // 2. Stripe Price Handling
    let newStripePriceId = existingPlan.stripePriceId;
    if (updates.price && (updates.price !== existingPlan.price || (updates.currency && updates.currency !== existingPlan.currency))) {
      // Price or currency changed, archive old Stripe price and create a new one.
      // await stripe.prices.update(existingPlan.stripePriceId, { active: false });
      // const newStripePrice = await stripe.prices.create({
      //   unit_amount: updates.price || existingPlan.price,
      //   currency: updates.currency || existingPlan.currency,
      //   recurring: { interval: 'month' },
      //   product: existingPlan.stripeProductId,
      // });
      // newStripePriceId = newStripePrice.id;
      newStripePriceId = "price_mock_updated_" + Date.now(); // Mock updated Stripe Price ID
      console.log(`Mock: Archived old price ${existingPlan.stripePriceId}, created new price ${newStripePriceId}`);
    }

    // Update Stripe Product if name/description changes (optional, Stripe Product can often be generic)
    // if (updates.planName || updates.description) {
    //   await stripe.products.update(existingPlan.stripeProductId, {
    //     name: updates.planName || existingPlan.planName,
    //     description: updates.description || existingPlan.description,
    //   });
    // }

    // 3. Update the plan document in Firestore
    const finalUpdates = {
      ...updates,
      stripePriceId: newStripePriceId, // Use new price ID if changed
      updatedAt: admin.firestore.FieldValue.serverTimestamp(),
    };
    await planRef.update(finalUpdates);

    // Note: Implications for existing subscribers are not handled here. Updates apply to new subscriptions.
    res.status(200).json({ id: planId, ...finalUpdates });
  } catch (error) {
    console.error("Error updating plan:", error);
    res.status(500).send("Failed to update plan.");
  }
});

adminApi.delete('/plans/:planId', async (req, res) => {
  // Purpose: "Archive" or effectively delete a plan.
  try {
    const { planId } = req.params;
    const planRef = admin.firestore().collection('plans').doc(planId);
    const planDoc = await planRef.get();

    if (!planDoc.exists) {
      return res.status(404).send("Plan not found.");
    }
    const planData = planDoc.data();

    // 1. Mark plan as "archived" in Firestore
    await planRef.update({
      status: 'archived',
      updatedAt: admin.firestore.FieldValue.serverTimestamp()
    });

    // 2. Archive the corresponding Price in Stripe
    // await stripe.prices.update(planData.stripePriceId, { active: false });
    console.log(`Mock: Archived Stripe Price ${planData.stripePriceId}`);

    // Optionally, archive Stripe Product if no other active prices exist for it.
    // This requires checking other plans or having a more complex logic.
    // For now, we just archive the price.

    res.status(200).send({ message: `Plan ${planId} archived successfully.` });
  } catch (error) {
    console.error("Error archiving plan:", error);
    res.status(500).send("Failed to archive plan.");
  }
});

// **Public API Endpoint for Plan Listing**
// While admin users manage plans via the `/admin/plans` endpoints, the end-user Shopify app will need to display available plans during onboarding. A separate, read-only endpoint, such as `GET /api/plans-listing`, should be created. This endpoint would:
//     *   Be authenticated using the standard Shopify app session token (not admin authentication).
//     *   Fetch active (non-archived) plans from the `plans` collection in Firestore.
//     *   Return a subset of plan data suitable for public display (e.g., plan name, description, price, currency, feature list like product processing limit, keyword limit, related questions enabled/count). It should not expose sensitive details like Stripe Product/Price IDs if not needed by the frontend client directly for display.

// Example: Tenant (Shop) Management
adminApi.get('/shops', async (req, res) => {
  try {
    const snapshot = await admin.firestore().collection('shops').get();
    const shops = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
    res.status(200).json(shops);
  } catch (error) {
    console.error("Error fetching shops:", error);
    res.status(500).send("Failed to fetch shops.");
  }
});

adminApi.get('/shops/:shop_id', async (req, res) => { /* ... get single shop ... */ });
adminApi.put('/shops/:shop_id', async (req, res) => { /* ... update shop details (e.g., plan, status) ... */ });
// Avoid POST for creating shops here if shop creation is tied to Shopify app install.
// Admins might "activate" or "manage" existing shops.

// Subscription Management (interacting with Stripe & Firestore)
// Note: The following endpoint is for admins to manage existing subscriptions.
// The creation of a checkout session for a new subscription is typically initiated by the Shopify app user.
adminApi.post('/shops/:shop_id/subscriptions', async (req, res) => {
  // const { shop_id } = req.params;
  // const { stripePriceId, couponId } = req.body; // couponId here would be a Stripe Coupon ID
  // Logic to create/update Stripe subscription and store details in Firestore
  // Ensure you have stripeCustomerId for the shop from Firestore.
});
adminApi.get('/shops/:shop_id/subscriptions', async (req, res) => { /* ... get subscription details ... */ });


// --- Shopify App User API Routes (Example for Checkout Session) ---
// This route would be called by the Shopify app frontend during onboarding/plan selection.
// It should be protected by Shopify session token middleware, not adminAuthMiddleware.
// For simplicity, Shopify session token middleware is not shown here but is assumed.
app.post('/api/subscriptions/create-checkout-session', async (req, res) => {
  // Purpose: Create a Stripe Checkout session for a new subscription.
  // Request Body: { planId: "pro_plan_id", shop_id: "verified_shop_id", discount_code: "SUMMER20" (optional string) }
  // `shop_id` would come from verified Shopify session token.
  try {
    const { planId, discount_code, shop_id } = req.body;

    if (!planId || !shop_id) {
      return res.status(400).send("Missing required parameters: planId and shop_id.");
    }

    // 1. Fetch plan details from Firestore to get stripePriceId
    const planRef = admin.firestore().collection('plans').doc(planId);
    const planDoc = await planRef.get();
    if (!planDoc.exists || planDoc.data().status !== 'active') {
      return res.status(404).send("Active plan not found or plan is not active.");
    }
    const stripePriceId = planDoc.data().stripePriceId;

    // 2. Retrieve or create Stripe Customer ID for the shop_id
    // (Logic to get stripeCustomerId from `shops/{shop_id}` in Firestore)
    const shopDoc = await admin.firestore().collection('shops').doc(shop_id).get();
    if (!shopDoc.exists) {
        return res.status(404).send("Shop not found.");
    }
    let stripeCustomerId = shopDoc.data().stripeCustomerId;
    if (!stripeCustomerId) {
        // const customer = await stripe.customers.create({
        //   email: shopDoc.data().contactEmail, // Or Shopify store email
        //   name: shopDoc.data().shopName,
        //   metadata: { shop_id: shop_id }
        // });
        // stripeCustomerId = customer.id;
        // await admin.firestore().collection('shops').doc(shop_id).update({ stripeCustomerId });
        stripeCustomerId = "cus_mock_" + Date.now(); // Mock customer ID
        console.log("Mock: Created Stripe customer " + stripeCustomerId);
    }

    // 3. Create Stripe Checkout Session
    let checkoutSessionParams = {
      payment_method_types: ['card'],
      line_items: [{ price: stripePriceId, quantity: 1 }],
      mode: 'subscription',
      customer: stripeCustomerId,
      success_url: `https://YOUR_APP_URL/onboarding/payment-success?session_id={CHECKOUT_SESSION_ID}`, // Replace with actual success URL
      cancel_url: `https://YOUR_APP_URL/onboarding/select-plan?payment_cancelled=true`,    // Replace with actual cancel URL
      metadata: { shop_id: shop_id, planId: planId },
    };

    // Handle discount code
    if (discount_code) {
      // Recommended V1 approach: Let Stripe's Checkout page handle promotion code validation.
      checkoutSessionParams.allow_promotion_codes = true;
      // If you wanted to apply a specific coupon/promotion code ID directly (after validating the user-entered code
      // and fetching its ID from Stripe API):
      // const promotionCodes = await stripe.promotionCodes.list({ code: discount_code, active: true, limit: 1 });
      // if (promotionCodes.data.length > 0) {
      //    checkoutSessionParams.discounts = [{ promotion_code: promotionCodes.data[0].id }];
      // } else {
      //    // Optionally handle invalid code from user here, or let Stripe handle it if allow_promotion_codes = true
      //    console.warn(`User entered discount code "${discount_code}" not found or inactive in Stripe.`);
      // }
    }

    // const session = await stripe.checkout.sessions.create(checkoutSessionParams);
    const mockSessionId = "cs_test_mock_" + Date.now(); // Mock session ID

    res.status(200).json({ sessionId: mockSessionId /* session.id */ });

  } catch (error) {
    console.error("Error creating Stripe Checkout session:", error);
    // If Stripe API call fails due to invalid coupon/promotion code in `discounts` array
    if (error.type === 'StripeInvalidRequestError' && error.code === 'resource_missing' && error.param === 'discounts[0][promotion_code]') {
        return res.status(400).json({
            error: {
                code: "INVALID_DISCOUNT_CODE",
                message: "The provided discount code is invalid or expired."
            }
        });
    }
    // Generic error based on your global error handling strategy
    res.status(500).send("Failed to create Stripe Checkout session.");
  }
});


// Credential Management (viewing status/type, not exposing encrypted values directly)
adminApi.get('/shops/:shop_id/credentials', async (req, res) => {
  // Fetch credential types and existence/update dates, not the encrypted values themselves.
  // Admins might need to trigger re-OAuth for a user or delete stale credentials.
});
// Storing/updating credentials is typically done by the Shopify app user,
// but admins might need to manage/override in exceptional cases (with extreme caution and auditing).

// API Usage Tracking (for admin viewing)
adminApi.get('/shops/:shop_id/usage', async (req, res) => { /* ... fetch usage data from Firestore ... */ });
adminApi.get('/usage_summary', async (req, res) => { /* ... fetch aggregated usage data ... */ });


app.use('/admin', adminApi); // Mount admin routes under /admin prefix

exports.api = functions.https.onRequest(app); // Expose main Express app
// Potentially, you might have a dedicated 'adminApi' export if you want a separate function URL for admin operations.
// exports.adminApi = functions.https.onRequest(app); // if app only contains admin routes
```

## 4. Credential Encryption

(Derived from Grok Outline Section 3.4, using Node.js `crypto` as per `docs/jules.authority.md`)

Tenant-provided third-party API credentials must be stored encrypted in Firestore.
*   Utilize the encryption/decryption utility functions (`encrypt`, `decrypt`) using Node.js `crypto` module as detailed in `docs/jules/serverside-setup.md` (Section: "Credential Management - Server-Side", subsection "Encryption and Decryption Utilities").
*   The encryption key must be securely stored in Firebase environment configuration or Google Cloud Secret Manager.
*   When admins manage credentials (e.g., initiating a re-OAuth for a user, or if a system allows admins to input credentials on behalf of a user - use with caution), ensure these functions are called.
*   **Admin Interface Note:** The admin dashboard should generally *not* display decrypted credentials. It might show the *status* of credentials (e.g., "Google Ads Token: Present", "Gemini Key: Not Set") or allow actions like "Request Re-authentication".

## 5. API Usage Tracking

(Derived from Grok Outline Section 3.5, adapted for Firestore)

The system needs to track API usage per tenant, especially for enforcing quotas under different plans or billing for usage of developer-provided keys.

### 5.1. Data Model for Usage
*   Use the `api_usage_logs` collection in Firestore as designed in `docs/jules/admin-dashboard/jules.infrastructure-setup.md`.
*   Alternatively, for simpler quota checking, maintain aggregated counters (e.g., `monthlyGeminiAICalls`) directly within each `shops/{shop_id}` document or a related `shop_usage_summary` document.

### 5.2. Logging Usage
*   Within your Cloud Functions that make third-party API calls on behalf of tenants:
    *   After a successful API call (or batch of calls), log the usage.
    *   Increment relevant counters in Firestore.
    ```javascript
    // Example: logging a single Gemini AI call
    // async function logApiCall(shopId, apiType, calls = 1, isDeveloperCredential = false) {
    //   const usageRef = admin.firestore().collection('api_usage_logs').doc();
    //   await usageRef.set({
    //     shop_id: shopId,
    //     apiType: apiType,
    //     timestamp: admin.firestore.FieldValue.serverTimestamp(),
    //     callCount: calls,
    //     isDeveloperCredential: isDeveloperCredential
    //   });
    //   // Also update aggregated counters if necessary
    // }
    ```

### 5.3. Enforcing Quotas
*   Before making a third-party API call, check the tenant's current usage against their plan's quota.
*   This check would involve reading the aggregated usage data from Firestore for the current billing period.
*   If quota is exceeded, the API call should be denied, and the user potentially notified or prompted to upgrade.

## 6. Logging (Application & Audit Logging)

(Derived from Grok Outline Section 3.6, using Firebase/Google Cloud Logging)

*   **Application Logging:**
    *   Use Firebase Functions built-in logging for debugging and monitoring function execution:
        ```javascript
        const functions = require('firebase-functions');
        functions.logger.info("This is an info log", { structuredData: true });
        functions.logger.error("This is an error log", errorObject);
        ```
    *   These logs are automatically sent to Google Cloud Logging.
*   **Audit Logging (for Admin Actions):**
    *   For significant actions performed by administrators via the admin dashboard (e.g., changing a tenant's plan, deleting credentials, modifying user permissions), create specific audit logs.
    *   These can be stored in a dedicated Firestore collection (e.g., `admin_audit_logs`) or logged to Google Cloud Logging with specific structured data indicating the admin user (from `req.user.uid`) and the action performed.
    ```javascript
    // Example: logging an admin action to Firestore
    // async function logAdminAction(adminUid, action, targetShopId, details = {}) {
    //   await admin.firestore().collection('admin_audit_logs').add({
    //     adminUid: adminUid,
    //     action: action, // e.g., "changed_plan", "deleted_credential"
    //     targetShopId: targetShopId,
    //     timestamp: admin.firestore.FieldValue.serverTimestamp(),
    //     details: details
    //   });
    // }
    ```

This backend setup provides the core C.R.U.D. (Create, Read, Update, Delete) operations and business logic necessary for the admin dashboard to function effectively and securely within our Firebase architecture.

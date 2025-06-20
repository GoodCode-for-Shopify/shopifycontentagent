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
adminApi.post('/shops/:shop_id/subscriptions', async (req, res) => {
  // const { shop_id } = req.params;
  // const { stripePriceId, couponId } = req.body;
  // Logic to create/update Stripe subscription and store details in Firestore
  // Ensure you have stripeCustomerId for the shop from Firestore.
});
adminApi.get('/shops/:shop_id/subscriptions', async (req, res) => { /* ... get subscription details ... */ });

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

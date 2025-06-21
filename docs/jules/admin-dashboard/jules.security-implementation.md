# Admin Dashboard: Security Implementation

Security is a critical aspect of the Admin Dashboard, especially as it handles sensitive tenant data, API credentials, and administrative functions. This document outlines key security measures, aligning with `docs/jules.authority.md` and `docs/jules/serverside-setup.md`.

## 1. Secure Credential Storage and Handling

(Derived from Grok Outline Section 6.1, adapted for Firebase stack)

The principles for handling tenant-provided third-party API credentials are covered extensively in `docs/jules/serverside-setup.md` and apply here as well.

*   **Encryption at Rest:**
    *   All sensitive credentials (e.g., tenant's Google Ads refresh tokens, tenant-provided AI API keys, Shopify access tokens for each shop) stored in Firestore **must** be encrypted using AES-256.
    *   The Node.js `crypto` module is used for encryption/decryption in Cloud Functions, as detailed in `docs/jules/serverside-setup.md`.
*   **Encryption Key Management:**
    *   The primary symmetric encryption key used by the `crypto` module **must** be stored securely, either in Firebase environment configuration (`functions.config().secrets.encryption_key`) or Google Cloud Secret Manager.
    *   **Never hardcode this key in the source code.**
    *   Access to this key (via IAM for Secret Manager or Firebase config permissions) should be restricted to only the Cloud Functions that perform encryption/decryption.
*   **Admin Access to Credentials:**
    *   The admin dashboard UI should **never** display decrypted sensitive credentials.
    *   Admins may see the *status* of a credential (e.g., "Set," "Not Set," "Requires Re-authentication") or metadata like update timestamps.
    *   Actions like initiating a re-OAuth flow for a tenant are performed by the backend; the admin UI only triggers this action. Deleting credentials should also be a backend operation, carefully logged.

## 2. API Route Protection (Backend Cloud Functions)

(Derived from Grok Outline Section 6.2, adapted for Firebase)

All backend API endpoints (Express routes within Cloud Functions) serving the admin dashboard must be rigorously protected.

*   **Authentication:**
    *   Utilize the `adminAuthMiddleware` (detailed in `docs/jules/admin-dashboard/jules.backend-development.md`) for all API routes intended for admin dashboard consumption.
    *   This middleware verifies the Firebase ID token sent from the React admin frontend and checks for the `isAdmin` custom claim.
*   **Authorization (Role-Based Access Control):**
    *   If different levels of admin access are required (e.g., 'superadmin' vs. 'support'), this can be implemented by assigning more granular custom claims via Firebase Authentication (e.g., `role: 'superadmin'`, `role: 'support'`).
    *   The `adminAuthMiddleware` or specific route handlers would then check for these specific roles.
*   **Input Validation:**
    *   All incoming data from the admin dashboard to API endpoints (request bodies, query parameters, URL parameters) must be validated on the server-side.
    *   Use libraries like `express-validator` or manual checks to ensure data types, formats, lengths, and ranges are as expected before processing. This helps prevent injection attacks and unexpected errors.
*   **CORS (Cross-Origin Resource Sharing):**
    *   The Cloud Functions backend must be configured with a specific list of allowed origins to prevent unauthorized cross-origin requests. This list should accommodate all frontend environments and Shopify domains.
    *   **Allowed Origins should include:**
        *   Production Admin Dashboard: `https://shopifycontentagent.com` (and/or `https://admin.shopifycontentagent.com`).
        *   Staging Admin Dashboard: `https://staging.shopifycontentagent.com` (and/or `https://admin-staging.shopifycontentagent.com`).
        *   Development Admin Dashboard: `https://dev.shopifycontentagent.com` (and/or `https://admin-dev.shopifycontentagent.com`), and development localhost URLs (e.g., `http://localhost:3000`, `http://localhost:3001`).
        *   Production Shopify App Frontend: e.g., `https://app.shopifycontentagent.com` (or the root `https://shopifycontentagent.com` if served from a path).
        *   Staging Shopify App Frontend: e.g., `https://app-staging.shopifycontentagent.com`.
        *   Development Shopify App Frontend: e.g., `https://app-dev.shopifycontentagent.com`, and `localhost` variants for ngrok/Shopify CLI tunnels.
        *   Shopify Admin Domains: `https://*.myshopify.com` and `https://admin.shopify.com` (crucial for embedded app functionality).
    *   **Conceptual `cors` middleware options in Express:**
        ```javascript
        // In functions/src/index.js (or a dedicated corsConfig.js)
        // const cors = require('cors');

        // Define base domains from environment variables for flexibility
        // const PROD_ADMIN_DOMAIN = process.env.PROD_ADMIN_DOMAIN || 'https://shopifycontentagent.com';
        // const STAGING_ADMIN_DOMAIN = process.env.STAGING_ADMIN_DOMAIN || 'https://staging.shopifycontentagent.com';
        // const DEV_ADMIN_DOMAIN = process.env.DEV_ADMIN_DOMAIN || 'https://dev.shopifycontentagent.com';
        // const PROD_APP_DOMAIN = process.env.PROD_APP_DOMAIN || 'https://app.shopifycontentagent.com';
        // ... and so on for other staging/dev app domains

        // const allowedOrigins = [
        //   PROD_ADMIN_DOMAIN,
        //   STAGING_ADMIN_DOMAIN,
        //   DEV_ADMIN_DOMAIN,
        //   PROD_APP_DOMAIN,
        //   // Add other staging/dev app domains and specific localhost ports used in development:
        //   'http://localhost:3000', // Common React dev port for Admin UI
        //   'http://localhost:3001', // Common React dev port for Shopify App UI (example)
        //   // RegExp for Shopify admin domains:
        //   /^https:\/\/[a-zA-Z0-9-]+\.myshopify\.com$/,
        //   /^https:\/\/admin\.shopify\.com$/
        // ];

        // app.use(cors({
        //   origin: function (origin, callback) {
        //     // Allow requests with no origin (like mobile apps or curl requests)
        //     if (!origin) return callback(null, true);

        //     if (allowedOrigins.some(o => typeof o === 'string' ? o === origin : o.test(origin))) {
        //       callback(null, true);
        //     } else {
        //       console.warn(`CORS: Origin ${origin} not allowed.`);
        //       callback(new Error('Not allowed by CORS'));
        //     }
        //   },
        //   methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'], // Specify allowed methods
        //   allowedHeaders: ['Authorization', 'Content-Type'], // Specify allowed headers
        //   credentials: true // If you need to handle cookies or authorization headers
        // }));
        ```
    *   **Management:** The list of allowed origins should ideally be managed via environment configuration (e.g., Firebase function config, Google Secret Manager) to allow flexibility between deployed environments (dev, staging, prod) without code changes, especially if the exact domain list varies. However, for a core set of domains as listed, they can be part of the code if they are stable. The RegExp for Shopify domains is generally stable.
*   **HTTPS:**
    *   All communication between the admin dashboard frontend and backend Cloud Functions will be over HTTPS by default (provided by Firebase Hosting and Cloud Functions).

## 3. Firestore Security Rules

While backend Cloud Functions using the Admin SDK bypass Firestore security rules, well-defined rules provide defense in depth.
*   **Default Deny:** Start with a default deny-all rule.
    ```
    rules_version = '2';
    service cloud.firestore {
      match /databases/{database}/documents {
        match /{document=**} {
          allow read, write: if false;
        }
      }
    }
    ```
*   **Admin Access:** Grant specific access to admin users based on their Firebase Auth UID and `isAdmin` custom claim. This is particularly relevant if any part of the admin dashboard attempts to read data directly from Firestore (though most data access should be via backend API calls).
    ```
    // Example: Allow admin to read all shop documents
    // match /shops/{shopId} {
    //   allow read: if request.auth.token.isAdmin == true;
    //   // Writes should still ideally go through backend for validation & logging
    //   allow write: if false; // Or allow write if request.auth.token.isAdmin == true for specific admin direct write scenarios
    // }
    ```
*   **Backend-Centric Operations:** For collections primarily managed by the backend (e.g., `store_credentials` where encryption occurs), rules can remain highly restrictive, as the Admin SDK handles privileged access. The primary security lies in the backend code's logic and authentication/authorization checks.

## 4. Compliance and Data Protection

(Derived from Grok Outline Section 6.3)

*   **GDPR/CCPA:**
    *   Be mindful of data privacy regulations. Understand what data constitutes PII (Personally Identifiable Information).
    *   Implement mechanisms for data export and deletion if requested by tenants (via the admin dashboard or user-facing app).
    *   Ensure your privacy policy is clear about data handling.
*   **Data Minimization:**
    *   Collect and store only the data that is absolutely necessary for the app's functionality.
    *   Regularly review stored data and purge unnecessary information.
*   **Secure Backups:**
    *   Firestore provides automatic backups. Understand the backup and restore capabilities offered by Google Cloud for Firestore.
*   **Audit Logs:**
    *   Maintain audit logs for critical admin actions (e.g., plan changes, credential deletions, user permission changes) as detailed in `docs/jules/admin-dashboard/jules.backend-development.md`. This is crucial for security analysis and accountability.

## 5. Other Security Considerations

*   **Dependency Security:** Regularly update Node.js packages in both `functions/` and `admin-dashboard-ui/` directories using `npm audit` and `npm update` to patch known vulnerabilities.
*   **Least Privilege for Service Accounts:** Review IAM permissions for the service account used by your Cloud Functions. By default, it has broad permissions. If feasible, create a custom service account with only the necessary permissions for interacting with Firestore, Secret Manager, etc.
*   **Firebase Project Security:**
    *   Secure access to your Firebase project console. Use strong, unique passwords and enable Multi-Factor Authentication (MFA) for all accounts with project access.
    *   Limit the number of users with owner/editor roles in your Firebase/Google Cloud project.

By implementing these security measures diligently, you can build a robust and trustworthy admin dashboard for your Shopify application. Security is an ongoing process, so regular reviews and updates are essential.

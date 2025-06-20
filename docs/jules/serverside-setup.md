# Server-Side and Database Setup for a Multi-Tenant Shopify App

This document provides comprehensive, step-by-step instructions for setting up the server-side infrastructure and database for a multi-tenant Shopify app. The app will feature individual credential management by users within their Shopify store admin areas.

We will be using the following technologies:

*   **Firestore:** A NoSQL document database for storing tenant-specific credentials and other application data.
*   **Cloud Functions for Firebase (2nd Gen):** For hosting our Node.js/Express backend logic in a serverless environment.
*   **Node.js/Express:** As the backend framework for creating APIs to manage credentials and interact with third-party services.
*   **Firebase Hosting:** For deploying the React/Polaris UI (frontend) of the Shopify app.
*   **React/Polaris:** For building the user interface that will be embedded within the Shopify admin area.

The goal is to create a secure, efficient, and scalable backend system that can handle multiple tenants (Shopify stores) and securely manage their API credentials for various third-party services.

## Prerequisites

Before you begin setting up the server-side infrastructure, please ensure you have the following accounts, tools, and command-line interface (CLI) utilities installed and configured:

1.  **Google Cloud Project:**
    *   A Google Cloud Platform (GCP) project is required to use Firebase services. If you don't have one, create one at the [Google Cloud Console](https://console.cloud.google.com/).
    *   Ensure billing is enabled for your GCP project.

2.  **Firebase Project:**
    *   Create a Firebase project and link it to your Google Cloud project. You can do this from the [Firebase Console](https://console.firebase.google.com/).

3.  **Node.js and npm:**
    *   Node.js (version 16 or later is recommended for Firebase Functions 2nd gen) and npm (Node Package Manager) must be installed. You can download them from [nodejs.org](https://nodejs.org/).
    *   Verify installation by running `node -v` and `npm -v` in your terminal.

4.  **Firebase CLI:**
    *   Install the Firebase Command Line Interface (CLI) globally:
        ```bash
        npm install -g firebase-tools
        ```
    *   Log in to Firebase using the CLI:
        ```bash
        firebase login
        ```
    *   Test the CLI by running `firebase --version`.

5.  **Google Cloud CLI (gcloud CLI):**
    *   Install the Google Cloud CLI for managing GCP resources and for some advanced Firebase configurations. Follow the installation instructions at [Google Cloud SDK documentation](https://cloud.google.com/sdk/docs/install).
    *   Initialize the gcloud CLI:
        ```bash
        gcloud init
        ```
    *   Log in with your Google account:
        ```bash
        gcloud auth login
        ```

6.  **Shopify Partner Account and Development Store:**
    *   A Shopify Partner account is needed to create and manage Shopify apps.
    *   A development store is essential for testing your app in a Shopify environment.

7.  **Basic Understanding:**
    *   Familiarity with JavaScript/Node.js, Express.js, REST APIs, and basic Firebase concepts will be beneficial.
    *   Understanding of how Shopify apps work (OAuth, embedded apps) is also helpful.

## Firebase Project Setup

This section guides you through setting up your Firebase project, including Firestore and Firebase Hosting.

1.  **Create a Firebase Project:**
    *   Go to the [Firebase Console](https://console.firebase.google.com/).
    *   Click on "Add project" or "Create a project."
    *   Enter your desired project name.
    *   Either link it to an existing Google Cloud project or let Firebase create a new one for you. It's generally recommended to manage it under a GCP project you control.
    *   Accept the terms and click "Create project."

2.  **Enable Firestore:**
    *   Once your project is created, navigate to the "Build" section in the left sidebar and click on "Firestore Database."
    *   Click "Create database."
    *   **Choose a mode:**
        *   **Production mode:** Start with locked down rules. This is recommended.
        *   **Test mode:** Start with open rules (allows all reads/writes for 30 days). Useful for initial development but be sure to secure it before launch.
    *   **Select a Firestore location:** Choose a region where your data will be stored. This cannot be changed later. Select a region close to your users or other services you might use (e.g., your Cloud Functions region).
    *   Click "Enable."
    *   **Initial Security Rules (Development):**
        After Firestore is enabled, go to the "Rules" tab. For initial development, if you chose test mode, it might look like this:
        ```
        rules_version = '2';
        service cloud.firestore {
          match /databases/{database}/documents {
            match /{document=**} {
              allow read, write: if request.time < timestamp.date(2024, 12, 31); // Or some future date
            }
          }
        }
        ```
        If you chose production mode, it will be more restrictive:
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
        **Important:** These are basic rules. You will need to define more granular rules later to secure your multi-tenant data (e.g., only allowing authenticated users or your backend server to access specific data). We will cover this in the "Database Design and Firestore Setup" section.

3.  **Enable Firebase Hosting:**
    *   In the Firebase Console, navigate to the "Build" section and click on "Hosting."
    *   Click "Get started."
    *   Follow the on-screen instructions. You'll primarily be using the Firebase CLI to deploy your hosting content (your React/Polaris UI), but enabling it here prepares your project.

4.  **Upgrade to Blaze Plan (Pay-as-you-go):**
    *   Cloud Functions for Firebase (2nd Gen) and some other Firebase features (like making outbound network requests from functions on the free "Spark" plan) require your Firebase project to be on the "Blaze" (pay-as-you-go) plan.
    *   In the Firebase Console, click on "Upgrade" or "Modify plan" near the bottom of the left sidebar.
    *   Select the Blaze plan and link your Google Cloud billing account. While it's a pay-as-you-go plan, Firebase offers a generous free tier for many services, so you might not incur costs for small projects.

## Backend Setup (Cloud Functions for Firebase)

We'll use Cloud Functions for Firebase to create our Node.js/Express backend. This provides a serverless environment that scales automatically. We'll be using 2nd generation functions, which offer better performance and features.

1.  **Initialize Firebase Functions in Your Project:**
    *   Open your terminal and navigate to your project's root directory (where you want to store your Firebase project configuration and code).
    *   If you haven't already, initialize Firebase in your project:
        ```bash
        firebase init
        ```
        *   Select "Functions" (and "Hosting" if you plan to manage its configuration from the same project directory, which is common).
        *   Choose "Use an existing project" and select the Firebase project you created earlier.
    *   If you've already initialized Firebase for other features (like Hosting), you can add Functions specifically:
        ```bash
        firebase init functions
        ```

2.  **Configure Functions:**
    *   **Language:** Choose **JavaScript** or **TypeScript** (this guide will use JavaScript for examples, but TypeScript is recommended for larger projects due to its strong typing).
    *   **Linter/Formatter:** Decide whether to use ESLint to catch probable bugs and enforce style. (Recommended: Yes).
    *   **Dependencies:** The CLI will ask if you want to install dependencies with npm now. (Recommended: Yes).

    This will create a `functions` directory in your project root with the following structure:
    ```
    project-root/
    ├── functions/
    │   ├── node_modules/
    │   ├── src/
    │   │   └── index.js  // Your main functions file
    │   ├── .eslintrc.js  // ESLint configuration (if selected)
    │   └── package.json  // Node.js dependencies and scripts
    ├── .firebaserc       // Firebase project configuration
    ├── firebase.json     // Firebase deployment rules and settings
    └── ...               // Other project files (e.g., for Hosting)
    ```
    If you chose TypeScript, `index.js` would be `index.ts` inside a `src` folder, and you'd have a `tsconfig.json`.

3.  **Modify `firebase.json` for 2nd Gen Functions (if needed):**
    Open `firebase.json` in your project root. Ensure it's configured for Node.js 16+ and specifies 2nd gen functions if not already set by `firebase init`.
    ```json
    {
      "functions": {
        "source": "functions",
        "runtime": "nodejs18" // Or nodejs16, nodejs20. Choose an LTS version.
      },
      "hosting": {
        // ... your hosting config
      }
      // ... other configurations
    }
    ```
    *Note: For 2nd gen functions, much of the configuration (like region, memory) is done directly in the code rather than `firebase.json`.*

4.  **Install Necessary NPM Packages:**
    *   Navigate to the `functions` directory:
        ```bash
        cd functions
        ```
    *   Install `express` for creating API endpoints and `cors` for handling cross-origin requests (especially if your frontend and backend are on different domains during local development or if you plan to allow other domains to access your API). Also install `firebase-admin` for interacting with Firebase services (like Firestore) from your backend and `firebase-functions` for defining Cloud Functions.
        ```bash
        npm install express cors firebase-admin firebase-functions
        ```
    *   Your `functions/package.json` should now include these dependencies.

5.  **Structure the Express App within a Cloud Function:**
    *   Open `functions/src/index.js` (or `functions/index.js` if no `src` folder was created).
    *   Replace its content with the following to set up a basic Express app that can be served via a single HTTP Cloud Function:

        ```javascript
        const functions = require("firebase-functions");
        const admin = require("firebase-admin");
        const express = require("express");
        const cors = require("cors");

        // Initialize Firebase Admin SDK
        // Make sure to do this only once per application lifecycle.
        if (admin.apps.length === 0) {
          admin.initializeApp();
        }

        const app = express();

        // Middleware
        app.use(cors({ origin: true })); // Enable CORS for all routes. For production, configure specific origins.
        app.use(express.json()); // To parse JSON request bodies

        // Define a simple test route
        app.get("/hello", (req, res) => {
          res.status(200).send("Hello from your Express API on Cloud Functions!");
        });

        // Placeholder for Shopify-specific routes (we'll add these later)
        const shopifyRouter = express.Router();
        shopifyRouter.get("/test-shopify", (req, res) => {
          // Here you would typically verify Shopify requests
          // For now, just a simple response
          res.status(200).send("Shopify API endpoint test successful!");
        });
        app.use("/api/shopify", shopifyRouter); // Prefix all shopify routes with /api/shopify

        // Placeholder for credential management routes
        const credentialsRouter = express.Router();
        // We will define POST and GET endpoints for credentials here later
        app.use("/api/credentials", credentialsRouter);

        // Expose the Express app as an HTTP Cloud Function
        // "api" will be the name of the function, and the URL will be something like:
        // https://<region>-<project-id>.cloudfunctions.net/api
        // Or if using Firebase Hosting rewrites: https://<your-domain>.com/api
        exports.api = functions.https.onRequest(app);

        // Example of a 2nd Gen HTTP function (alternative to wrapping the whole Express app)
        // exports.helloWorld2ndGen = functions.v2.https.onRequest((request, response) => {
        //   functions.logger.info("Hello logs!", {structuredData: true});
        //   response.send("Hello from Firebase 2nd Gen!");
        // });
        ```
    *   **Note on `admin.initializeApp()`:** This initializes the Firebase Admin SDK, allowing your functions to interact with Firebase services like Firestore with admin privileges. It uses Application Default Credentials when deployed to Firebase, so no explicit credential file is needed in the deployed environment.

6.  **Environment Variables for Firebase Functions:**
    *   For API keys that are hardcoded (app-wide, not per-tenant, as discussed in `Grok_multi-tenant-Shopify-app-credentials-flow.md`), you should use Firebase environment variables.
    *   **Do NOT hardcode secrets directly in your `index.js` file.**
    *   To set environment variables, use the Firebase CLI:
        ```bash
        firebase functions:config:set someservice.key="YOUR_API_KEY" someservice.id="YOUR_ID"
        ```
        For example, for the Google Ads Developer Token (if you manage it centrally):
        ```bash
        firebase functions:config:set googleads.developer_token="YOUR_DEVELOPER_TOKEN"
        ```
    *   In your `index.js`, you can access these like this:
        ```javascript
        // const developerToken = functions.config().googleads.developer_token;
        ```
    *   After setting config variables, you need to deploy your functions for the changes to take effect. You can view active configuration with `firebase functions:config:get`.
    *   For more sensitive keys, consider Google Cloud Secret Manager, which can be accessed from Cloud Functions. Firebase Functions also has built-in support for secrets.

7.  **Local Emulation (Optional but Recommended):**
    *   The Firebase Local Emulator Suite allows you to test your functions (including HTTP functions, Firestore triggers, etc.) and even your web app locally.
    *   Install the Emulator Suite:
        ```bash
        firebase init emulators
        ```
        Select "Functions", "Firestore", and "Hosting" emulators. Configure the ports if needed.
    *   Start the emulators:
        ```bash
        firebase emulators:start
        ```
    *   This will provide local URLs for your functions and Firestore instance, allowing for rapid development and testing without deploying.

This setup provides a basic Express server running within a Cloud Function. In the next sections, we'll expand on this by adding specific routes for credential management and Shopify interactions.

## Database Design and Firestore Setup

We'll use Firestore to store tenant-specific data, primarily the API credentials for third-party services. Each Shopify store (tenant) will have its credentials stored securely and isolated.

1.  **Data Structure for `store_credentials`:**
    *   As outlined in the `Grok_multi-tenant-Shopify-app-credentials-flow.md` document, we need a way to store credentials associated with each Shopify store.
    *   We can create a top-level collection in Firestore, for example, named `shops` or `tenants`. Each document in this collection will represent a Shopify store, keyed by the unique `shop_id` (e.g., `your-store-name.myshopify.com`).
    *   Within each shop's document, we can store various configuration details and a subcollection for credentials, or store credentials as an array of maps/objects if the number of credential types per shop is limited and known. Using a subcollection (`credentials`) might be more flexible if you anticipate many credential types or need to query them independently.

    **Option A: Credentials as a map within the shop document (Simpler for a few known credential types):**
    ```
    shops/{shop_id}/
        ├── shopInfo: { ... } // General info about the shop
        └── credentials: {
                google_ads_refresh_token: {
                    iv: "some_iv_string_for_encryption",
                    encrypted_value: "encrypted_refresh_token_value"
                },
                google_ads_customer_id: "123-456-7890", // Not always sensitive, but can be stored here
                gemini_ai_api_key: {
                    iv: "another_iv_string",
                    encrypted_value: "encrypted_gemini_key_value"
                }
                // ... other credentials
            }
    ```

    **Option B: Credentials as a subcollection (More scalable for many credential types):**
    ```
    shops/{shop_id}/
        ├── shopInfo: { ... }
        └── credentials/{credential_type_slug}/  // e.g., 'google_ads_refresh_token'
                ├── iv: "some_iv_string_for_encryption"
                └── encrypted_value: "encrypted_refresh_token_value"
    ```
    For this guide, we'll lean towards **Option A** for simplicity in the examples, assuming a manageable set of credentials per shop. If you have many, Option B is better. The `credential_type` from the original document would be the key in the `credentials` map.

    **Example Firestore Document Structure for a Shop (using `shop_id` as document ID):**
    Collection: `shops`
    Document ID: `cool-gadgets.myshopify.com`
    ```json
    {
      "shopName": "Cool Gadgets Inc.",
      "installedAt": "2023-10-26T10:00:00Z",
      "shopifyAccessToken": { // Store Shopify access token securely (encrypted)
        "iv": "shopify_token_iv",
        "encrypted_value": "encrypted_shopify_access_token"
      },
      "credentials": {
        "google_ads_refresh_token": {
          "iv": "hex_encoded_iv_for_google_ads_token",
          "encrypted_value": "hex_encoded_encrypted_google_ads_token"
        },
        "google_ads_customer_id": "123-456-7890", // Often not secret, but good to keep with related credentials
        "gemini_ai_api_key": { // Example if user provides their own Gemini key
          "iv": "hex_encoded_iv_for_gemini_key",
          "encrypted_value": "hex_encoded_encrypted_gemini_key"
        }
        // Add other credential types here as needed
      }
    }
    ```
    *   `shop_id`: The unique identifier for the Shopify store (e.g., `shop.myshopify.com`). This will be used as the document ID in the `shops` collection for easy lookup.
    *   `iv`: The initialization vector used for encryption (must be stored alongside the encrypted value).
    *   `encrypted_value`: The encrypted credential.

2.  **Firestore Security Rules for Multi-Tenancy:**
    *   Security rules are crucial for protecting tenant data. You need to ensure that a tenant can only access their own data and that your backend functions have the necessary permissions.
    *   Navigate to Firestore Database > Rules tab in the Firebase Console.

    **Example Security Rules:**
    These rules assume your backend functions authenticate using a service account (which they do by default when using `admin.initializeApp()`) and that you might also want to allow logged-in Shopify users (identified by their `shop_id` if you pass it as a custom claim in a Firebase Auth token, for example) to read their own non-sensitive configuration. For writing credentials, it's often best to restrict this to backend operations only.

    ```
    rules_version = '2';
    service cloud.firestore {
      match /databases/{database}/documents {

        // Shops collection:
        // Each document is keyed by shop_id (e.g., store-name.myshopify.com)
        match /shops/{shopId} {
          // Allow read if the request is from an authenticated user whose shop_id matches,
          // OR if the request is from your backend (service account - admin access).
          // `request.auth.token.shop_id` would come from a custom claim if you use Firebase Auth for users.
          // For backend-only access to credentials, you might simplify this.
          allow read: if request.auth != null && request.auth.token.shop_id == shopId ||
                         isBackendRequest(); // A custom function to identify backend requests (see below)

          // Generally, writes to credentials should only be done by your backend service.
          // This ensures proper encryption and validation.
          allow write: if isBackendRequest();

          // If you have subcollections like 'shop_config_items' that users can manage:
          // match /shop_config_items/{itemId} {
          //   allow read, write: if request.auth != null && request.auth.token.shop_id == shopId;
          // }
        }

        // Helper function to identify requests from your backend (service accounts)
        // This is a simplified example. In practice, service account requests often bypass
        // these rules if using the Admin SDK, but explicit rules can be good for clarity
        // or when using client libraries with specific auth.
        // A common pattern is to check if auth is null and you trust your functions' environment.
        // Or, if your functions act on behalf of a user but with elevated privileges,
        // you might check for a specific custom claim set by your backend.
        // For simplicity here, we'll assume Admin SDK bypasses these for writes from backend.
        // For reads, if your function needs to check user-specific access based on rules, it would need
        // to pass the user's auth context or use a more specific rule.

        // A more robust `isBackendRequest` might involve checking for a specific UID
        // if your functions authenticate with a dedicated service account user, or rely on the fact
        // that Admin SDK calls are typically privileged.
        // For this context, we'll assume Admin SDK calls have appropriate access.
        // For client-side initiated operations (e.g., user viewing their own non-sensitive config),
        // the `request.auth.token.shop_id == shopId` is key.

        // Fallback rule: deny all other access
        match /{document=**} {
          allow read, write: if false;
        }
      }
    }

    // Note on `isBackendRequest()`:
    // True backend requests using the Firebase Admin SDK (like in your Cloud Functions)
    // typically bypass Firestore security rules by default, as they operate with admin-level privileges.
    // The rules above are more for scenarios where you might have client-side SDK access
    // (e.g., your React app directly talking to Firestore, which is NOT what we are doing for credentials)
    // or for defining very explicit server-side validation logic if not relying on Admin SDK bypass.

    // For our scenario (backend handles all credential R/W):
    // The primary concern is that the Cloud Function for a given request correctly identifies the `shopId`
    // (e.g., from a verified Shopify session/JWT) and only operates on that `shopId`'s document.
    // The Firestore rules then act as a secondary defense.
    // A simpler rule set focusing on backend-only write access for credentials:
    /*
    rules_version = '2';
    service cloud.firestore {
      match /databases/{database}/documents {
        match /shops/{shopId} {
          // Allow backend (Admin SDK) to read/write.
          // For reads from a client (e.g. Shopify admin UI via your backend proxying data):
          // The backend function would be responsible for ensuring the user is authorized for shopId.
          // If the client (React app) were to read Firestore directly (not recommended for this use case):
          // allow read: if request.auth != null && request.auth.token.shop_id == shopId;

          // For our backend-centric model:
          allow read, write: if get(/databases/$(database)/documents/users/$(request.auth.uid)).data.isAdmin == true; // Example: if using Firebase Auth and an admin custom claim
                                // OR, if your backend is the only writer, and it's trusted:
                                // allow write: if true; (assuming function code is secure and only writes to correct shopId)
                                // However, this is too open.
                                // A common practice for backend-only is to rely on IAM permissions for the function's service account
                                // and keep rules restrictive, e.g., `allow read, write: if false;` and let Admin SDK bypass.
                                // Or, use a rule that can only be satisfied by a token your backend mints with a specific claim.

          // A practical approach:
          // Assume functions use Admin SDK, which bypasses these rules for its operations.
          // If you *also* have client-side SDK access for other non-sensitive parts of shop data:
          // allow read: if request.auth != null && request.auth.token.shop_id == shopId; // User can read their own shop's document
          // allow write: if false; // Disallow client-side writes to the main shop document directly
        }

        // Default deny
        match /{document=**} {
          allow read, write: if false;
        }
      }
    }
    */
    **Key Principle for Security Rules with Backend:** Your backend Cloud Functions, using the Firebase Admin SDK, will have privileged access to Firestore, effectively bypassing user-based security rules for their operations. The crucial part is ensuring your *backend code* correctly validates the `shop_id` from the incoming Shopify request (e.g., from a verified session token) and only performs operations for that specific `shop_id`. The Firestore rules then serve as an additional layer, especially if you ever consider direct client access for some data parts (which is not the case for writing credentials here).

    For the `store_credentials` which are highly sensitive and managed exclusively by the backend:
    1.  Your backend function receives a request (e.g., save API key).
    2.  It authenticates the request (e.g., verifies Shopify HMAC or JWT) to get the legitimate `shop_id`.
    3.  It then uses `admin.firestore().collection('shops').doc(shop_id).set(...)` to write the data. This operation, via Admin SDK, has full access.
    Security rules can be set to `allow read, write: if false;` for the credentials part if *only* Admin SDK access is intended, providing a strong default deny.

3.  **Firestore Indexes:**
    *   Firestore automatically creates indexes for single fields.
    *   If you need to query by multiple fields in the same document (e.g., `shop_id` and `credential_type` if you used a flat collection of credentials), you might need to define composite indexes.
    *   For our primary structure (`shops/{shop_id}/credentials/...`), direct lookups by `shop_id` are efficient. If you later add queries like "find all shops that have a 'google_ads_refresh_token'", you might need to adjust your data model or add specific fields to enable such queries (e.g., a boolean flag `has_google_ads` at the shop document level).
    *   The Firebase console will provide links to create missing indexes if your queries fail due to their absence during development.

This database structure and security rule considerations provide a foundation for securely storing multi-tenant credentials. The actual implementation of CRUD operations will be handled by our Cloud Functions.

## Credential Management - Server-Side

This section details how the backend (Cloud Functions) will handle the storage, retrieval, encryption, and decryption of tenant-specific credentials. We'll expand on the Express app created earlier.

1.  **Encryption and Decryption Utilities:**
    *   Sensitive credentials must be encrypted before being stored in Firestore. We'll use Node.js's built-in `crypto` module.
    *   Create a utility file for encryption, e.g., `functions/src/encryption.js`:

        ```javascript
        // functions/src/encryption.js
        const crypto = require('crypto');
        const functions = require('firebase-functions');

        // IMPORTANT: Store your encryption key securely!
        // Use Firebase environment configuration or Google Cloud Secret Manager.
        // For example, set it via Firebase CLI:
        // firebase functions:config:set secrets.encryption_key="a_very_strong_random_32_byte_hex_string"
        // The key should be a 32-byte (256-bit) hex-encoded string for AES-256.
        let encryptionKey;
        try {
            encryptionKey = functions.config().secrets.encryption_key;
            if (!encryptionKey || encryptionKey.length !== 64) { // 32 bytes = 64 hex characters
                console.error("Encryption key is not defined or not the correct length (64 hex chars for 32 bytes). Deploy one using 'firebase functions:config:set secrets.encryption_key=YOUR_KEY'");
                // Fallback or throw error - for production, this should halt or use a known bad state.
            }
        } catch (error) {
            console.error("Error accessing functions.config().secrets.encryption_key. Make sure it's set.", error);
            // Handle error: perhaps the config isn't set, or emulators don't have it via this method.
            // For local emulation, you might need to set this in .runtimeconfig.json or handle differently.
            // See: https://firebase.google.com/docs/functions/local-emulator#set_up_functions_configuration
            // Example for .runtimeconfig.json in functions directory: { "secrets": { "encryption_key": "your_64_char_hex_key_for_local_testing" } }
        }


        const ALGORITHM = 'aes-256-cbc';
        const IV_LENGTH = 16; // For AES, this is always 16

        function encrypt(text) {
            if (!encryptionKey) {
                throw new Error("Encryption key is not available. Cannot encrypt data.");
            }
            const keyBuffer = Buffer.from(encryptionKey, 'hex');
            const iv = crypto.randomBytes(IV_LENGTH);
            const cipher = crypto.createCipheriv(ALGORITHM, keyBuffer, iv);
            let encrypted = cipher.update(text, 'utf8', 'hex');
            encrypted += cipher.final('hex');
            return {
                iv: iv.toString('hex'),
                encryptedData: encrypted
            };
        }

        function decrypt(encryptedData, ivHex) {
            if (!encryptionKey) {
                throw new Error("Encryption key is not available. Cannot decrypt data.");
            }
            const keyBuffer = Buffer.from(encryptionKey, 'hex');
            const iv = Buffer.from(ivHex, 'hex');
            const decipher = crypto.createDecipheriv(ALGORITHM, keyBuffer, iv);
            let decrypted = decipher.update(encryptedData, 'hex', 'utf8');
            decrypted += decipher.final('utf8');
            return decrypted;
        }

        module.exports = { encrypt, decrypt };
        ```
    *   **Security Note on Encryption Key:**
        *   The `encryptionKey` **MUST** be a strong, randomly generated key (e.g., 32 bytes / 256 bits, represented as 64 hexadecimal characters).
        *   Store it securely using Firebase environment configuration (`firebase functions:config:set secrets.encryption_key=YOUR_KEY`) or Google Cloud Secret Manager. **Never hardcode it.**
        *   For local development with Firebase Emulators, you can create a `functions/.runtimeconfig.json` file with the key:
            ```json
            {
              "secrets": {
                "encryption_key": "put_a_64_character_hex_string_here_for_testing"
              }
            }
            ```
            Ensure `.runtimeconfig.json` is added to your `.gitignore` file to prevent committing it.

2.  **API Endpoints for Storing and Retrieving Credentials:**
    *   Modify `functions/src/index.js` to include routes for managing credentials.
    *   We need a middleware to authenticate Shopify requests and extract `shop_id`. For now, we'll placeholder this and assume `shop_id` is available. (Full Shopify auth will be discussed later).

    ```javascript
    // functions/src/index.js
    // ... (other imports: functions, admin, express, cors)
    const { encrypt, decrypt } = require('./encryption'); // Import encryption utilities

    // ... (admin.initializeApp(), app setup, cors, express.json())

    // --- Mock Shopify Auth Middleware (Replace with actual Shopify verification later) ---
    // This middleware simulates extracting shop_id from a verified Shopify request.
    // In a real app, this would involve HMAC validation or session token validation.
    const authenticateShopifyRequest = (req, res, next) => {
      // For demonstration, let's assume shop_id comes from a header or query param.
      // In a real Shopify app, this would be from a verified session (e.g., JWT from App Bridge).
      const shopId = req.headers['x-shop-id'] || req.query.shop_id;
      if (!shopId) {
        return res.status(401).send("Unauthorized: Missing shop_id.");
      }
      req.shop_id = shopId; // Add shop_id to the request object
      console.log(`Request authenticated for shop: ${shopId}`);
      next();
    };
    // --- End Mock Shopify Auth Middleware ---


    const credentialsRouter = express.Router();
    credentialsRouter.use(authenticateShopifyRequest); // Apply mock auth to all credential routes

    // Endpoint to save/update a specific credential for a shop
    // POST /api/credentials/:credential_type (e.g., /api/credentials/google_ads_token)
    credentialsRouter.post('/:credentialType', async (req, res) => {
      const { shop_id } = req; // From authenticateShopifyRequest middleware
      const { credentialType } = req.params;
      const { value, customer_id } = req.body; // Expecting 'value' for the secret, 'customer_id' optional

      if (!value) {
        return res.status(400).send(`Bad Request: 'value' for credential ${credentialType} is missing.`);
      }

      try {
        const { iv, encryptedData } = encrypt(value);
        const shopRef = admin.firestore().collection('shops').doc(shop_id);

        const credentialData = {
          iv: iv,
          encrypted_value: encryptedData
        };
        // If customer_id is part of this credential (like for Google Ads), store it directly.
        // It's often not encrypted if it's just an identifier.
        if (customer_id) {
          credentialData.customer_id = customer_id;
        }

        // Using FieldPaths to update specific keys within the 'credentials' map
        const updatePath = `credentials.${credentialType.replace(/-/g, '_')}`; // e.g. credentials.google_ads_refresh_token

        await shopRef.set({
          [updatePath]: credentialData
        }, { merge: true });

        console.log(`Credential ${credentialType} stored for shop ${shop_id}`);
        res.status(200).send(`Credential ${credentialType} stored successfully.`);
      } catch (error) {
        console.error(`Error storing credential ${credentialType} for shop ${shop_id}:`, error);
        if (error.message.includes("Encryption key is not available")) {
            return res.status(500).send("Server configuration error: Encryption service not available.");
        }
        res.status(500).send("Error storing credential.");
      }
    });

    // Endpoint for the backend to retrieve and decrypt a credential
    // This is an internal helper function, not typically exposed directly as a GET without strong auth.
    // Or, if exposed, it needs strict permissions.
    // For now, let's assume this is called internally by other backend functions.
    async function getDecryptedCredential(shopId, credentialType) {
      const shopRef = admin.firestore().collection('shops').doc(shopId);
      const doc = await shopRef.get();

      if (!doc.exists) {
        console.warn(`Shop document not found for shopId: ${shopId}`);
        return null;
      }

      const shopData = doc.data();
      const credentialPath = `credentials.${credentialType.replace(/-/g, '_')}`;

      // Navigate the path if it's nested (e.g., credentials.google_ads_refresh_token.encrypted_value)
      const credentialSet = credentialPath.split('.').reduce((o, k) => (o || {})[k], shopData);

      if (credentialSet && credentialSet.encrypted_value && credentialSet.iv) {
        try {
          const decryptedValue = decrypt(credentialSet.encrypted_value, credentialSet.iv);
          // If there's also a customer_id or similar non-encrypted part, return it too.
          return {
            value: decryptedValue,
            customer_id: credentialSet.customer_id // Will be undefined if not present
          };
        } catch (error) {
          console.error(`Error decrypting credential ${credentialType} for shop ${shopId}:`, error);
          throw new Error(`Could not decrypt credential ${credentialType}.`);
        }
      } else {
        console.warn(`Credential ${credentialType} not found or incomplete for shop ${shopId}`);
        return null;
      }
    }

    // Example of using getDecryptedCredential internally:
    // (This would be part of another function that needs the credential)
    // shopifyRouter.get('/make-google-ads-call', async (req, res) => {
    //   const { shop_id } = req;
    //   try {
    //     const tokenData = await getDecryptedCredential(shop_id, 'google_ads_refresh_token');
    //     if (tokenData && tokenData.value) {
    //       // Now use tokenData.value (decrypted token) and tokenData.customer_id to make an API call
    //       res.status(200).send(`Successfully retrieved token: ${tokenData.value.substring(0,10)}... and customer ID: ${tokenData.customer_id}`);
    //     } else {
    //       res.status(404).send("Google Ads token not configured for this shop.");
    //     }
    //   } catch (error) {
    //     res.status(500).send(`Error: ${error.message}`);
    //   }
    // });


    app.use("/api/credentials", credentialsRouter); // Mount the credentials router

    // ... (Shopify router and exports.api = functions.https.onRequest(app);)
    ```

3.  **Handling OAuth Flows (e.g., Google Ads Refresh Token):**
    *   OAuth flows typically involve redirecting the user to the third-party provider (e.g., Google) and then handling a callback.
    *   You'll need two endpoints: one to initiate the flow and one for the callback.

    ```javascript
    // functions/src/index.js
    // ... (other imports)
    // For Google OAuth: npm install googleapis
    const { google } = require('googleapis');

    // --- Google OAuth Configuration ---
    // These should be from your Firebase environment config, similar to the encryption key
    // firebase functions:config:set google.client_id="YOUR_GOOGLE_CLIENT_ID"
    // firebase functions:config:set google.client_secret="YOUR_GOOGLE_CLIENT_SECRET"
    // firebase functions:config:set google.redirect_uri="https://<your-function-url>/api/oauth/google/callback"
    // Or for local: http://localhost:5001/<project-id>/<region>/api/oauth/google/callback

    let googleClientId, googleClientSecret, googleRedirectUri;
    try {
        googleClientId = functions.config().google.client_id;
        googleClientSecret = functions.config().google.client_secret;
        googleRedirectUri = functions.config().google.redirect_uri; // Ensure this matches your GCP console & function URL
    } catch(e) {
        console.error("Google OAuth config missing. Set google.client_id, google.client_secret, google.redirect_uri in Firebase config.");
    }

    const oauth2Client = new google.auth.OAuth2(
        googleClientId,
        googleClientSecret,
        googleRedirectUri
    );
    // --- End Google OAuth Configuration ---

    const oauthRouter = express.Router();
    oauthRouter.use(authenticateShopifyRequest); // Protect OAuth initiation with Shopify auth

    // 1. Endpoint to initiate Google OAuth flow
    // GET /api/oauth/google/initiate
    oauthRouter.get('/google/initiate', (req, res) => {
      const { shop_id } = req; // From authenticateShopifyRequest

      if (!googleClientId) { // Check if config loaded
          return res.status(500).send("Google OAuth is not configured on the server.");
      }

      const scopes = [
        'https://www.googleapis.com/auth/adwords', // Google Ads API
        // Add other scopes if needed
      ];

      const authorizeUrl = oauth2Client.generateAuthUrl({
        access_type: 'offline', // Important to get a refresh token
        scope: scopes,
        // Pass shop_id in the state to identify the user upon callback
        state: shop_id,
        prompt: 'consent' // Ensures the consent screen is shown, good for refresh tokens
      });
      res.redirect(authorizeUrl);
    });

    // 2. Endpoint to handle Google OAuth callback
    // GET /api/oauth/google/callback
    oauthRouter.get('/google/callback', async (req, res) => {
      const { code, state: shop_id, error: oauthError } = req.query;

      if (oauthError) {
        console.error("OAuth Error on callback:", oauthError);
        return res.status(400).send(`OAuth Error: ${oauthError}. Please try again.`);
      }

      if (!code) {
        return res.status(400).send("Missing authorization code from Google.");
      }
      if (!shop_id) {
        // This should ideally not happen if state was correctly passed and returned
        return res.status(400).send("Missing state (shop_id) from Google callback.");
      }

      try {
        const { tokens } = await oauth2Client.getToken(code);
        const refreshToken = tokens.refresh_token;
        // Note: Access tokens (tokens.access_token) are short-lived.
        // The refresh_token is long-lived and should be stored securely.

        if (!refreshToken) {
          // This can happen if the user has already granted consent and a refresh token was not re-issued.
          // Or if 'prompt: consent' was not used, or if the app is not configured for "offline" access.
          console.warn(`Refresh token not received for shop ${shop_id}. Access token: ${tokens.access_token ? 'received' : 'not received'}`);
          // You might need to guide the user or just store the access token if your use case allows.
          // For long-term access (like Google Ads), refresh token is essential.
          // If this happens, you might need to instruct users to revoke app access from their Google account and retry.
          return res.status(400).send("Could not obtain a refresh token. Please ensure you haven't already authorized this app, or try revoking and re-authorizing. If the problem persists, contact support.");
        }

        // Encrypt and store the refresh token
        const { iv, encryptedData } = encrypt(refreshToken);
        const shopRef = admin.firestore().collection('shops').doc(shop_id);

        // Store it as 'google_ads_refresh_token'
        await shopRef.set({
          credentials: {
            google_ads_refresh_token: {
              iv: iv,
              encrypted_value: encryptedData
            }
          }
        }, { merge: true }); // Merge: true to not overwrite other credentials or shop data

        console.log(`Google Ads Refresh Token stored for shop ${shop_id}`);
        // Redirect user back to your app's settings page in Shopify admin
        // The actual redirect URL will depend on your app's navigation
        res.redirect(`https://${shop_id}/admin/apps/your-app-name/settings?status=google_oauth_success`);

      } catch (error) {
        console.error(`Error exchanging code or storing token for shop ${shop_id}:`, error.response ? error.response.data : error.message);
        res.status(500).send("Error processing Google OAuth callback.");
      }
    });

    app.use("/api/oauth", oauthRouter); // Mount the OAuth router

    // Make sure to export the main 'api' function
    // exports.api = functions.https.onRequest(app); // Already defined in previous steps
    ```
    *   **Important OAuth Notes:**
        *   The `redirect_uri` in your Google Cloud Console project **must exactly match** the URL of your callback function.
        *   For local testing, you'll need to use `ngrok` or a similar tool to get a public HTTPS URL for your local emulator, and add that as a valid redirect URI in Google Cloud Console.
        *   Always include the `state` parameter in the initial OAuth request and verify it in the callback to prevent CSRF attacks. Here, we use `shop_id` as the state.

This provides a robust server-side mechanism for managing credentials, including encryption/decryption and handling OAuth flows. The next step is to integrate this with Shopify's authentication.

## Shopify App Backend Logic

Integrating with Shopify requires your backend to securely handle requests originating from your app's frontend (running in the Shopify admin) and potentially webhooks from Shopify. Key aspects include request verification and obtaining the `shop_id`.

1.  **Shopify Request Verification:**
    *   All requests from your Shopify app frontend (embedded in Shopify Admin) or webhooks from Shopify must be verified to ensure they are legitimate and originate from Shopify.
    *   **For requests from Embedded Frontend (using App Bridge):**
        *   Shopify App Bridge provides a way to get a session token (JWT). This token should be sent with every AJAX request from your React/Polaris frontend to your backend.
        *   Your backend must then verify this JWT. Libraries like `jsonwebtoken` can be used, and you'll need Shopify's JWKS (JSON Web Key Set) URI to get the public keys.
        *   The decoded JWT will contain the `dest` (shop domain, e.g., `your-store.myshopify.com`), which is your `shop_id`.
    *   **For Webhooks:**
        *   Shopify signs webhook requests with a secret key using HMAC-SHA256. You must verify this signature.
    *   **For OAuth (App Installation):**
        *   The initial app installation process uses OAuth 2.0. You'll exchange an authorization code for an access token. The shop domain is provided during this process. This access token should be stored securely (encrypted in Firestore, similar to other credentials) and associated with the `shop_id`.

2.  **Replacing Mock Authentication with Actual Shopify JWT Verification:**
    *   Let's refine the `authenticateShopifyRequest` middleware in `functions/src/index.js`.
    *   You'll need `jsonwebtoken` and `jwks-rsa`. Install them:
        ```bash
        cd functions
        npm install jsonwebtoken jwks-rsa
        ```

    ```javascript
    // functions/src/index.js
    // ... (other imports)
    const jwt = require('jsonwebtoken');
    const jwksClient = require('jwks-rsa');

    // --- Shopify JWT Authentication Middleware ---
    // Configure this with your Shopify App API key and secret if needed for other checks,
    // but for App Bridge JWTs, the main thing is the JWKS client.
    // const SHOPIFY_API_KEY = functions.config().shopify.api_key; // Set via firebase functions:config:set shopify.api_key="..."
    // const SHOPIFY_API_SECRET = functions.config().shopify.api_secret; // Set via firebase functions:config:set shopify.api_secret="..."

    const shopifyJwksClient = jwksClient({
      jwksUri: `https://shopify-app-jwks.shopifycloud.com/YOUR_SHOPIFY_APP_API_KEY/.well-known/jwks.json`
      // IMPORTANT: Replace YOUR_SHOPIFY_APP_API_KEY with your actual Shopify App's API key.
      // You might want to make YOUR_SHOPIFY_APP_API_KEY a Firebase config variable.
      // e.g., functions.config().shopify.api_key
    });

    function getKey(header, callback) {
      shopifyJwksClient.getSigningKey(header.kid, function(err, key) {
        if (err) {
          console.error("Error fetching signing key:", err);
          return callback(err);
        }
        const signingKey = key.publicKey || key.rsaPublicKey;
        callback(null, signingKey);
      });
    }

    const authenticateShopifyRequest = (req, res, next) => {
      const authHeader = req.headers.authorization;
      if (!authHeader || !authHeader.startsWith('Bearer ')) {
        console.warn("Missing or invalid Authorization header.");
        return res.status(401).send('Unauthorized: Missing or invalid token.');
      }

      const token = authHeader.substring(7); // Remove "Bearer " prefix

      // Verify the JWT
      // The audience ('aud') and issuer ('iss') claims should also be validated against your app's API key and Shopify's issuer URL.
      // Audience should be your Shopify App API Key.
      // Issuer should be `https://[shop_domain]/admin` where [shop_domain] is the `dest` claim.
      jwt.verify(token, getKey, { algorithms: ['RS256'] }, (err, decoded) => {
        if (err) {
          console.error('JWT verification error:', err.message);
          return res.status(401).send('Unauthorized: Invalid token.');
        }

        // Validate audience and issuer
        // const expectedAudience = SHOPIFY_API_KEY;
        // if (decoded.aud !== expectedAudience) {
        //   console.warn(`Invalid JWT audience. Expected: ${expectedAudience}, Got: ${decoded.aud}`);
        //   return res.status(401).send('Unauthorized: Invalid token audience.');
        // }
        // The `dest` claim contains the shop's myshopify.com domain (this is the shop_id)
        // The `iss` claim should be `https://<dest_domain>/admin`
        // const expectedIssuer = `https://${decoded.dest}/admin`;
        // if (decoded.iss !== expectedIssuer) {
        //    console.warn(`Invalid JWT issuer. Expected: ${expectedIssuer}, Got: ${decoded.iss}`);
        //    return res.status(401).send('Unauthorized: Invalid token issuer.');
        // }


        // The `dest` claim usually holds the shop's *.myshopify.com domain.
        // This is your shop_id.
        const shopId = decoded.dest.replace(/^https?:\/\//, ''); // Remove http(s):// prefix if present
        if (!shopId) {
          console.warn("JWT 'dest' claim missing or invalid.");
          return res.status(401).send('Unauthorized: Invalid token content.');
        }
        req.shop_id = shopId;
        req.shopify_session_token_payload = decoded; // Store full decoded payload if needed

        console.log(`Request authenticated via JWT for shop: ${shopId}`);
        next();
      });
    };
    // --- End Shopify JWT Authentication Middleware ---

    // Make sure to replace the mock middleware with this one in your routers:
    // credentialsRouter.use(authenticateShopifyRequest);
    // oauthRouter.use(authenticateShopifyRequest); // For initiating OAuth from an authenticated frontend
    // shopifyRouter.use(authenticateShopifyRequest);
    ```
    *   **Important:** You **must** replace `YOUR_SHOPIFY_APP_API_KEY` in the `jwksUri` with your actual Shopify app's API key. It's best to load this from Firebase config.
    *   Properly validate `aud` (audience) and `iss` (issuer) claims for full security. The audience is your app's API key. The issuer is `https://{shop_domain}/admin`, where `{shop_domain}` is the value from the `dest` claim.

3.  **Using `shop_id` to Fetch Credentials and Make API Calls:**
    *   Once `authenticateShopifyRequest` has run, `req.shop_id` will be available in your route handlers.
    *   You can then use this `shop_id` with the `getDecryptedCredential` function (defined in the previous section) to retrieve the necessary API keys or tokens for that specific shop.

    ```javascript
    // Example: In functions/src/index.js, within your shopifyRouter

    // const { getDecryptedCredential } = require('./credentialManager'); // Assuming you refactored it

    shopifyRouter.get('/get-my-google-ads-data', async (req, res) => {
      const { shop_id } = req; // Provided by authenticateShopifyRequest middleware

      try {
        // Retrieve the Google Ads refresh token and customer ID for this shop
        const googleAdsCreds = await getDecryptedCredential(shop_id, 'google_ads_refresh_token');

        if (!googleAdsCreds || !googleAdsCreds.value) {
          return res.status(404).send({ message: "Google Ads credentials not configured for this shop." });
        }

        const refreshToken = googleAdsCreds.value;
        const customerId = googleAdsCreds.customer_id; // This was stored alongside the token

        if (!customerId) {
            return res.status(404).send({ message: "Google Ads Customer ID not configured for this shop." });
        }

        // Now, use the refreshToken and customerId to make a call to Google Ads API
        // This part would involve using the googleapis library, setting credentials on oauth2Client,
        // and then calling the Google Ads service.

        // --- Placeholder for Google Ads API Call ---
        console.log(`Attempting Google Ads API call for shop ${shop_id}, customer ${customerId} using token starting with ${refreshToken.substring(0,10)}...`);
        // const adsData = await callYourGoogleAdsFunction(refreshToken, customerId, functions.config().googleads.developer_token);
        const adsData = { message: "Successfully fetched Google Ads data (mocked).", customerId: customerId, tokenUsed: refreshToken.substring(0,10)+"..." };
        // --- End Placeholder ---

        res.status(200).json(adsData);

      } catch (error) {
        console.error(`Error in /get-my-google-ads-data for shop ${shop_id}:`, error);
        if (error.message.includes("Could not decrypt credential")) {
            return res.status(500).send({ message: "Error accessing credentials. Please check configuration."});
        }
        res.status(500).send({ message: "An error occurred while fetching Google Ads data." });
      }
    });

    // app.use("/api/shopify", shopifyRouter); // Ensure it's mounted
    ```

4.  **Storing Shopify Access Token:**
    *   During the app installation (OAuth flow with Shopify), you will receive an access token for the shop. This token is needed to make Shopify Admin API calls on behalf of the shop.
    *   This token is per-shop and should be stored securely, encrypted, in your Firestore `shops/{shop_id}` document, just like other credentials.
    *   Example structure in Firestore for `shops/{shop_id}`:
        ```json
        {
          "shopifyAccessToken": {
            "iv": "some_iv",
            "encrypted_value": "encrypted_shopify_access_token"
          },
          "credentials": {
            // ... other credentials
          }
          // ... other shop data
        }
        ```
    *   Your backend will need a function similar to `getDecryptedCredential` to retrieve and decrypt this Shopify access token when needed.

This setup ensures that your backend can securely identify the Shopify store making the request and then use the correct set of credentials for any third-party API interactions. Remember to thoroughly test the JWT validation and the overall flow.

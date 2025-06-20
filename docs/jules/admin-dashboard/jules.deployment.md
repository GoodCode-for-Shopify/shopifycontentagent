# Admin Dashboard: Deployment

This document outlines the deployment process for the Admin Dashboard, including its backend (Cloud Functions for Firebase) and frontend (React/Polaris app on Firebase Hosting).

## 1. Prerequisites for Deployment

*   **Firebase Project:** Ensure your Firebase project is set up and you are on the Blaze (pay-as-you-go) plan, as required for deploying 2nd Gen Cloud Functions and for any significant outbound networking.
*   **Firebase CLI:** Make sure the Firebase CLI is installed and you are logged into the correct Google account associated with your Firebase project (`firebase login`).
*   **Project Configuration:** Your `firebase.json` and `.firebaserc` files should be correctly configured, associating your local project with your Firebase project.
*   **Environment Variables:** All necessary environment variables for your Cloud Functions (e.g., Stripe keys, API keys for Google services, your encryption key) must be set in your Firebase project's runtime configuration.
    ```bash
    # Example:
    firebase functions:config:set stripe.secret_key="sk_live_YOUR_STRIPE_SECRET_KEY"
    firebase functions:config:set secrets.encryption_key="YOUR_SECURE_ENCRYPTION_KEY"
    # ... and so on for all required keys.
    # Verify with: firebase functions:config:get
    ```
    **Important:** Use your **live** mode Stripe keys for production deployments, not test keys.

## 2. Deploying Backend (Cloud Functions)

(Derived from Grok Outline Section 8.1, adapted for Firebase)

Your backend, consisting of Express.js routes running within Cloud Functions, is deployed using the Firebase CLI.

### 2.1. Deployment Command
*   Navigate to your project root directory (where `firebase.json` is located).
*   To deploy all functions defined in `functions/src/index.js` (or as per your `firebase.json` functions source):
    ```bash
    firebase deploy --only functions
    ```
*   If you have multiple exported functions and only want to deploy a specific one (e.g., the main Express app exported as `api` or `adminApi`):
    ```bash
    firebase deploy --only functions:api
    # or firebase deploy --only functions:adminApi
    ```
*   The CLI will show the deployment progress. Upon completion, it will list the URLs of your HTTP-triggered functions.

### 2.2. Regional Configuration
*   For 2nd Gen Cloud Functions, the region is typically specified in the code:
    ```javascript
    // In functions/src/index.js
    // exports.api = functions.region('your-chosen-region').https.onRequest(app);
    ```
*   Ensure `your-chosen-region` (e.g., `us-central1`, `europe-west1`) is consistent and appropriate for your user base and other services.

## 3. Deploying Frontend (React/Polaris Admin Dashboard)

(Derived from Grok Outline Section 8.1, adapted for Firebase Hosting)

The React/Polaris admin dashboard frontend is deployed to Firebase Hosting.

### 3.1. Build Your React Application
*   Before deploying, create a production build of your React app.
*   Navigate to your frontend application's directory (e.g., `admin-dashboard-ui/`):
    ```bash
    cd admin-dashboard-ui
    npm run build
    cd .. # Return to project root
    ```
    (The build command might be different if you used Vite, e.g., `npm run build` which executes `vite build`).
    This command will typically create a `build` or `dist` folder within `admin-dashboard-ui/` containing the static assets.

### 3.2. Firebase Hosting Configuration
*   Ensure your `firebase.json` correctly points to the build output directory and includes necessary rewrites:
    ```json
    {
      "hosting": {
        "public": "admin-dashboard-ui/build", // Or dist for Vite
        "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
        "rewrites": [
          {
            "source": "/api/admin/**", // Or your admin API path
            "function": "api"         // Your main Cloud Function name
          },
          {
            "source": "**",
            "destination": "/index.html"
          }
        ]
      }
      // ... functions configuration ...
    }
    ```
    *   The `public` directory must match your React app's production build output folder.
    *   The API rewrite ensures that calls from your frontend to your backend API (e.g., `https://your-project-id.web.app/api/admin/...`) are routed to your Cloud Function.
    *   The SPA rewrite (`"source": "**", "destination": "/index.html"`) is essential for client-side routing in React.

### 3.3. Deployment Command
*   From your project root directory:
    ```bash
    firebase deploy --only hosting
    ```
*   The CLI will upload your static assets. After completion, it will provide the URL where your admin dashboard is hosted (e.g., `https://your-project-id.web.app`).

## 4. Deploying Both Backend and Frontend
*   To deploy all Firebase services (Functions, Hosting, Firestore rules, etc.) defined in your `firebase.json` simultaneously:
    ```bash
    firebase deploy
    ```

## 5. Testing Webhooks After Deployment

(Derived from Grok Outline Section 8.2)

*   **Stripe Webhooks:**
    *   Once your Cloud Function webhook handler (e.g., `stripeWebhookHandler`) is deployed, update the endpoint URL in your Stripe Dashboard (Developers > Webhooks).
    *   Use the **live** URL of your deployed Cloud Function.
    *   Ensure you are using your **live** Stripe Webhook Signing Secret in your Firebase environment configuration for the deployed function.
    *   Send test events from the Stripe Dashboard to your live webhook endpoint to verify it's working correctly.
    *   Monitor logs for your webhook Cloud Function in the Firebase Console or Google Cloud Logging to troubleshoot any issues.

## 6. Monitoring and Optimization Post-Deployment

(Derived from Grok Outline Section 8.3)

*   **Firebase Console:**
    *   **Functions:** Monitor invocation counts, execution times, error rates, and logs.
    *   **Hosting:** Track data storage, transfer, and cache performance.
    *   **Firestore:** Monitor usage (reads, writes, deletes), storage size, and index performance.
*   **Google Cloud Monitoring (Cloud Logging & Cloud Monitoring):**
    *   Provides more detailed metrics and logging capabilities for your Cloud Functions and other Google Cloud resources used by Firebase.
    *   Set up alerts for high error rates, excessive function execution times, or nearing Firestore quotas.
*   **Performance Optimization:**
    *   Based on monitoring data, identify and optimize slow-running Cloud Functions or inefficient Firestore queries.
    *   Adjust Cloud Function memory allocation if needed.
    *   Review and optimize frontend bundle sizes and loading performance.

Regularly deploying updates and monitoring the health of your deployed application is crucial for maintaining a stable and performant admin dashboard. Consider setting up a CI/CD pipeline (e.g., using GitHub Actions with Firebase CLI, or Google Cloud Build) for automated deployments to staging and production environments.

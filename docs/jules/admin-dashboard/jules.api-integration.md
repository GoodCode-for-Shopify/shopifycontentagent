# Admin Dashboard: Third-Party API Integration

This document describes how the backend (Cloud Functions for Firebase) integrates with third-party APIs, specifically Google Ads API and Gemini AI API (or equivalent Google AI services). It also covers strategies for rate limiting and queuing API calls.

## 1. Google Ads API Integration

(Derived from Grok Outline Section 5.1, adapted for Cloud Functions)

Integration with the Google Ads API allows the application to manage and report on ad campaigns on behalf of tenants.

### 1.1. Google Ads API Client Setup
*   **Node.js Client Library:** Use Google's official Node.js client library for the Google Ads API.
    ```bash
    # In functions/ directory
    npm install google-ads-api
    ```
*   **Developer Token:**
    *   A Google Ads Developer Token is required for all API calls.
    *   Store this token securely in Firebase environment configuration, as it's an app-wide credential:
        ```bash
        firebase functions:config:set googleads.developer_token="YOUR_DEVELOPER_TOKEN"
        ```
    *   Access it in your Cloud Function: `functions.config().googleads.developer_token`.

### 1.2. OAuth 2.0 for Tenant Authorization
*   Tenants must authorize your application to access their Google Ads data via OAuth 2.0.
*   The OAuth flow (initiation and callback handling) is managed by backend Cloud Functions, as detailed in `docs/jules/serverside-setup.md` (Section: "Credential Management - Server-Side", subsection "Handling OAuth Flows").
*   **Client ID and Client Secret:** Your Google Cloud project's OAuth 2.0 Client ID and Client Secret are used in this flow. Store them securely in Firebase environment configuration:
    ```bash
    firebase functions:config:set googleoauth.client_id="YOUR_GOOGLE_CLIENT_ID"
    firebase functions:config:set googleoauth.client_secret="YOUR_GOOGLE_CLIENT_SECRET"
    firebase functions:config:set googleoauth.redirect_uri="YOUR_CLOUD_FUNCTION_CALLBACK_URL"
    ```
*   **Refresh Tokens:** The obtained OAuth refresh token for each tenant is encrypted and stored in Firestore (e.g., `shops/{shop_id}/credentials/google_ads_refresh_token`).

### 1.3. Making API Calls
*   When a tenant-specific Google Ads API call is needed (e.g., triggered by the admin dashboard or the end-user Shopify app):
    1.  A Cloud Function is invoked.
    2.  Retrieve the tenant's encrypted Google Ads refresh token and their Google Ads Customer ID from Firestore.
    3.  Decrypt the refresh token.
    4.  Initialize the Google Ads API client using:
        *   Your Developer Token.
        *   The tenant's OAuth refresh token (to obtain an access token).
        *   Your OAuth Client ID and Client Secret.
        *   The tenant's Google Ads Customer ID (and Login Customer ID, if applicable, also stored per-tenant).
    5.  Make the desired API call (e.g., fetch campaigns, create reports).

    ```javascript
    // Simplified example in a Cloud Function
    // const { GoogleAdsApi } = require('google-ads-api');
    // const functions = require('firebase-functions');
    // const { getDecryptedCredential } = require('./credentialManager'); // Your utility

    // async function fetchCampaignsForShop(shopId, customerId) {
    //   const devToken = functions.config().googleads.developer_token;
    //   const clientId = functions.config().googleoauth.client_id;
    //   const clientSecret = functions.config().googleoauth.client_secret;

    //   const refreshTokenData = await getDecryptedCredential(shopId, 'google_ads_refresh_token');
    //   if (!refreshTokenData || !refreshTokenData.value) {
    //     throw new Error('Missing Google Ads refresh token for shop.');
    //   }
    //   const refreshToken = refreshTokenData.value;

    //   const client = new GoogleAdsApi({
    //     client_id: clientId,
    //     client_secret: clientSecret,
    //     developer_token: devToken,
    //   });

    //   const customer = client.Customer({
    //     customer_id: customerId, // Tenant's Google Ads Customer ID
    //     refresh_token: refreshToken,
    //     // login_customer_id: tenantLoginCustomerId, // If managing via MCC
    //   });

    //   const results = await customer.query(`
    //     SELECT campaign.id, campaign.name FROM campaign ORDER BY campaign.name
    //   `);
    //   return results;
    // }
    ```

## 2. Gemini AI API (or other Google AI Services) Integration

(Derived from Grok Outline Section 5.2, adapted for Google AI and Firebase)

Integration with Google's AI services like Gemini (via Vertex AI or Google AI Studio SDKs) provides AI-driven features.

### 2.1. API Key Management
*   **Developer-Provided API Key:**
    *   If you offer AI features using your own Google AI API Key (e.g., for paid plans with an additional fee):
        *   Store your Google AI API Key securely in Firebase environment configuration:
            ```bash
            firebase functions:config:set googleai.api_key="YOUR_GEMINI_OR_VERTEX_AI_API_KEY"
            ```
        *   Access it in Cloud Functions: `functions.config().googleai.api_key`.
        *   Track usage per tenant meticulously for billing and quota management (see `docs/jules/admin-dashboard/jules.backend-development.md` - API Usage Tracking).
*   **Tenant-Provided API Key:**
    *   If tenants (especially on free or lower-tier plans) are required to use their own Google AI API Key:
        *   The key is provided by the tenant via the app's settings UI.
        *   It's encrypted and stored in Firestore (e.g., `shops/{shop_id}/credentials/gemini_ai_api_key`).
        *   The Cloud Function retrieves and decrypts this key when making AI calls for that tenant.

### 2.2. Making AI API Calls
*   Use the appropriate Google AI Node.js client library (e.g., ` @google/generative-ai ` for Gemini through Google AI Studio, or ` @google-cloud/vertexai ` for Vertex AI).
    ```bash
    # In functions/ directory
    npm install @google/generative-ai
    # or
    npm install @google-cloud/vertexai
    ```
*   Initialize the client library with the relevant API key (either the developer's or the tenant's decrypted key).

    ```javascript
    // Simplified example using @google/generative-ai
    // const { GoogleGenerativeAI } = require("@google/generative-ai");
    // const functions = require('firebase-functions');
    // const { getDecryptedCredential } = require('./credentialManager');

    // async function generateTextWithGemini(shopId, prompt) {
    //   let apiKey;
    //   // Logic to determine if using developer key or tenant key based on plan/opt-in
    //   const useDeveloperKey = await shouldUseDeveloperAIKey(shopId); // Your custom logic

    //   if (useDeveloperKey) {
    //     apiKey = functions.config().googleai.api_key;
    //   } else {
    //     const apiKeyData = await getDecryptedCredential(shopId, 'gemini_ai_api_key');
    //     if (!apiKeyData || !apiKeyData.value) {
    //       throw new Error('Missing Gemini AI API key for shop.');
    //     }
    //     apiKey = apiKeyData.value;
    //   }

    //   if (!apiKey) throw new Error('AI API Key not available.');

    //   const genAI = new GoogleGenerativeAI(apiKey);
    //   const model = genAI.getGenerativeModel({ model: "gemini-pro" }); // Or other model
    //   const result = await model.generateContent(prompt);
    //   const response = await result.response;
    //   const text = response.text();

    //   // Log API usage
    //   await logApiCall(shopId, 'gemini_ai', 1, useDeveloperKey);
    //   return text;
    // }
    ```

## 3. Rate Limiting and Queuing

(Derived from Grok Outline Section 5.3, adapted for Firebase)

Third-party APIs have rate limits. Exceeding them can lead to temporary blocking or errors.

### 3.1. Rate Limiting Strategies
*   **Firestore-based Counters:** For simple API call frequency limits per tenant (e.g., X calls per minute/hour), use atomic counters in Firestore. Before an API call, check and increment the counter. Reset periodically using a scheduled function or time-based logic.
*   **Cloud Functions Concurrency:** Be mindful of Cloud Functions' own concurrency limits. While they scale, a massive burst of requests to your functions could bottleneck if each makes a slow third-party call.
*   **API Client Library Features:** Some client libraries offer built-in retry mechanisms with exponential backoff. Utilize these where available.

### 3.2. Queuing API Calls
For operations that are not time-sensitive or involve many API calls (e.g., batch processing, large report generation), consider queuing.

*   **Firebase Task Queues (Cloud Tasks via Firebase Functions):**
    *   This is the most Firebase-native way to implement queuing.
    *   When a task needs to be performed, a Cloud Function can enqueue a message to a Cloud Tasks queue.
    *   Another HTTP Cloud Function (a task handler) is configured to process messages from this queue.
    *   Cloud Tasks provides controls for execution rate, retries, and concurrency, helping to smooth out calls to third-party APIs.
    ```javascript
    // Example: Enqueuing a task
    // const { getFunctions } = require('firebase-admin/functions');
    // const queue = getFunctions().taskQueue('yourTaskHandlerFunctionName', 'projects/YOUR_PROJECT_ID/locations/YOUR_REGION');
    // await queue.enqueue({ shopId: "shop123", reportType: "campaign_performance" });

    // Example: Task handler function
    // exports.yourTaskHandlerFunctionName = functions.tasks.taskQueue().onDispatch(async (data) => {
    //   const { shopId, reportType } = data;
    //   // Perform the long-running Google Ads API call here
    // });
    ```
*   **Considerations:**
    *   This is more robust than simple Firestore counters for managing API call rates over time.
    *   Replaces the need for external queuing systems like BullMQ mentioned in the Grok document for many use cases within the Firebase ecosystem.

### 3.3. Monitoring External API Usage
*   Regularly monitor your usage of third-party APIs via their respective dashboards (Google Cloud Console for Google Ads and Google AI APIs).
*   Set up alerts in those dashboards if available to get notified when approaching quotas.
*   Correlate this with your internal API usage logs (stored in Firestore) to identify which tenants or features are consuming the most quota.

By carefully managing API keys, implementing robust OAuth flows, and considering rate limiting/queuing strategies, your backend can reliably interact with these essential third-party services.

# Content Agent Shopify App: Overview and Setup

This document provides an overview of the "Content Agent" Shopify app and outlines the initial project setup for its frontend. The app aims to help merchants create SEO-enhanced, conversion-optimized blog articles for their Shopify store or for export to other platforms.
The setup described herein will be replicated for development, staging, and production environments, adhering to the project's 'Multi-Environment Strategy' (see `docs/jules.authority.md`) and detailed in `docs/jules/environments-and-cicd-strategy.md`.

## 1. App Overview

### 1.1. Purpose
Content Agent is designed to streamline the creation of high-quality blog content. Key features include:
*   AI-assisted keyword generation and analysis.
*   AI-driven "People Also Ask" (PAA) question sourcing (via a dedicated Python service).
*   AI-powered article outline creation.
*   AI article construction with a focus on SEO and conversion.
*   Optional AI image generation.
*   Publishing articles as drafts to the Shopify blog or exporting as Markdown/Plain Text.
*   API-based publishing to select external platforms (e.g., WordPress) for paid plan users.
*   Plan-based feature tiers (Free, Basic, Pro, Enterprise) with dynamically configurable limits managed via an admin dashboard.

### 1.2. Core User Flow Summary
1.  **Installation & Onboarding:** Install from Shopify App Store, accept terms, select a plan (Free by default).
2.  **Credential Setup:** Guided walkthrough to input necessary third-party API keys (e.g., Google Ads, Gemini AI, WordPress if applicable for export) based on plan requirements. Credentials are tested upon submission.
3.  **Content Creation (Multi-Step Form):**
    *   **Product Selection:** Choose Shopify product(s) to base content on (single or multiple based on plan).
    *   **Keyword Generation:** AI generates keywords; user refines list based on Google search insights.
    *   **Related Questions (Plan-Dependent):** PAA questions are fetched and selected.
    *   **Article Outline Generation:** AI creates SEO-optimized outlines; user can edit.
    *   **(If multiple articles) Outline Management:** User finalizes outlines before proceeding.
    *   **Article Construction:** AI writes articles based on outlines.
    *   **Image Generation (Plan-Dependent):** AI generates images based on suggestions.
    *   **(If multiple articles) Construction/Imaging Management:** User cycles through construction/imaging for each article.
    *   **Commit/Export:** Commit articles as drafts to Shopify blog, or download (MD/TXT), or publish to a single external platform via API (paid plans).
4.  **Styling & Structure:** The app provides a structure for generated blog posts and allows basic color customization.

### 1.3. Technology Stack Reminder
*   **Frontend:** React with Shopify Polaris, hosted on Firebase Hosting.
*   **Backend:** Node.js/Express on Cloud Functions for Firebase.
*   **Database:** Firestore.
*   **Admin Dashboard:** Separate React/Polaris application for managing plans, tenants, etc.
*   **Python PAA Service:** Separate Python Cloud Function or Cloud Run service for fetching PAA data.

## 2. Shopify App Project Setup

This section focuses on setting up the React/Polaris frontend for the Shopify app that merchants will interact with inside their Shopify admin.

### 2.1. Initialize Shopify App Project
*   Use the Shopify CLI to create a new app project. This provides a recommended structure and tools for Shopify app development.
    ```bash
    npm init @shopify/app@latest content-agent-shopify-app
    ```
*   When prompted, choose **Node.js** as the backend language (even if much of your core backend is in Firebase, the Shopify CLI template often includes a small Node server for OAuth and webhook handling).
*   Choose **React** for the frontend. The Shopify CLI will set up a basic React project, often using Vite or a similar bundler.
*   Navigate into your new app directory: `cd content-agent-shopify-app`

### 2.2. Frontend Directory Structure
The Shopify CLI will create a `web/frontend` directory (or similar, e.g., `frontend/` in older versions). This is where your React/Polaris UI code will live.
*   Familiarize yourself with the structure, typically including `index.jsx` or `main.jsx`, `App.jsx`, and potentially folders for `components/`, `pages/`, `hooks/`, etc.

### 2.3. Install Shopify Polaris
*   If not already included by the Shopify CLI template, or to ensure you have the latest, install Shopify Polaris and its peer dependencies:
    ```bash
    # Inside your app's frontend directory (e.g., web/frontend)
    npm install @shopify/polaris @shopify/app-bridge-react react-router-dom
    ```
    (`react-router-dom` is commonly used for client-side navigation within the embedded app).

### 2.4. Firebase Hosting for Shopify App Frontend
*   **Build Output:** Your React app (in `web/frontend`) will have a build command (e.g., `npm run build`) that generates static assets into a `dist` or `build` folder within `web/frontend`.
*   **`firebase.json` Configuration:** In your *main project root* (where your Firebase project is initialized, which might be one level above the `content-agent-shopify-app` directory if you keep Firebase config separate, or within it if you initialize Firebase there), configure `firebase.json` to deploy this build output.
    *   If your Firebase project is initialized in a parent directory of `content-agent-shopify-app`:
        ```json
        // In <parent-directory>/firebase.json
        {
          "hosting": [
            {
              "target": "admin-dashboard", // If you have a separate target for admin UI
              "public": "admin-dashboard-ui/build", // Or dist
              // ... admin rewrites ...
            },
            {
              "target": "shopify-app", // New target for the Shopify app frontend
              "public": "content-agent-shopify-app/web/frontend/dist", // Adjust path as needed
              "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
              "rewrites": [
                {
                  // API calls from Shopify app go to your Firebase Cloud Functions
                  "source": "/api/**", // Generic API prefix
                  "function": "api"    // Your main Firebase Cloud Function for the app
                },
                {
                  "source": "**",
                  "destination": "/index.html"
                }
              ]
            }
          ]
          // ... functions configuration ...
        }
        ```
        You would then deploy using `firebase deploy --only hosting:shopify-app`.
    *   If Firebase is initialized *within* `content-agent-shopify-app`:
        ```json
        // In content-agent-shopify-app/firebase.json
        {
          "hosting": {
            "public": "web/frontend/dist", // Adjust path as needed
            "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
            "rewrites": [
              {
                "source": "/api/**",
                "function": "api"
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
*   **Shopify App URL:** In your Shopify Partner Dashboard, the App URL for your application will point to the Firebase Hosting URL for this deployment (e.g., `https://your-project-id.web.app` or `https://shopify-app.your-project-id.web.app` if using multiple hosting sites).
    The Shopify App URL configured in the Shopify Partner Dashboard will ultimately point to the production custom domain for the app's frontend (e.g., `app.shopifycontentagent.com` or a specific path on `shopifycontentagent.com` if structured that way).
    For development and staging environments, distinct URLs pointing to their respective Firebase Hosting sites (e.g., `app-dev.shopifycontentagent.com`, `app-staging.shopifycontentagent.com`) will be used. This may require separate Shopify App configurations in the Partner Dashboard for each environment if distinct App URLs and API key/secret pairs are needed per environment, or careful management of a single app configuration with dynamic redirect URIs if supported and chosen.
    Refer to `docs/jules/environments-and-cicd-strategy.md` for the overall custom domain and environment strategy.

### 2.5. Firebase SDK (JavaScript) for Frontend
*   Install the Firebase JavaScript SDK in your Shopify app's frontend:
    ```bash
    # Inside content-agent-shopify-app/web/frontend
    npm install firebase
    ```
*   Initialize Firebase in your React app (e.g., in `web/frontend/src/firebaseConfig.js`), similar to how it was done for the admin dashboard. This will be used for:
    *   Potentially fetching dynamic configuration or user-specific settings if not always proxied via your backend.
    *   (If ever needed) Direct interaction with Firebase services like Firestore (though most data access for the Shopify app should go through your authenticated backend Cloud Functions).
    *   **Important:** Ensure your Firebase Hosting and Cloud Functions are configured to correctly handle requests and authentication from the Shopify app's domain.

### 2.6. Basic App Structure (React/Polaris)
*   **`App.jsx`:**
    *   Wrap with Polaris `AppProvider`.
    *   If using App Bridge, wrap with `Provider` from `@shopify/app-bridge-react` and configure it with your Shopify app API key and the shop origin.
    *   Set up `react-router-dom` for navigation (e.g., `/`, `/onboarding`, `/create-article`, `/settings`).
*   **API Service Module:** Create a dedicated module (e.g., `src/services/api.js`) for making authenticated `fetch` calls to your backend Cloud Functions. This module will use App Bridge to get the session token for the `Authorization` header. The API service module will need to be configured with the correct backend API base URL (e.g., `https://api.shopifycontentagent.by.goodcode.ca` for production, or its dev/staging equivalents) for the environment it's built for. This is typically managed via environment variables during the CI/CD build process.

This initial setup creates the skeleton for your "Content Agent" Shopify app frontend, ready for implementing authentication, onboarding, and the core content generation features.

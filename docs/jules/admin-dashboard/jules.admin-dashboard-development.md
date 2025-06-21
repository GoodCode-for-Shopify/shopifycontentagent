# Admin Dashboard: Frontend Development (React/Polaris)

This document outlines the development of the Administrative Dashboard's frontend using React and Shopify Polaris. The dashboard will be a web application hosted on Firebase Hosting, interacting with the Firebase Cloud Functions backend.

## 1. Frontend Project Setup

(Derived from Grok Outline Section 4.1, adapted for React/Polaris and Firebase Hosting)

### 1.1. Initialize React Project
*   Use Create React App or Vite to scaffold a new React application in the `admin-dashboard-ui` directory (or your chosen frontend directory).
    ```bash
    # Using Create React App
    npx create-react-app admin-dashboard-ui
    cd admin-dashboard-ui

    # OR Using Vite (recommended for faster setup)
    npm create vite@latest admin-dashboard-ui -- --template react
    cd admin-dashboard-ui
    npm install
    ```
*   **Install Shopify Polaris and related dependencies:**
    ```bash
    npm install @shopify/polaris @shopify/app-bridge-react
    # App Bridge might not be strictly necessary if this is a standalone admin panel not embedded in Shopify,
    # but Polaris components are still useful for UI consistency if admins are familiar with Shopify.
    # If it IS embedded or needs to interact like an embedded app, App Bridge is key.
    # For a standalone admin panel, direct Firebase Auth is more likely for admin users.
    ```
    Consider if `@shopify/app-bridge-react` is needed if this admin panel is *not* an embedded Shopify app. If it's purely for your own administrative staff, direct Firebase Authentication for admins is the primary auth mechanism. Polaris can still be used for its UI components.

### 1.2. Configure Firebase Hosting
*   In your main `firebase.json` (project root), configure Firebase Hosting to serve the build output of your React application.
    ```json
    {
      "hosting": {
        "public": "admin-dashboard-ui/build", // Or "admin-dashboard-ui/dist" if using Vite
        "ignore": [
          "firebase.json",
          "**/.*",
          "**/node_modules/**"
        ],
        "rewrites": [
          {
            "source": "/api/admin/**", // Or just /admin/** if your function routes are prefixed
            "function": "api" // Or your specific Cloud Function name for admin APIs
          },
          {
            "source": "**",
            "destination": "/index.html"
          }
        ]
      }
      // ... other firebase configurations (functions, etc.)
    }
    ```
    *   The first rewrite directs API calls from the admin dashboard (e.g., to `/api/admin/...`) to your main Cloud Function. Adjust the `source` path and `function` name as needed.
    *   The second rewrite is standard for Single Page Applications (SPAs).

### 1.3. Firebase SDK for Frontend
*   Install the Firebase JavaScript SDK to interact with Firebase Authentication (for admin login) and potentially Firestore if admins need direct (but secure and rule-protected) read access for some data.
    ```bash
    npm install firebase
    ```
*   Initialize Firebase in your React app (e.g., in `src/firebaseConfig.js`):
    ```javascript
    // src/firebaseConfig.js
    import { initializeApp } from "firebase/app";
    import { getAuth } from "firebase/auth";
    // import { getFirestore } from "firebase/firestore"; // If needed

    const firebaseConfig = {
      apiKey: "YOUR_API_KEY",
      authDomain: "YOUR_AUTH_DOMAIN",
      projectId: "YOUR_PROJECT_ID",
      storageBucket: "YOUR_STORAGE_BUCKET",
      messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
      appId: "YOUR_APP_ID"
    };

    // Initialize Firebase
    const app = initializeApp(firebaseConfig);
    const auth = getAuth(app);
    // const db = getFirestore(app); // If direct Firestore access is planned

    export { auth /*, db */ };
    ```
    *   Store your Firebase project configuration securely, typically using environment variables (e.g., `REACT_APP_FIREBASE_API_KEY`).

## 2. Core UI Components & Layout

(Derived from Grok Outline Section 4.1)

### 2.1. AppProvider (Polaris)
*   Wrap your main React application component (e.g., `App.js`) with Shopify Polaris's `AppProvider` to enable Polaris styling and features.
    ```jsx
    // src/App.js
    import React from 'react';
    import { AppProvider } from '@shopify/polaris';
    import enTranslations from '@shopify/polaris/locales/en.json';
    import '@shopify/polaris/build/esm/styles.css';
    // ... other imports (Router, components)

    function App() {
      return (
        <AppProvider i18n={enTranslations}>
          {/* Your Router and main layout components go here */}
        </AppProvider>
      );
    }
    export default App;
    ```

### 2.2. Routing
*   Implement client-side routing using a library like `react-router-dom`.
    ```bash
    npm install react-router-dom
    ```
*   Define routes for login, dashboard overview, tenant management, subscription details, etc.

### 2.3. Main Layout
*   Create a main layout component that includes navigation (e.g., a Polaris `Navigation` component) and a content area for different pages.
*   Protected routes should only be accessible after admin authentication.

### 2.4. Authentication UI (Login Page)
*   Create a login page (` LoginPage.js `) using Polaris components (`TextField`, `Button`, `Card`).
*   Use Firebase Authentication SDK to handle admin sign-in (e.g., `signInWithEmailAndPassword`).
*   Store ID token upon successful login (e.g., in React context, state management, or even `localStorage` with care) to be sent with API requests to your backend.

    ```jsx
    // Example: Basic Login Functionality
    // In your LoginPage component
    // import { auth } from './firebaseConfig';
    // import { signInWithEmailAndPassword } from "firebase/auth";
    // ...
    // const handleLogin = async () => {
    //   try {
    //     const userCredential = await signInWithEmailAndPassword(auth, email, password);
    //     const idToken = await userCredential.user.getIdToken();
    //     // Store idToken, navigate to dashboard
    //   } catch (error) {
    //     // Handle login error
    //   }
    // };
    ```

## 3. Admin Dashboard Features UI & Logic

(Derived from Grok Outline Sections 4.2, 4.3, 4.4)

The admin dashboard will provide UIs for managing tenants, subscriptions, credentials (status, not values), and monitoring API usage. All data operations will be performed by making authenticated API calls to your Firebase Cloud Functions backend.

### 3.1. Tenant (Shop) Management UI
*   **Shop List Page:**
    *   Display a table (`DataTable` in Polaris) of all shops/tenants.
    *   Columns: Shop ID/Name, Plan, Subscription Status, Install Date, Actions (e.g., "View Details").
    *   Fetch data from `/api/admin/shops` backend endpoint.
*   **Shop Details Page:**
    *   Display detailed information for a selected shop.
    *   Allow admins to:
        *   View current plan and subscription status.
        *   Change a shop's plan (triggers backend call to update Stripe & Firestore).
        *   View list of stored credential types (e.g., "Google Ads Token: Set", "Gemini Key: Not Set").
        *   Potentially trigger actions like "Request Re-authentication" for a credential, which would involve backend logic.
        *   View API usage logs/summary for the shop.

### 3.2. Plan Configuration Interface
*   **Overview:** This interface allows admins to create, view, edit, and archive subscription plans, with changes reflected in both Firestore and Stripe (via backend API calls).
*   **Plan List Page:**
    *   UI: A Shopify Polaris `Page` with a `Card` containing a `DataTable` or `ResourceList`.
    *   Columns/Items: Plan Name, Price, Status (Active/Archived), Number of active subscribers (optional, might require extra backend logic to count), Actions (Edit, Archive).
    *   Functionality: Fetches and displays all plans from `/api/admin/plans`. A "Create New Plan" button navigates to the plan creation form.
*   **Plan Creation/Edit Form Page:**
    *   UI: A Shopify Polaris `Page` with a `Form` component and various `FormLayout` groups for input fields.
    *   Input Fields (using Polaris `TextField`, `Select`, `Checkbox`, `RangeSlider` or custom components for numeric inputs):
        *   `planName`: (String)
        *   `description`: (String, multiline)
        *   `price`: (Number, input for amount in major currency unit, backend converts to cents)
        *   `currency`: (Select, e.g., USD, EUR)
        *   `status`: (Select: Active, Archived) - Might be "Active" by default on create, and only editable to "Archived" on edit.
        *   **Feature Configuration Section:**
            *   `productProcessingLimit`: (Number TextField)
            *   `keywordGenerationLimit`: (Number TextField)
            *   `relatedQuestions.enabled`: (Polaris `Checkbox`) Label: "Enable 'Related Questions' Feature".
            *   `relatedQuestions.maxQuestionsToRetrieve`: (Polaris `TextField` for numbers, conditionally visible based on the checkbox) Label: "Max 'Related Questions' to Retrieve (per keyword/product)".
            *   (Add other dynamic feature fields as defined in the `plans` schema in `jules.infrastructure-setup.md`)
            *   *Note on 'Related Questions'*: Enabling this feature and setting `maxQuestionsToRetrieve` means that when this plan is active for a tenant, the backend will attempt to call the Python PAA service to fetch "People Also Ask" data, respecting the configured limit.
    *   Functionality:
        *   On submit (for create): POSTs data to `/api/admin/plans`.
        *   On submit (for edit): PUTs data to `/api/admin/plans/:planId`.
        *   Handles loading states and displays success/error messages from the API.
        *   Navigation back to the plan list page upon successful save/cancel.
*   **Archiving a Plan:**
    *   Functionality: Typically an action on the Plan List Page or Edit Page.
    *   UI: Requires a confirmation `Modal` (Polaris `Modal`) before proceeding.
    *   Action: Calls `DELETE /api/admin/plans/:planId` (which archives the plan in backend and Stripe). Updates the list upon success.
*   **State Management:** Briefly mention that local component state (`useState`, `useReducer`) or a more robust state management solution (React Context, Zustand, Redux) will be needed to manage form data, loading states, and API responses.
*   **API Interaction:** Reiterate that all actions (CRUD operations for plans) involve making authenticated API calls to the backend endpoints detailed in `jules.backend-development.md`.

### 3.2.1. Promotions & Discount Code Awareness (Conceptual)

*   **Purpose:** While discount codes (Coupons/Promotion Codes) are primarily created and managed directly in the Stripe Dashboard (as per `docs/jules/admin-dashboard/jules.infrastructure-setup.md`), the admin dashboard can provide awareness of these codes for internal tracking, marketing reference, and support purposes.
*   **UI (Conceptual):**
    *   A simple view, potentially on a dedicated "Promotions" page or a `Card` within a relevant settings area.
    *   It might feature a Polaris `ResourceList` or `DataTable` to list significant or commonly used discount codes.
    *   **Data Source (V1 - Manual/Internal Note-Keeping):**
        *   For V1, this list would **not** be populated by direct, real-time Stripe API calls from the admin frontend to fetch *all* coupons (which can be extensive and paginated).
        *   Instead, it could be a manually curated list. Admins could copy-paste important active codes from Stripe into a simple internal note-keeping feature or a basic CRUD interface within this section of the admin dashboard (e.g., fields for Code, Discount Info, Notes). This data would be stored in a simple Firestore collection accessible only to admins.
    *   **Data Source (V2 - Future Enhancement):**
        *   A backend function could periodically fetch a curated subset of active coupon/promotion code data from Stripe (e.g., those with specific metadata or recent activity) and cache it in Firestore for display in this admin panel section. This would be more complex due to Stripe API pagination, data volume considerations, and caching logic.
    *   **Displayed Information (for V1 manual list example):**
        *   `Code`: e.g., "SUMMER2024"
        *   `Discount Info`: e.g., "20% off for 3 months"
        *   `Internal Note`: e.g., "Summer campaign, expires Sept 1st"
        *   `Status`: (Manually set, e.g., Active, Expired)
    *   **Link to Stripe:** A prominent link or button: "Manage All Discount Codes in Stripe Dashboard" which redirects the admin to the appropriate section in their Stripe account.
*   **No Create/Edit via Admin UI (V1):** It's important to reiterate that for V1, there is **no functionality to create, edit, or manage the lifecycle of discount codes directly from this Content Agent admin dashboard**. All such management tasks are performed in the Stripe Dashboard. This section is for awareness and internal reference only.

### 3.3. Subscription Management UI
*   Integrated into the **Shop Details Page** or a separate section. This section now refers to managing a *tenant's subscription to a plan*, not defining the plans themselves.
*   **Functionality:**
    *   View current subscription details (plan, price, status, next billing date) fetched from your backend (which gets it from Stripe/Firestore).
    *   Allow admins to manually change a tenant's plan. This action will:
        *   Call a backend endpoint (e.g., `/api/admin/shops/:shop_id/subscriptions`).
        *   The backend then interacts with Stripe to update the subscription and updates Firestore accordingly.
    *   Display billing history (if Stripe API provides this easily and backend exposes it).
    *   Apply/view discount codes (if admin has this capability).

### 3.4. Credential Management UI (Status & Actions)
*   Typically on the **Shop Details Page**.
*   **Functionality:**
    *   **Display Status:** Show which credential types are configured for the tenant (e.g., "Google Ads Refresh Token: Present", "Gemini AI API Key: Not Set"). **Do NOT display the actual encrypted credentials.**
    *   **Actions:**
        *   **Trigger Re-OAuth:** For OAuth-based credentials like Google Ads, a button to "Invalidate & Request Re-authentication" might be useful. This tells the backend to mark the current token as needing refresh and possibly notify the tenant.
        *   **Delete Credential:** Allow an admin to delete a specific stored credential for a tenant (with strong confirmation).
    *   **Toggle Developer-Provided Credentials:** If a tenant is on a plan that allows choosing between their own vs. developer-provided keys for certain APIs (e.g., Gemini AI):
        *   A toggle/select list for the admin to change this setting.
        *   This action calls a backend endpoint. The backend updates Firestore (`shops/{shop_id}.developerCredentialsOptIn`) and potentially triggers a Stripe subscription update if this change affects billing (e.g., moves to a different Stripe Price ID).

### 3.5. API Usage Monitoring UI
*   On the **Shop Details Page** or a dedicated "API Usage" section.
*   **Functionality:**
    *   Display API usage for a selected tenant (fetched from `/api/admin/shops/:shop_id/usage` or similar backend endpoint).
    *   Show usage against quotas for different API types (e.g., "Gemini AI Calls: 350/500 this month").
    *   Provide charts or summaries of usage over time.
    *   Admins might have tools to view global API usage across all tenants (from `/api/admin/usage_summary`).

### 3.6. Making Authenticated API Calls to Backend
*   All `fetch` or `axios` calls from the React admin dashboard to your backend API (Cloud Functions) must include the Firebase Auth ID token in the `Authorization` header.
    ```javascript
    // Example API call utility
    // async function adminApiFetch(path, options = {}, idToken) {
    //   const defaultHeaders = {
    //     'Authorization': `Bearer ${idToken}`,
    //     'Content-Type': 'application/json',
    //   };
    //   const config = {
    //     ...options,
    //     headers: {
    //       ...defaultHeaders,
    //       ...options.headers,
    //     },
    //   };
    //   const response = await fetch(`/api/admin${path}`, config); // Assuming /api/admin prefix from hosting rewrite
    //   if (!response.ok) {
    //     const errorData = await response.json().catch(() => ({ message: response.statusText }));
    //     throw new Error(errorData.message || `API request failed with status ${response.status}`);
    //   }
    //   return response.json();
    // }

    // Usage:
    // const shops = await adminApiFetch('/shops', {}, firebaseUser.idToken);
    ```

This frontend structure, built with React and Shopify Polaris, and interacting with a secure Firebase backend, will provide administrators with the tools they need to manage the application and its tenants.

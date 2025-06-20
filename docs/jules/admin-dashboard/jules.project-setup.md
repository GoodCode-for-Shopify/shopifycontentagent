# Admin Dashboard: Project Setup

This document outlines the initial project setup for the Shopify App's Administrative Dashboard. It covers requirements definition and development environment configuration, ensuring alignment with our core technology stack (Firebase, Firestore, Cloud Functions, React/Polaris).

## 1. Requirements Definition

(Derived from Grok "Shopify-App-Admin-Dashboard-Instructional-Outline.md" Section 1.1 and "Shopify-App-Admin-Dashboard-Guide.md" Section 1 & 2)

Before diving into development, it's crucial to clearly define the project's requirements.

### 1.1. Freemium Model & Pricing Strategy
The application will operate on a freemium model, offering several tiers to cater to different user needs. Stripe will be used for subscription billing and discount code management.

*   **Free Plan:**
    *   Limited features (e.g., basic reporting, a specific cap on AI usage or API calls per month).
    *   Requires tenants to provide all their own third-party API credentials (e.g., Google Ads Refresh Token, Customer ID, Google API Key, Gemini AI API Key).
    *   No access to developer-provided/managed API credentials.
*   **Basic Plan (e.g., $29/month):**
    *   Expanded feature set (e.g., more detailed reporting, higher API call/AI usage limits).
    *   Option for tenants to provide their own credentials.
    *   Option for tenants to use developer-provided credentials for select APIs (e.g., Google API Key, Gemini AI API Key) for an additional fee, integrated into the Stripe price for this plan variation.
    *   Google Ads will always require tenant-provided Refresh Token and Customer ID due to its account-specific OAuth nature.
    *   Support for discount codes (e.g., 10% off).
*   **Pro Plan (e.g., $99/month):**
    *   Full feature set (e.g., automation tools, highest API call/AI usage limits).
    *   Same credential options as the Basic Plan, potentially with higher associated fees for using developer-provided credentials.
    *   Support for more significant discount codes (e.g., 20% off).
*   **Enterprise Plan (Custom Pricing):**
    *   Unlimited or very high usage features, dedicated support, custom API quotas.
    *   Flexible credential options, potentially including fully managed credentials where feasible and secure.

**Key Considerations for Credential Options:**
*   **Tenant-Provided:** Reduces developer's API costs and quota usage. Default for Free plan.
*   **Developer-Provided:** Simplifies setup for some users on paid plans but increases developer's API costs and quota management responsibilities. This option must be carefully priced and monitored.

### 1.2. API Integrations
The application will integrate with the following third-party APIs. All server-side interactions with these APIs will be handled by Cloud Functions.
*   **Google Ads API:** For managing and reporting on Google Ads campaigns. Requires tenant-specific OAuth.
*   **Gemini AI API (or equivalent Google AI Service):** For leveraging AI capabilities within the app.
*   **Stripe API:** For subscription management, payment processing, and discount codes.

### 1.3. Security Requirements
Security is a top priority. The following must be addressed (refer to `docs/jules.authority.md` and `docs/jules/serverside-setup.md` for specifics on our stack):
*   **Data Encryption:** Sensitive data, especially API credentials stored in Firestore, must be encrypted at rest (AES-256).
*   **Authentication:**
    *   **Admin Users:** Secure authentication for administrators accessing the dashboard (e.g., Firebase Authentication with email/password or Google Sign-In).
    *   **Shopify App Users (for the future end-user app):** Authenticated via Shopify session tokens (JWTs).
*   **Authorization:** Role-based access control (RBAC) for admin users (e.g., super admin, support admin) if applicable, managed via Firebase Auth custom claims.
*   **Secure Communication:** HTTPS for all data in transit (default with Firebase Hosting and Cloud Functions).
*   **Compliance:** Adherence to data protection regulations like GDPR/CCPA. Secure handling of all Personally Identifiable Information (PII).
*   **Third-Party API Key Security:** Secure storage and handling of both tenant-provided and developer-managed API keys, as detailed in `docs/jules/serverside-setup.md`.

## 2. Development Environment Setup

(Derived from Grok Outline Section 1.2, adapted for Firebase stack)

A consistent and correctly configured development environment is essential.

### 2.1. Core Tools Installation
*   **Node.js and npm:**
    *   Install Node.js (LTS version, e.g., 18.x or 20.x, compatible with Firebase Functions 2nd Gen). Npm is included with Node.js.
    *   Verify with `node -v` and `npm -v`.
*   **Firebase CLI:**
    *   Install globally: `npm install -g firebase-tools`
    *   Log in: `firebase login`
    *   Used for initializing Firebase projects, deploying functions and hosting, managing emulators, and environment configuration.
*   **Google Cloud CLI (gcloud CLI):**
    *   Install from [Google Cloud SDK documentation](https://cloud.google.com/sdk/docs/install).
    *   Initialize: `gcloud init`
    *   Log in: `gcloud auth login`
    *   Useful for managing underlying Google Cloud resources if needed.

### 2.2. Version Control
*   **Git:** Install Git for version control.
*   **GitHub (or similar):** Set up a remote repository for collaboration and backup.
    *   Initialize a Git repository in your project root: `git init`
    *   Make initial commits and push to the remote repository.

### 2.3. Integrated Development Environment (IDE)
*   **VS Code (Recommended):** A popular choice with excellent support for JavaScript, Node.js, and Firebase.
*   **Recommended VS Code Extensions:**
    *   **ESLint:** For code linting to enforce code quality and catch errors early. (Configure according to project standards).
    *   **Prettier - Code formatter:** For consistent code formatting.
    *   **DotENV:** For syntax highlighting of `.env` files (though Firebase uses a different mechanism for environment configuration for deployed functions).
    *   Consider extensions for React/JSX if not already part of your standard setup.

### 2.4. Project Directory Structure (Example)
A typical Firebase project structure for this application might look like:

```
shopify-admin-app/
├── docs/                     # Project documentation
│   ├── jules.authority.md
│   ├── jules/
│   │   ├── serverside-setup.md
│   │   └── admin-dashboard/
│   │       └── jules.project-setup.md
│   │       └── ... (other admin dashboard docs)
│   └── grok/
│       └── ... (original Grok documents)
├── functions/                # Firebase Cloud Functions (Backend: Node.js/Express)
│   ├── src/
│   │   ├── index.js          # Main Express app and function definitions
│   │   ├── encryption.js     # Encryption utilities
│   │   └── ... (other backend modules for Stripe, Google APIs, etc.)
│   ├── package.json
│   ├── .eslintrc.js
│   └── .runtimeconfig.json   # For local emulator secrets (add to .gitignore)
├── admin-dashboard-ui/       # React/Polaris Admin Dashboard Frontend
│   ├── public/
│   ├── src/
│   │   ├── index.js          # React app entry point
│   │   ├── App.js
│   │   ├── components/
│   │   └── pages/
│   ├── package.json
│   └── ...
├── .firebaserc               # Firebase project association
├── firebase.json             # Firebase deployment rules (hosting, functions)
├── .gitignore
└── README.md
```
*   The `admin-dashboard-ui/` directory will house the React/Polaris frontend for the admin dashboard.
*   The `functions/` directory contains the Firebase Cloud Functions backend.
*   This structure promotes separation of concerns between the backend, frontend, and documentation.

With these foundational elements defined and the development environment configured, you're ready to proceed with setting up the infrastructure.

# Environments and CI/CD Strategy for Content Agent

## 1. Introduction

### 1.1. Purpose
This document outlines the strategy for managing different deployment environments and implementing Continuous Integration/Continuous Deployment (CI/CD) for the "Content Agent" project. This includes the Admin Dashboard, the Shopify App (frontend and backend Cloud Functions), and the Python PAA service.

A robust environment and CI/CD strategy is crucial for ensuring code quality, deployment reliability, and efficient development workflows.

### 1.2. Goals
*   Define distinct environments for development, staging, and production.
*   Automate build, test, and deployment processes.
*   Ensure consistency and reliability of deployments.
*   Enable rapid iteration and feedback loops.
*   Manage environment-specific configurations securely.
*   Minimize deployment-related risks.

## 2. Environment Strategy

We will utilize multiple Firebase projects to achieve true environment isolation, especially for Firestore databases, Cloud Functions, and Hosting. Using Firebase project aliases will be key.

### 2.1. Proposed Environments

1.  **Development (`dev`)**
    *   **Firebase Project:** A dedicated Firebase project (e.g., `content-agent-dev`).
    *   **Purpose:** Used by individual developers for daily development and testing of new features. Data can be volatile.
    *   **Data:** Sample data, potentially seeded. Firestore rules might be more permissive for easier debugging.
    *   **Third-Party Services (Stripe, Google APIs):** Use test modes/keys. Stripe in test mode. Google APIs might use developer accounts or sandboxed projects.
    *   **Deployment:** Developers can deploy to this environment from their feature branches or personal forks (manual or CI triggered). Firebase Local Emulator Suite should be heavily used locally before deploying here.

2.  **Staging (`staging`)**
    *   **Firebase Project:** A dedicated Firebase project (e.g., `content-agent-staging`).
    *   **Purpose:** Pre-production environment for testing integrated features, running full end-to-end tests, and UAT (User Acceptance Testing) by the project team or beta testers.
    *   **Data:** A sanitized, anonymized subset of production data, or more stable, curated test data. Firestore rules should mirror production.
    *   **Third-Party Services (Stripe, Google APIs):** Use test modes/keys. Stripe in test mode, but with configurations mirroring production plans.
    *   **Deployment:** Automated deployments from a specific branch (e.g., `develop` or `release/candidate`) via CI/CD pipeline after all automated tests pass.

3.  **Production (`prod`)**
    *   **Firebase Project:** The live Firebase project (e.g., `content-agent-prod` or the main `content-agent` project ID).
    *   **Purpose:** The live environment used by actual customers (Shopify merchants and admins).
    *   **Data:** Live customer data. Firestore rules are enforced strictly.
    *   **Third-Party Services (Stripe, Google APIs):** Use live modes/keys.
    *   **Deployment:** Automated or manually-triggered (with approval steps) deployments from the main/master branch via CI/CD pipeline, only after successful staging validation.

### 2.2. Firebase Project Aliases
*   The Firebase CLI allows using project aliases to switch between Firebase projects easily.
    *   `firebase use dev`
    *   `firebase use staging`
    *   `firebase use prod`
*   This is crucial for deploying the correct code and configuration to the intended environment.

### 2.3. Configuration Management
*   **Environment Variables:** Firebase Cloud Functions support environment variables. These will be used to store environment-specific configurations:
    *   API keys for third-party services (Stripe, Google, Gemini AI).
    *   Python PAA service URL (if different per environment).
    *   Logging levels.
    *   Frontend API base URLs (if not just `/api/`).
    *   Shopify App API Key and Secret (these might be the same for dev/staging if using a single dev store, but production app will have its own).
    *   Firebase function configuration can be set using `firebase functions:config:set mykey.value="my value"` and deployed with `firebase deploy --only functions`. These should be managed per Firebase project alias.
*   **Configuration Files (Frontend):** React frontends can have environment-specific `.env` files (e.g., `.env.development`, `.env.staging`, `.env.production`) managed by the build process (e.g., Vite or Create React App). These should primarily contain non-sensitive build-time configurations. Sensitive keys for frontend-facing Firebase SDK initialization should be handled carefully, possibly fetched from a secure backend endpoint if needed post-build, or by ensuring Firebase Hosting config is environment-specific.
*   **Secure Storage:** Sensitive keys (like Stripe secret keys, Google API private keys) must be stored securely using Firebase environment configuration (as mentioned above) or Google Secret Manager, accessible only by the Cloud Functions. They should NOT be committed to the Git repository.

## 3. CI/CD Strategy

We will use a Git-based workflow with a CI/CD platform like GitHub Actions, GitLab CI, or Google Cloud Build.

### 3.1. Git Branching Strategy
*   **`main` (or `master`):** Represents the production-ready code. Merges to this branch trigger production deployments (or manual approval for production deployment).
*   **`develop`:** Integration branch. Features are merged here from feature branches. Merges to `develop` trigger deployments to the `staging` environment.
*   **`feature/*` (or `feat/*`):** Individual feature branches created from `develop`. Developers work here. Pull Requests (PRs) are made from feature branches to `develop`.
*   **`release/*` (Optional):** For preparing production releases. Branched from `develop` when ready for a release cycle. Only bug fixes are merged here. Once stable, merged to `main` and `develop`.
*   **`hotfix/*`:** For critical bug fixes in production. Branched from `main`, fixed, then merged back to `main` and `develop`.

### 3.2. CI/CD Pipeline - Key Stages (Conceptual)

**For each component (Admin Backend, Admin Frontend, Shopify App Backend, Shopify App Frontend, Python PAA Service):**

1.  **Trigger:** Commit/merge to specific branches (`feature/*` PRs, `develop`, `main`).
2.  **Lint & Code Style Check:** Run linters (ESLint, Flake8 for Python) and code formatters (Prettier).
3.  **Unit Tests:** Run unit tests for backend logic and frontend components (Jest, PyTest). Code coverage checks can be enforced.
4.  **Integration Tests (Backend):** Run integration tests that might involve emulated services or live test instances of Firebase services (Firestore, Auth).
5.  **Build:**
    *   Frontend: `npm run build` (or equivalent) to generate static assets.
    *   Backend (Node.js/Python): Usually no explicit "build" step for Firebase Functions other than packaging, but transpilation (e.g., TypeScript to JavaScript) would happen here.
6.  **Security Scans (Optional but Recommended):**
    *   Static Application Security Testing (SAST) tools.
    *   Dependency vulnerability scanning (e.g., `npm audit`, `pip-audit`).
7.  **Deployment:**
    *   **To `dev` (from feature branches, optional):**
        *   `firebase use dev`
        *   `firebase deploy --only functions:yourFunctionName,hosting:yourTarget` (for specific changes)
    *   **To `staging` (from `develop` branch):**
        *   `firebase use staging`
        *   Set staging environment variables for Functions.
        *   `firebase deploy --only functions,hosting,firestore:rules,storage:rules` (deploy all relevant parts; adjust targets as needed per component, e.g., Python service might only be `functions`).
    *   **To `prod` (from `main` branch, potentially with manual approval step):**
        *   `firebase use prod`
        *   Set production environment variables for Functions.
        *   `firebase deploy --only functions,hosting,firestore:rules,storage:rules` (adjust targets as needed per component).
8.  **Post-Deployment Smoke Tests/Health Checks (Optional but Recommended):**
    *   Simple automated tests that verify the deployed application is running and key endpoints are responsive.
9.  **Notifications:** Notify team on Slack/email about build status, test results, and deployment outcomes.

### 3.3. Specific Component CI/CD Considerations

*   **Admin Dashboard Frontend (React/Polaris):**
    *   Pipeline: Lint -> Unit Tests (Jest/RTL) -> Build -> Deploy to Firebase Hosting (target: `admin-dashboard`).
*   **Admin Dashboard Backend (Node.js Cloud Functions):**
    *   Pipeline: Lint -> Unit Tests (Jest/Mocha) -> Integration Tests -> Deploy Functions.
*   **Shopify App Frontend (React/Polaris):**
    *   Pipeline: Lint -> Unit Tests (Jest/RTL) -> Build -> Deploy to Firebase Hosting (target: `shopify-app`).
*   **Shopify App Backend (Node.js Cloud Functions, including Shopify OAuth handlers):**
    *   Pipeline: Lint -> Unit Tests (Jest/Mocha) -> Integration Tests -> Deploy Functions.
*   **Python PAA Service (Python Cloud Function/Cloud Run):**
    *   Pipeline: Lint (Flake8) -> Unit Tests (PyTest) -> Package -> Deploy Function/Service.

### 3.4. Managing Firebase Deployments
*   **Atomic Deployments:** Firebase Hosting deployments are atomic. Cloud Function deployments update functions individually or as a group.
*   **Firestore Rules & Indexes:**
    *   `firestore.rules` and `firestore.indexes.json` should be part of the Git repository.
    *   Deploy them via `firebase deploy --only firestore`.
*   **Firebase Hosting Rewrites:**
    *   `firebase.json` (containing hosting configurations, rewrites, and targets) should be version-controlled.
    *   Different `firebase.json` configurations might be needed per environment if targets or function names differ significantly, or conditional logic within the CI/CD script can select parts of a master `firebase.json`. For simplicity, separate Firebase projects (dev, staging, prod) are recommended, each with its own `firebase.json` (or a base one selected by `firebase use <alias>`). The `.firebaserc` file will map aliases to project IDs and is crucial.

## 4. Database Management (Firestore)
*   **Schema Migrations:** Firestore is schema-less, but data structure changes might require data migrations.
    *   For minor changes, scripts can be written and run manually or via a CI/CD job against the appropriate environment.
    *   For larger migrations, a more robust migration framework might be considered (though often not needed for Firestore initially).
*   **Seeding Data:**
    *   `dev` environment: Scripts to seed basic data (e.g., sample plans, dummy users/shops).
    *   `staging` environment: Scripts to anonymize and import a subset of production data, or more comprehensive seed data.

## 5. Shopify App Configuration
*   **App Setup in Partner Dashboard:**
    *   The App URL, Allowed Redirection URL(s) will need to be configured for each environment (e.g., `dev.yourapp.com`, `staging.yourapp.com`, `prod.yourapp.com`).
    *   This might mean creating separate Shopify app listings in the Partner Dashboard for `dev` and `staging` if they need distinct API keys or URLs not manageable by a single app configuration. Often, a single dev store is used for dev/staging with a single app config, and production has its own.
*   **API Keys:** The Shopify App API Key and Secret will be environment variables for the backend functions.

## 6. Security in CI/CD
*   **Secret Management:**
    *   Use the CI/CD platform's built-in secret management for storing Firebase tokens, API keys for third-party services, etc. Do not hardcode secrets in CI/CD scripts.
*   **Service Accounts & Permissions:**
    *   Use dedicated Google Cloud service accounts for CI/CD with the least privilege necessary to perform deployments to Firebase/Google Cloud.
*   **Branch Protection:** Protect `develop` and `main` branches in Git (e.g., require PR reviews, passing status checks before merging).

## 7. Custom Domain Strategy for Environments

Setting up custom domains for each environment enhances professionalism and makes access easier. This requires configuring DNS records for your domains to point to Firebase Hosting or other relevant services.

### 7.1. Production Domains
*   **Admin App & Potential Marketing Site:**
    *   **Domain:** `shopifycontentagent.com` (and `www.shopifycontentagent.com`).
    *   **Service:** Points to Firebase Hosting for the `prod` Firebase project, serving the Admin Dashboard UI and any marketing/landing pages.
*   **API Backend (Cloud Functions):**
    *   **Domain:** `api.shopifycontentagent.by.goodcode.ca` (or a subdomain like `api.shopifycontentagent.com` if `shopifycontentagent.com` is managed where Cloudflare or a similar full proxy/DNS manager can also manage subdomains for the API).
    *   **Service:** This domain would be mapped to the production Cloud Functions. This can be achieved via:
        *   Firebase Hosting rewrites: If the main `shopifycontentagent.com` is on Firebase Hosting, a rewrite rule can direct `api.shopifycontentagent.com/*` to the main Cloud Function URL.
        *   Google Cloud Load Balancer: For more complex routing, SSL handling, or if functions are in different regions, a load balancer can be configured with a custom domain frontend.
        *   The `by.goodcode.ca` subdomain implies it might be managed under a different primary domain, which is also fine.

### 7.2. Staging Domains
*   **Admin App (Staging):**
    *   **Domain:** `staging.shopifycontentagent.com` (or `admin-staging.shopifycontentagent.com`).
    *   **Service:** Points to Firebase Hosting for the `content-agent-staging` Firebase project.
*   **API Backend (Staging):**
    *   **Domain:** `staging.api.shopifycontentagent.by.goodcode.ca` (or `api-staging.shopifycontentagent.com`).
    *   **Service:** Mapped to the Cloud Functions in the `content-agent-staging` Firebase project, using similar methods as production (Firebase Hosting rewrites on `staging.shopifycontentagent.com` or a separate Load Balancer setup for the staging project).

### 7.3. Development Domains
*   **Admin App (Dev):**
    *   **Domain:** `dev.shopifycontentagent.com` (or `admin-dev.shopifycontentagent.com`) or primarily `localhost:<port>` when using Firebase Local Emulator Suite.
    *   **Service (if custom domain used):** Points to Firebase Hosting for the `content-agent-dev` Firebase project.
*   **API Backend (Dev):**
    *   **Domain:** `dev.api.shopifycontentagent.by.goodcode.ca` (or `api-dev.shopifycontentagent.com`) or `localhost:<port>` (via Firebase Local Emulator Suite).
    *   **Service (if custom domain used):** Mapped to Cloud Functions in the `content-agent-dev` Firebase project.
    *   Firebase Local Emulators are highly recommended for most development to avoid needing many cloud deployments.

### 7.4. Shopify App URL Considerations
The embedded Shopify App's frontend is also hosted on Firebase Hosting and will require environment-specific URLs.
*   **Production:** e.g., `app.shopifycontentagent.com` (or the default `web.app` URL if a custom domain isn't a priority for the embedded app itself). This URL is configured in the Shopify Partner Dashboard for the production app listing.
*   **Staging:** e.g., `app-staging.shopifycontentagent.com` (or the staging Firebase project's `web.app` URL). This would ideally be configured in a separate "staging" or "test" app configuration in the Shopify Partner Dashboard, pointing to a development or test Shopify store.
*   **Development:**
    *   Typically uses `localhost` tunneled via a service like `ngrok` or the Shopify CLI's built-in tunneling (`npm run dev` from Shopify CLI app template often handles this). The ngrok/Shopify CLI URL is then used in a dev app configuration in the Partner Dashboard, pointing to a development store.
    *   Alternatively, if deploying feature branches to the `dev` Firebase project, the `content-agent-dev` Firebase Hosting URL for the Shopify app target could be used with a dev store.

Managing these custom domains requires DNS configuration (A, CNAME records) and provisioning SSL certificates (Firebase Hosting does this automatically for connected custom domains).

This strategy provides a solid foundation for managing environments and automating deployments, aiming for a balance between agility and stability. It will need to be adapted and refined as the project evolves.

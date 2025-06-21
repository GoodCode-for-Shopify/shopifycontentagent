# Content Agent: Development Roadmap

## 1. Introduction

**Important Disclaimer:** All time estimates in this document are very rough, placeholder estimates. They are intended to give a relative sense of effort and for high-level planning discussion only. Actual development time will vary significantly based on developer experience, team velocity, specific technical challenges encountered, and the final depth of features. These estimates must be reviewed and adjusted by a project manager or development team before being used for formal planning or commitments.

This document outlines a proposed development roadmap for the "Content Agent" project. It prioritizes features into phases to deliver core functionality first, followed by enhancements and more advanced features. This roadmap considers dependencies between the Admin Dashboard, the Shopify App, and backend services.

**Core Philosophy:**
*   **Admin Dashboard First (for core config):** Critical Admin Dashboard features for plan management and system configuration need to be in place to support the Shopify App.
*   **MVP for Shopify App:** Deliver a functional Shopify App with core content generation capabilities for a single plan structure first.
*   **Iterative Enhancements:** Build upon the MVP with more features, plan flexibility, and V2 enhancements as outlined in various TDDs.

## 2. Phase 0: Foundational Setup & Project Initialization

**Duration:** 1-2 Sprints

**Goals:** Establish core infrastructure, project structures, and basic authentication.

*   **General:**
    *   [ ] Finalize core technology choices (already largely done: Firebase, Node.js, React, Python). (e.g., 1-2h)
    *   [ ] Set up Git repository, CI/CD pipelines (basic). (e.g., 4-8h)
    *   [ ] Define initial `jules.authority.md` directives. (e.g., 1-2h)
*   **Firebase Project:**
    *   [ ] Initialize Firebase project (Firestore, Functions, Hosting, Auth). (e.g., 2-4h)
    *   [ ] Configure Blaze plan for outbound requests. (e.g., 0.5-1h)
*   **Admin Dashboard (Shell & Auth):**
    *   [ ] Initialize React/Polaris project for Admin Dashboard. (e.g., 2-4h)
    *   [ ] Implement Firebase Authentication for admin users (Email/Password, Google Sign-In). (e.g., 1-2d)
    *   [ ] Basic layout/navigation for Admin Dashboard. (e.g., 4-8h)
    *   [ ] `adminAuthMiddleware` for backend. (e.g., 4-8h)
*   **Shopify App (Shell & Auth):**
    *   [ ] Initialize Shopify App project using Shopify CLI (Node.js backend, React frontend). (e.g., 2-4h)
    *   [ ] Implement Shopify OAuth flow for app installation and access token storage (encrypted in Firestore). (e.g., 1-2d)
    *   [ ] Implement JWT verification middleware for Shopify App backend requests. (e.g., 4-8h)
    *   [ ] Basic embedded app structure with Polaris `AppProvider`. (e.g., 2-4h)
*   **Shared Backend Logic:**
    *   [ ] Implement credential encryption/decryption utilities (Node.js `crypto`). (e.g., 4-8h)
    *   [ ] Set up Global API Error Handling Strategy (`tech-design.api-error-handling-strategy.md`). Apply to initial backend endpoints. (e.g., 1-2d)

## 3. Phase 1: MVP - Core Functionality & Single Plan Structure

**Duration:** 3-4 Sprints

**Goals:** Functional Admin Dashboard for basic plan setup (non-dynamic initially, or very basic dynamic), and a Shopify App that can generate articles based on a single, predefined plan structure.

*   **Admin Dashboard:**
    *   [ ] **Plan Management (Simplified V1):**
        *   UI to define features/limits for a few predefined plan tiers (e.g., Free, Basic, Pro). Initially, these might not be fully Stripe-integrated or as dynamic as in the `tech-design.dynamic-plans-stripe.md`. Focus on storing these limits in Firestore. (e.g., 2-3d)
        *   Ensure the structure of these simplified plans in Firestore is forward-compatible with the full dynamic plan schema to minimize migration later. (e.g., 4-8h)
        *   Reference: `jules.admin-dashboard.infrastructure-setup.md` (initial `plans` schema).
    *   [ ] **Tenant Management (Basic):** View list of installed shops, their current (manually assigned) plan, and basic status. (e.g., 1-2d)
    *   [ ] UI for managing global API keys (e.g., developer's Gemini, Google Ads API keys if used by default for some tiers). (e.g., 1-2d)
*   **Shopify App Backend:**
    *   [ ] API endpoint to fetch plan features/limits for the current shop. (e.g., 4-8h)
    *   [ ] API endpoints for storing and retrieving user-provided API credentials (e.g., Gemini, Google Ads) - encrypted. (e.g., 1-2d)
    *   [ ] **Python PAA Service (V1):**
        *   Implement and deploy V1 as per `jules.python-paa-service-setup.md`. (e.g., 2-3d)
        *   Node.js backend integration to call the PAA service. (e.g., 4-8h)
    *   [ ] **Core Content Generation APIs (Single Article Flow):**
        *   Product selection/validation. (e.g., 4-8h)
        *   Keyword generation (using Gemini, Google Ads). (e.g., 1-2d)
        *   PAA question fetching (via Python PAA service). (e.g., 2-4h)
        *   Article outline generation (Gemini). (e.g., 4-8h)
        *   Article text construction (Gemini, with watermarking). (e.g., 1-2d)
        *   (Optional for MVP, can defer) AI Image generation (basic, one provider). (e.g., 2-3d)
        *   Commit to Shopify blog (as draft, with author tag). (e.g., 1-2d)
        *   Download as MD/TXT. (e.g., 4-8h)
*   **Shopify App Frontend:**
    *   [ ] **Onboarding:** Terms acceptance, display of available plan features (fetched from Firestore based on simplified plan setup, but no active selection/Stripe integration yet), credential input walkthrough for user-provided keys. Reference: `jules.authentication-and-onboarding.md`. (e.g., 2-3d)
    *   [ ] **Main UI (Single Article Flow):**
        *   Implement the multi-step form for: Product Selection, Keyword Gen, Related Questions, Outline Gen, Text Construction, (Optional Imaging), Commit/Export. (e.g., 3-5d)
        *   This will initially NOT use the "Article Workspace" from `tech-design.multi-article-flow.md` but will be a simpler, single-pass flow.
    *   [ ] Settings page for users to update their API credentials. (e.g., 1-2d)
*   **General:**
    *   [ ] Basic end-to-end testing of the single article flow. (e.g., 2-3d)

## 4. Phase 2: Dynamic Plans & Multi-Article Flow Introduction

**Duration:** 3-4 Sprints

**Goals:** Implement full dynamic plan management in Admin Dashboard with Stripe integration. Introduce the multi-article processing flow in the Shopify App.

*   **Admin Dashboard:**
    *   [ ] **Full Dynamic Plan Management with Stripe Integration:**
        *   Implement as per `tech-design.dynamic-plans-stripe.md`. (e.g., 3-5d)
        *   UI for CRUD operations on plans (name, features, limits, pricing, interval). (e.g., 2-3d)
        *   Backend logic to create/update Stripe Products & Prices. (e.g., 2-3d)
        *   Archive plans. (e.g., 4-8h)
    *   [ ] **Subscription Management (Basic):** View subscriptions linked to shops (data from Stripe via webhooks or direct API calls). (e.g., 2-3d)
    *   [ ] UI for managing Stripe API keys and webhook endpoints. (e.g., 1-2d)
*   **Shopify App Backend:**
    *   [ ] **Stripe Integration:**
        *   Endpoints for creating Stripe Checkout Sessions. (e.g., 1-2d)
        *   Webhooks for `checkout.session.completed`, `customer.subscription.created/updated/deleted` to update Firestore `shops` and `subscriptions` collections. (e.g., 2-3d)
    *   [ ] API endpoint `GET /api/plans-listing` for Shopify app to fetch publicly visible, active plans from Firestore. (e.g., 4-8h)
    *   [ ] **Multi-Article Flow Backend Support:**
        *   Implement `POST /api/content/initiate-multi-article-job` as per `tech-design.multi-article-flow.md`. (e.g., 1-2d)
        *   Implement `GET /api/content/multi-article-job/{jobId}`. (e.g., 4-8h)
        *   Ensure single-article APIs (`construct-article`, `generate-image`, `commit-to-shopify`, etc.) accept `articleId` and update `job_articles` collection. (e.g., 1-2d)
*   **Shopify App Frontend:**
    *   [ ] **Plan Selection during Onboarding:** Fetch plans from `GET /api/plans-listing`, allow user to select, redirect to Stripe Checkout for paid plans. (e.g., 2-3d)
    *   [ ] **Subscription Management UI (Basic):** Allow user to see current plan, possibly link to Stripe Customer Portal (if configured). (e.g., 1-2d)
    *   [ ] **Multi-Article Flow UI Implementation:**
        *   Implement `MultiArticleOrchestrator`, `ArticleWorkspace`, `SingleArticleProcessor` components as per `tech-design.multi-article-flow.md`. (e.g., 3-5d)
        *   Adapt initial steps (Product Selection, Keywords, Questions) to support batch input. (e.g., 1-2d)
        *   "Generate All Outlines" calls `initiate-multi-article-job`. (e.g., 2-4h)
        *   Article Workspace displays articles from job, allows selection for `SingleArticleProcessor`. (e.g., 1-2d)
        *   `SingleArticleProcessor` handles one article then returns to Workspace. (e.g., 4-8h)
*   **General:**
    *   [ ] Testing for dynamic plan changes, Stripe subscriptions, and multi-article flow. (e.g., 3-5d)

## 5. Phase 3: Enhancements & V2 Features

**Duration:** Ongoing / Multiple Sprints

**Goals:** Improve existing features, add more value, and increase robustness based on V2 TDDs.

*   **Admin Dashboard:**
    *   [ ] **API Usage Tracking Display:** UI to show API usage per tenant (based on `api_usage_logs`). (e.g., 2-3d)
    *   [ ] **Advanced Subscription Management:** UI for admins to cancel/modify user subscriptions (with care, logging, and Stripe interaction). (e.g., 2-3d)
    *   [ ] Discount/Coupon management interface (integrating with Stripe Coupons). (e.g., 2-3d)
*   **Shopify App & Backend:**
    *   [ ] **Python PAA Service V2 Enhancements:**
        *   Implement selected features from `tech-design.python-paa-service-enhancements.md` (e.g., advanced error handling, SERP API fallback, enhanced data). (e.g., 3-5d)
    *   [ ] **External Publishing to WordPress (API-based):**
        *   Secure credential storage for WordPress. (e.g., 4-8h)
        *   Backend logic and UI for publishing to WordPress. (e.g., 2-3d)
    *   [ ] **Feature Customization & Styling:**
        *   Implement as per `docs/jules/shopify-app/jules.feature-customization-and-styling.md`. (e.g., 2-3d)
        *   Theme App Extension for article structure and styling. (e.g., 2-3d)
        *   App settings for default colors, CTA text. (e.g., 1-2d)
    *   [ ] **Asynchronous Backend Operations (V2 from TDDs):**
        *   Refactor long-running AI operations (batch outline, text construction) to use Cloud Tasks for better resilience and UX, as mentioned in `tech-design.multi-article-flow.md`. (e.g., 3-5d)
    *   [ ] **Advanced Error Recovery:** Implement more sophisticated error recovery in multi-article flow (e.g., retry individual failed articles within a job from workspace). (e.g., 2-3d)
    *   [ ] **User Notifications:** For long-running jobs or important events. (e.g., 1-2d)
*   **General:**
    *   [ ] Performance optimization. (Ongoing) (e.g., per iteration/sprint)
    *   [ ] Enhanced security audit and hardening. (Ongoing) (e.g., per major release)
    *   [ ] Comprehensive analytics and monitoring dashboards. (e.g., 3-5d initially, then ongoing)
    *   [ ] User documentation and support resources. (Ongoing) (e.g., per feature/release)

## 6. Post-Launch & Ongoing Maintenance
*   Monitoring of all services (Firebase, Stripe, Python PAA, third-party APIs). (Ongoing)
*   Regular dependency updates. (Ongoing)
*   User support and feedback collection. (Ongoing)
*   Iterative improvements based on usage data and feedback. (Ongoing)
*   Cost management and optimization. (Ongoing)

This roadmap provides a structured approach to developing the Content Agent project. Timelines are estimates and can be adjusted based on team velocity and priorities. Each phase should include dedicated time for testing and quality assurance.

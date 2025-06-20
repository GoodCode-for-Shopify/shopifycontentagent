# Instructional Outline: Building a Next.js Administrative Dashboard for a Freemium Shopify App

## 1. Project Setup
### 1.1 Define Requirements
- Confirm freemium model: Free, Basic ($29/month), Pro ($99/month), Enterprise (custom).
- Define pricing tiers for developer-provided credentials (e.g., +$20/month for Basic, +$50/month for Pro).
- Specify API integrations: Google Ads API, Gemini AI API, Stripe.
- Outline security requirements: encryption, authentication, GDPR/CCPA compliance.

### 1.2 Set Up Development Environment
- Install Node.js, npm, and Next.js CLI.
- Set up version control (Git, GitHub).
- Configure IDE (e.g., VS Code) with ESLint, Prettier, and Prisma extensions.
- Create project directory: `admin-dashboard`.

## 2. Infrastructure Setup
### 2.1 Database Configuration
- Provision PostgreSQL on AWS RDS.
- Define database schema (shops, plans, subscriptions, store_credentials, api_usage).
- Initialize Prisma ORM and run migrations.

### 2.2 Cloud Services
- Set up AWS KMS for credential encryption.
- Configure Redis (via Upstash or AWS ElastiCache) for rate limiting and caching.
- Create Vercel account for Next.js deployment.
- Set up environment variables for AWS, Redis, and database.

### 2.3 Stripe Configuration
- Create Stripe account and obtain API keys (secret, webhook).
- Define price tiers in Stripe Dashboard (Free, Basic, Pro, with/without developer credentials).
- Create coupons for discounts (e.g., 10% off, $50 off).
- Configure Stripe webhooks for subscription events.

## 3. Backend Development
### 3.1 Initialize Next.js Project
- Run `npx create-next-app@latest admin-dashboard --js --tailwind --app`.
- Install dependencies: `@shopify/polaris`, `stripe`, `@prisma/client`, `next-auth`, `redis`, `bullmq`, `@aws-sdk/client-kms`, `winston`.

### 3.2 Set Up Authentication
- Configure NextAuth.js for admin login (email/password, MFA).
- Implement role-based access control (admin vs. tenant).
- Create API route for admin authentication (`/api/auth/[...nextauth]`).

### 3.3 Develop API Routes
- Create endpoints for:
  - Tenant management (`/api/shops`): CRUD operations for shops.
  - Subscription management (`/api/subscriptions`): Create, update, cancel subscriptions.
  - Credential management (`/api/credentials`): Store and retrieve encrypted credentials.
  - Usage tracking (`/api/usage`): Log and monitor API calls.
- Implement middleware for authentication and input validation.

### 3.4 Implement Credential Encryption
- Integrate AWS KMS for encrypting/decrypting credentials.
- Create utility functions (`encrypt`, `decrypt`) in `lib/crypto.js`.
- Store encrypted credentials in `store_credentials` table with initialization vector (if applicable).

### 3.5 Set Up API Usage Tracking
- Implement Redis-based counter for API usage (`incrementApiUsage`).
- Store usage data in `api_usage` table via Prisma.
- Enforce per-tenant API quotas based on plan.

### 3.6 Configure Logging
- Set up Winston for audit logging (credential access, subscription changes).
- Create log files (`error.log`, `combined.log`) for debugging and compliance.

## 4. Admin Dashboard Development
### 4.1 Build Core UI Components
- Create dashboard layout using Shopify Polaris (`app/layout.jsx`).
- Implement main dashboard page (`app/dashboard/page.jsx`) with tenant overview (DataTable).
- Design shop details page (`app/shops/[id]/page.jsx`) for managing credentials and subscriptions.

### 4.2 Implement Subscription Management
- Create UI for viewing/editing subscriptions (plan selection, discount code input).
- Integrate Stripe Checkout for upgrading plans.
- Display subscription status and billing history.

### 4.3 Develop Credential Management UI
- Build form for tenants to input API credentials (Google Ads Customer ID, Google API Key, Gemini AI API Key).
- Implement OAuth flow for Google Ads Refresh Token (redirect to Google’s consent screen).
- Add toggle for using developer-provided credentials (updates subscription price).

### 4.4 Add Usage Monitoring
- Display per-tenant API usage (Google Ads, Gemini AI) in dashboard.
- Show warnings for approaching quota limits.
- Allow admins to reset or adjust quotas for Enterprise plans.

## 5. API Integration
### 5.1 Google Ads API
- Set up Google Ads API client (e.g., `google-ads-api` for Node.js).
- Implement OAuth flow for tenant-specific refresh tokens.
- Create utility functions for common API calls (e.g., fetch campaigns, reports).
- Store developer credentials (Client ID, Client Secret, Developer Token) in environment variables.

### 5.2 Gemini AI API
- Configure Gemini AI API client (or equivalent, e.g., Vertex AI).
- Allow tenants to input their own API key or use developer’s key.
- Track usage against developer’s key for billing purposes.

### 5.3 Rate Limiting and Queuing
- Use BullMQ to queue API calls and avoid rate limit violations.
- Monitor usage in Google Cloud Console and adjust quotas as needed.

## 6. Security Implementation
### 6.1 Secure Credential Storage
- Ensure all tenant credentials are encrypted before storage.
- Restrict access to AWS KMS key via IAM policies.
- Validate all user inputs to prevent injection attacks.

### 6.2 Protect API Routes
- Apply authentication middleware to all admin routes.
- Implement CORS and rate limiting (Redis).
- Use HTTPS for all communications.

### 6.3 Compliance
- Document data handling practices for GDPR/CCPA compliance.
- Implement data minimization (store only necessary credentials).
- Set up secure database backups on AWS RDS.

## 7. Testing
### 7.1 Unit and Integration Tests
- Write tests for API routes (using Jest).
- Test OAuth flows and credential encryption.
- Verify Stripe subscription and webhook handling.

### 7.2 Security Testing
- Conduct penetration testing for XSS, SQL injection, and API vulnerabilities.
- Audit encryption and access controls.
- Test MFA and role-based access.

### 7.3 Performance Testing
- Simulate multiple tenants to test API quotas and rate limiting.
- Verify scalability of Redis and BullMQ under load.
- Test database performance with Prisma.

## 8. Deployment
### 8.1 Deploy Backend
- Deploy Next.js app to Vercel.
- Configure environment variables in Vercel Dashboard.
- Set up AWS RDS, KMS, and Redis connections.

### 8.2 Test Webhooks
- Use Stripe CLI to test webhooks locally.
- Verify webhook handling in production (e.g., subscription updates).

### 8.3 Monitor and Optimize
- Set up monitoring (Vercel Analytics, AWS CloudWatch).
- Optimize API usage based on real-world data.
- Adjust Redis and BullMQ configurations for performance.

## 9. Preparation for Shopify App
### 9.1 Plan End-User App
- Outline requirements for Shopify app (embedded in Shopify admin).
- Reuse backend API routes for tenant-facing functionality.
- Design UI with Polaris for consistency.

### 9.2 Shopify OAuth
- Implement Shopify OAuth for tenant authentication.
- Store Shopify access tokens in database (encrypted).

### 9.3 Freemium Features
- Restrict features based on plan (e.g., hide advanced reports for Free Plan).
- Integrate Stripe Checkout for in-app upgrades.
- Allow credential input or developer-provided credential toggle.

## 10. Shopify App Store Submission
### 10.1 Documentation
- Write user guides for tenant credential setup (Google Ads, Gemini AI).
- Document freemium features and pricing.

### 10.2 Compliance
- Ensure secure credential handling for Shopify App Store review.
- Verify GDPR/CCPA compliance.
- Submit app for review (after end-user app is built).

## 11. Post-Launch
### 11.1 Monitor Usage
- Track API usage (Google Ads, Gemini AI) to manage costs.
- Adjust pricing tiers based on real-world data.
- Monitor Stripe billing for payment issues.

### 11.2 Support and Maintenance
- Set up support channels for tenants (e.g., email, in-app chat).
- Regularly update dependencies and security patches.
- Plan for scalability (e.g., sharding database for high tenant volumes).
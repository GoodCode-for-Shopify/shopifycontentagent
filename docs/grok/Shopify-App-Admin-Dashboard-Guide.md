# Implementing a Next.js Administrative Dashboard for a Freemium Shopify App

## 1. Overview
This document provides a detailed guide for building a Next.js administrative dashboard to manage a multi-tenant Shopify app offered as a freemium product. The app integrates with AI and third-party APIs (e.g., Google Ads, Gemini AI) and uses Stripe for subscription billing, including discount code support. It addresses the option for customers to use the app developer’s API credentials (with higher fees) or provide their own, considering API usage limits. The dashboard and server-side functionality will be built first to ensure robust management capabilities before developing the end-user Shopify app. Comprehensive security is prioritized throughout.

### 1.1 Objectives
- Create a secure Next.js administrative dashboard to manage tenants, subscriptions, and API credentials.
- Implement a freemium model with Stripe for billing, including free and paid plans (Basic, Pro, Enterprise).
- Support discount codes via Stripe.
- Allow customers to either provide their own API credentials or use the developer’s credentials for a higher fee.
- Address API usage limits and billing implications.
- Build server-side functionality first, ensuring scalability and security.
- Prepare for Shopify App Store compliance and end-user app development.

### 1.2 Technology Stack
- **Frontend**: Next.js (React-based, with App Router), Tailwind CSS, Shopify Polaris (for UI consistency).
- **Backend**: Next.js API Routes, PostgreSQL (database), Prisma (ORM).
- **Payment Processing**: Stripe (subscriptions, discount codes).
- **Authentication**: NextAuth.js (for admin and Shopify OAuth).
- **API Integrations**: Google Ads API, Gemini AI API (or equivalent).
- **Security**: Encryption (AES-256), Key Management (AWS KMS), HTTPS, JWT.
- **Hosting**: Vercel (Next.js), AWS RDS (PostgreSQL), AWS KMS (key management).
- **Other**: Redis (rate limiting, caching), BullMQ (job queue), Winston (logging).

## 2. Freemium Model and Pricing Strategy

### 2.1 Freemium Model
- **Free Plan**:
  - Limited features (e.g., basic reporting, 100 API calls/month).
  - Requires tenants to provide their own API credentials (e.g., Google Ads Refresh Token, Customer ID, Google API Key, Gemini AI API Key).
  - No access to developer’s API credentials.
- **Basic Plan** ($29/month):
  - More features (e.g., advanced reporting, 1,000 API calls/month).
  - Option to provide own credentials or use developer’s credentials for Google API Key and Gemini AI API Key (Google Ads still requires tenant’s Refresh Token and Customer ID).
  - 10% discount code support.
- **Pro Plan** ($99/month):
  - Full features (e.g., automation, 10,000 API calls/month).
  - Same credential options as Basic Plan.
  - 20% discount code support.
- **Enterprise Plan** (Custom pricing):
  - Unlimited features, dedicated support, custom API quotas.
  - Same credential options, with potential for fully managed credentials (if feasible).

### 2.2 Credential Options
- **Tenant-Provided Credentials**:
  - Tenants provide their own Google Ads Refresh Token (via OAuth), Customer ID, Login Customer ID (if applicable), Google API Key, and Gemini AI API Key.
  - Reduces your API costs and usage against your Developer Token or API Keys.
  - Suitable for Free and paid plans.
- **Developer-Provided Credentials**:
  - Tenants use your Google API Key and Gemini AI API Key (hardcoded in environment variables) for a higher fee (e.g., +$20/month for Basic, +$50/month for Pro).
  - Google Ads still requires tenant-specific Refresh Token and Customer ID due to OAuth and account-specific access.
  - Increases your API costs and usage, requiring careful quota management.
  - Feasible for paid plans only, subject to API rate limits.

### 2.3 API Usage Limits
- **Google Ads API**:
  - Billable and rate-limited (via Developer Token).
  - Usage tracked against your Developer Token, regardless of tenant credentials.
  - Mitigate with tenant-provided Refresh Tokens and Customer IDs to isolate account-specific calls.
  - Use Redis and BullMQ to queue API calls and avoid hitting quotas.
- **Google API Key** (e.g., for non-Ads APIs):
  - Billable and rate-limited.
  - If tenants provide their own, their usage is isolated.
  - If using your key, monitor usage in Google Cloud Console and set per-tenant quotas (e.g., 500 calls/month for Basic Plan).
- **Gemini AI API**:
  - Billable and rate-limited.
  - Same considerations as Google API Key.
  - If centralized, use usage tracking to prevent overages.

### 2.4 Stripe Integration
- **Subscriptions**: Use Stripe Subscriptions for recurring billing (monthly/yearly).
- **Discount Codes**: Stripe supports coupons (e.g., 10% off for 3 months, one-time $50 off).
- **Higher Fees for Developer Credentials**: Create separate price tiers in Stripe for plans with developer-provided credentials (e.g., “Pro with Developer Credentials” at $149/month vs. $99/month).
- **Webhooks**: Handle events like subscription updates, payment failures, and cancellations.

## 3. Architecture

### 3.1 System Overview
- **Admin Dashboard**: Next.js app for managing tenants, subscriptions, credentials, and usage.
- **Shopify App**: Future end-user app embedded in Shopify admin (to be built later).
- **Backend**: Next.js API Routes for server-side logic, connected to PostgreSQL via Prisma.
- **External Services**: Stripe, Google Ads API, Gemini AI API, AWS KMS, Redis.
- **Security**: Encrypted credentials, JWT-based auth, rate limiting, logging.

### 3.2 Database Schema
Using PostgreSQL with Prisma, key tables include:
- **shops**:
  - `id` (PK), `shop_id` (Shopify domain, e.g., `store.myshopify.com`), `access_token` (Shopify OAuth), `plan_id` (FK), `created_at`, `updated_at`.
- **plans**:
  - `id` (PK), `name` (e.g., Free, Basic), `price` (monthly fee), `api_quota` (e.g., 1000 calls), `uses_developer_credentials` (boolean), `stripe_price_id`.
- **subscriptions**:
  - `id` (PK), `shop_id` (FK), `stripe_subscription_id`, `status` (active, canceled), `discount_code` (Stripe coupon ID), `created_at`, `updated_at`.
- **store_credentials**:
  - `id` (PK), `shop_id` (FK), `credential_type` (e.g., `google_ads_refresh_token`), `credential_value` (encrypted), `iv` (encryption IV), `created_at`.
- **api_usage**:
  - `id` (PK), `shop_id` (FK), `api_type` (e.g., `google_ads`), `call_count`, `date`, `created_at`.

### 3.3 Security Architecture
- **Authentication**: NextAuth.js for admin login (email/password, MFA), Shopify OAuth for tenant auth.
- **Authorization**: Role-based access (admin vs. tenant), JWT tokens.
- **Credential Storage**: Encrypt tenant credentials (AES-256) with key in AWS KMS.
- **API Security**: Rate limiting (Redis), input validation, CORS, HTTPS.
- **Logging**: Winston for audit logs (e.g., credential access, subscription changes).
- **Data Protection**: GDPR/CCPA compliance, data minimization, secure backups.

## 4. Implementation Steps

### 4.1 Server-Side Setup
Build the backend first to support the admin dashboard and future Shopify app.

#### 4.1.1 Database Setup
- Initialize PostgreSQL on AWS RDS.
- Define schema with Prisma:
```prisma
model Shop {
  id                Int            @id @default(autoincrement())
  shop_id           String         @unique
  access_token      String
  plan              Plan?          @relation(fields: [planId], references: [id])
  planId            Int?
  subscriptions     Subscription[]
  credentials       StoreCredential[]
  apiUsage          ApiUsage[]
  created_at        DateTime       @default(now())
  updated_at        DateTime       @updatedAt
}

model Plan {
  id                    Int            @id @default(autoincrement())
  name                  String
  price                 Float
  api_quota             Int
  uses_developer_credentials Boolean
  stripe_price_id       String
  shops                 Shop[]
  created_at            DateTime       @default(now())
}

model Subscription {
  id                    Int            @id @default(autoincrement())
  shop_id               String
  shop                  Shop           @relation(fields: [shop_id], references: [shop_id])
  stripe_subscription_id String
  status                String
  discount_code         String?
  created_at            DateTime       @default(now())
  updated_at            DateTime       @updatedAt
}

model StoreCredential {
  id                Int            @id @default(autoincrement())
  shop_id           String
  shop              Shop           @relation(fields: [shop_id], references: [shop_id])
  credential_type   String
  credential_value  String
  iv                String
  created_at        DateTime       @default(now())
}

model ApiUsage {
  id                Int            @id @default(autoincrement())
  shop_id           String
  shop              Shop           @relation(fields: [shop_id], references: [shop_id])
  api_type          String
  call_count        Int
  date              DateTime
  created_at        DateTime       @default(now())
}
```
- Run migrations: `npx prisma migrate dev`.

#### 4.1.2 Next.js API Routes
- Create API endpoints for:
  - Tenant management (`/api/shops`).
  - Subscription management (`/api/subscriptions`).
  - Credential management (`/api/credentials`).
  - Usage tracking (`/api/usage`).
- Example: Create a shop:
```javascript
// pages/api/shops/create.js
import { withAuth } from '@/middleware/auth';
import prisma from '@/lib/prisma';
import { encrypt } from '@/lib/crypto';

export default withAuth(async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).json({ error: 'Method not allowed' });
  
  const { shop_id, access_token } = req.body;
  try {
    const shop = await prisma.shop.create({
      data: {
        shop_id,
        access_token: encrypt(access_token).encryptedData,
        planId: 1, // Default to Free Plan
      },
    });
    res.status(201).json(shop);
  } catch (error) {
    res.status(500).json({ error: 'Failed to create shop' });
  }
});
```

#### 4.1.3 Stripe Integration
- Install Stripe: `npm install stripe`.
- Create price tiers in Stripe Dashboard:
  - Free Plan: $0/month.
  - Basic Plan: $29/month, $49/month (with developer credentials).
  - Pro Plan: $99/month, $149/month (with developer credentials).
- Create coupons (e.g., `10OFF` for 10% off).
- Handle subscriptions:
```javascript
// lib/stripe.js
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);

export async function createSubscription(shopId, planId, discountCode) {
  const plan = await prisma.plan.findUnique({ where: { id: planId } });
  const customer = await stripe.customers.create({ metadata: { shopId } });
  
  const subscription = await stripe.subscriptions.create({
    customer: customer.id,
    items: [{ price: plan.stripe_price_id }],
    coupon: discountCode || null,
  });
  
  await prisma.subscription.create({
    data: {
      shop_id: shopId,
      stripe_subscription_id: subscription.id,
      status: subscription.status,
      discount_code: discountCode,
    },
  });
  
  return subscription;
}
```
- Webhook for subscription events:
```javascript
// pages/api/stripe/webhook.js
import Stripe from 'stripe';
import { buffer } from 'micro';
import prisma from '@/lib/prisma';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

export const config = { api: { bodyParser: false } };

export default async function handler(req, res) {
  const buf = await buffer(req);
  const sig = req.headers['stripe-signature'];
  
  try {
    const event = stripe.webhooks.constructEvent(buf, sig, process.env.STRIPE_WEBHOOK_SECRET);
    
    if (event.type === 'customer.subscription.updated') {
      const subscription = event.data.object;
      await prisma.subscription.update({
        where: { stripe_subscription_id: subscription.id },
        data: { status: subscription.status },
      });
    }
    
    res.json({ received: true });
  } catch (error) {
    res.status(400).json({ error: 'Webhook error' });
  }
};
```

#### 4.1.4 Credential Management
- Encrypt credentials with AWS KMS:
```javascript
// lib/crypto.js
const AWS = require('aws-sdk');
const kms = new AWS.KMS({ region: process.env.AWS_REGION });

export async function encrypt(text) {
  const { CiphertextBlob } = await kms.encrypt({
    KeyId: process.env.KMS_KEY_ID,
    Plaintext: text,
  }).promise();
  return { encryptedData: CiphertextBlob.toString('base64'), iv: null };
}

export async function decrypt(encryptedData) {
  const { Plaintext } = await kms.decrypt({
    CiphertextBlob: Buffer.from(encryptedData, 'base64'),
  }).promise();
  return Plaintext.toString('utf-8');
}
```
- Store credentials:
```javascript
// pages/api/credentials/store.js
import { withAuth } from '@/middleware/auth';
import prisma from '@/lib/prisma';
import { encrypt } from '@/lib/crypto';

export default withAuth(async function handler(req, res) {
  const { shop_id, credential_type, value } = req.body;
  const { encryptedData } = await encrypt(value);
  
  await prisma.storeCredential.create({
    data: {
      shop_id,
      credential_type,
      credential_value: encryptedData,
    },
  });
  
  res.status(201).json({ success: true });
});
```

#### 4.1.5 API Usage Tracking
- Track usage with Redis:
```javascript
// lib/usage.js
import Redis from 'ioredis';
import prisma from './prisma';

const redis = new Redis(process.env.REDIS_URL);

export async function incrementApiUsage(shopId, apiType) {
  const key = `usage:${shopId}:${apiType}:${new Date().toISOString().split('T')[0]}`;
  const count = await redis.incr(key);
  
  await prisma.apiUsage.upsert({
    where: { shop_id_api_type_date: { shop_id: shopId, api_type: apiType, date: new Date() } },
    update: { call_count: count },
    create: { shop_id: shopId, api_type: apiType, call_count: count, date: new Date() },
  });
  
  return count;
}
```

### 4.2 Admin Dashboard Implementation
#### 4.2.1 Setup Next.js
- Initialize: `npx create-next-app@latest admin-dashboard --js --tailwind --app`.
- Install dependencies:
```bash
npm install @shopify/polaris @stripe/stripe-js stripe @prisma/client next-auth redis bullmq aws-sdk-js-v3 winston
npx prisma init
```

#### 4.2.2 Authentication
- Configure NextAuth.js:
```javascript
// pages/api/auth/[...nextauth].js
import NextAuth from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import prisma from '@/lib/prisma';
import bcrypt from 'bcryptjs';

export default NextAuth({
  providers: [
    CredentialsProvider({
      name: 'Credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        const admin = await prisma.admin.findUnique({ where: { email: credentials.email } });
        if (admin && bcrypt.compareSync(credentials.password, admin.password)) {
          return { id: admin.id, email: admin.email, role: admin.role };
        }
        return null;
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) token.role = user.role;
      return token;
    },
    async session({ session, token }) {
      session.user.role = token.role;
      return session;
    },
  },
});
```

#### 4.2.3 Dashboard UI
- Use Shopify Polaris for consistent UI:
```jsx
// app/dashboard/page.jsx
import { AppProvider, Page, Card, DataTable, Button } from '@shopify/polaris';
import { useSession } from 'next-auth/react';
import { useRouter } from 'next/navigation';
import { useEffect, useState } from 'react';

export default function Dashboard() {
  const { data: session, status } = useSession();
  const router = useRouter();
  const [shops, setShops] = useState([]);

  useEffect(() => {
    if (status === 'unauthenticated') router.push('/login');
    if (status === 'authenticated') {
      fetch('/api/shops').then(res => res.json()).then(setShops);
    }
  }, [status, router]);

  const rows = shops.map(shop => [
    shop.shop_id,
    shop.plan?.name || 'Free',
    shop.subscriptions[0]?.status || 'None',
    <Button onClick={() => router.push(`/shops/${shop.id}`)}>View</Button>,
  ]);

  return (
    <AppProvider>
      <Page title="Admin Dashboard">
        <Card>
          <DataTable
            columnContentTypes={['text', 'text', 'text', 'text']}
            headings={['Shop ID', 'Plan', 'Subscription Status', 'Actions']}
            rows={rows}
          />
        </Card>
      </Page>
    </AppProvider>
  );
}
```

#### 4.2.4 Tenant Management
- Create pages for viewing/editing shops, subscriptions, credentials, and usage.
- Example: Shop details page:
```jsx
// app/shops/[id]/page.jsx
import { AppProvider, Page, Card, Form, FormLayout, TextField, Button } from '@shopify/polaris';
import { useRouter } from 'next/navigation';
import { useState, useEffect } from 'react';

export default function ShopDetails({ params }) {
  const router = useRouter();
  const [shop, setShop] = useState(null);
  const [credential, setCredential] = useState('');

  useEffect(() => {
    fetch(`/api/shops/${params.id}`).then(res => res.json()).then(setShop);
  }, [params.id]);

  const handleAddCredential = async () => {
    await fetch('/api/credentials/store', {
      method: 'POST',
      body: JSON.stringify({
        shop_id: shop.shop_id,
        credential_type: 'google_ads_refresh_token',
        value: credential,
      }),
    });
    setCredential('');
  };

  if (!shop) return <div>Loading...</div>;

  return (
    <AppProvider>
      <Page title={`Shop: ${shop.shop_id}`}>
        <Card>
          <Form onSubmit={handleAddCredential}>
            <FormLayout>
              <TextField
                label="Add Google Ads Refresh Token"
                value={credential}
                onChange={setCredential}
                autoComplete="off"
              />
              <Button submit>Add Credential</Button>
            </FormLayout>
          </Form>
        </Card>
      </Page>
    </AppProvider>
  );
}
```

### 4.3 Security Implementation
- **Environment Variables**:
```env
NEXTAUTH_SECRET=your_nextauth_secret
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
AWS_ACCESS_KEY_ID=your_aws_key
AWS_SECRET_ACCESS_KEY=your_aws_secret
AWS_REGION=us-east-1
KMS_KEY_ID=your_kms_key
REDIS_URL=redis://...
DATABASE_URL=postgresql://...
GOOGLE_ADS_CLIENT_ID=your_client_id
GOOGLE_ADS_CLIENT_SECRET=your_client_secret
GOOGLE_ADS_DEVELOPER_TOKEN=your_developer_token
GOOGLE_API_KEY=your_api_key
GEMINI_AI_API_KEY=your_gemini_key
```
- **Middleware**: Protect routes with auth and role checks:
```javascript
// middleware/auth.js
import { getSession } from 'next-auth/react';

export function withAuth(handler) {
  return async (req, res) => {
    const session = await getSession({ req });
    if (!session || session.user.role !== 'admin') {
      return res.status(403).json({ error: 'Forbidden' });
    }
    return handler(req, res);
  };
}
```
- **Rate Limiting**: Use Redis to limit API requests.
- **Logging**: Use Winston for audit logs:
```javascript
// lib/logger.js
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

export default logger;
```

### 4.4 Deployment
- Deploy Next.js app to Vercel.
- Configure environment variables in Vercel Dashboard.
- Set up AWS RDS, KMS, and Redis (via Upstash or AWS ElastiCache).
- Test Stripe webhooks locally with Stripe CLI.

## 5. Building the Shopify App
- **Postpone End-User App**: Focus on admin dashboard and backend first.
- **Shopify App Setup**:
  - Use Shopify CLI to scaffold a Node.js app: `npm init @shopify/app@latest`.
  - Integrate with admin dashboard’s backend (same API Routes).
  - Embed settings page in Shopify admin using Polaris.
  - Implement OAuth for Google Ads (same flow as admin dashboard).
- **Freemium Features**:
  - Restrict features based on plan (e.g., hide advanced reports for Free Plan).
  - Prompt users to upgrade via Stripe Checkout.
- **Credential Input**:
  - Allow tenants to input Customer ID, Google API Key, Gemini AI API Key.
  - Offer “Use Developer Credentials” option for paid plans, updating subscription in Stripe.

## 6. Compliance and Testing
- **Shopify App Store**:
  - Ensure secure credential handling (encrypted storage, HTTPS).
  - Provide clear user documentation.
  - Submit for review after end-user app is built.
- **Security Testing**:
  - Penetration testing for XSS, SQL injection, etc.
  - Audit credential encryption and access controls.
- **Performance Testing**:
  - Simulate multiple tenants to test API quotas and rate limiting.
  - Monitor Stripe webhook reliability.

## 7. Conclusion
This guide outlines a secure, scalable approach to building a Next.js administrative dashboard for a freemium Shopify app. By prioritizing the server-side functionality and admin dashboard, you establish a robust foundation for managing tenants, subscriptions, and credentials before developing the end-user app. Stripe enables flexible billing, including discount codes and higher fees for developer-provided credentials. Careful management of API usage limits ensures feasibility, while comprehensive security measures protect sensitive data.

For further assistance, consider:
- Detailed code for specific features (e.g., OAuth flow, Stripe Checkout).
- Guidance on Shopify App Store submission.
- Scaling strategies for high tenant volumes.

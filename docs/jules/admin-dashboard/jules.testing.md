# Admin Dashboard: Testing Strategies

Thorough testing is crucial to ensure the reliability, security, and performance of the Admin Dashboard and its associated backend services. This document outlines testing strategies tailored to our Firebase, Cloud Functions, and React/Polaris stack.

## 1. Unit and Integration Testing (Backend - Cloud Functions)

(Derived from Grok Outline Section 7.1, adapted for Firebase)

### 1.1. Testing Cloud Functions
*   **Frameworks:** Use JavaScript testing frameworks like **Jest** or **Mocha** with **Chai** for assertions.
*   **Firebase Local Emulator Suite:** This is an indispensable tool. It allows you to run emulators for Cloud Functions, Firestore, Firebase Authentication, Pub/Sub, and Firebase Hosting locally.
    *   Write tests that interact with the emulated services.
    *   Initialize Firebase Admin SDK in your tests to point to the emulators.
    *   Clear emulator data (e.g., Firestore data) before each test run or suite for consistent results.
*   **Unit Tests:**
    *   Focus on individual functions or modules within your Cloud Functions backend (e.g., testing the logic of an encryption utility, a specific business logic calculation).
    *   Mock external dependencies that are not part of the unit being tested.
*   **Integration Tests:**
    *   Test the interaction between different parts of your backend, or between your functions and emulated Firebase services.
    *   Example: Test an API endpoint by sending an HTTP request to the emulated function URL and verifying the response and any Firestore data changes.
    *   Test authentication middleware by sending requests with and without valid Firebase Auth ID tokens.

    ```javascript
    // Example (Conceptual Jest test for an Express endpoint in Cloud Functions)
    // const admin = require('firebase-admin');
    // const supertest = require('supertest'); // For HTTP testing
    // const { api } = require('../src/index'); // Your main Express app export
    // const { setupEmulators, clearFirestoreData } = require('./testSetup'); // Your test helpers

    // beforeAll(async () => await setupEmulators());
    // beforeEach(async () => await clearFirestoreData());

    // describe('GET /admin/shops', () => {
    //   it('should return 403 for unauthenticated requests', async () => {
    //     await supertest(api).get('/admin/shops').expect(403);
    //   });

    //   it('should return 200 and shop list for authenticated admin', async () => {
    //     const adminIdToken = await getAdminIdToken(); // Helper to get a valid admin token from Auth emulator
    //     // Seed Firestore with some shop data via Admin SDK
    //     await admin.firestore().collection('shops').doc('shop1').set({ name: 'Test Shop 1'});

    //     const response = await supertest(api)
    //       .get('/admin/shops')
    //       .set('Authorization', `Bearer ${adminIdToken}`)
    //       .expect(200);
    //     expect(response.body).toBeInstanceOf(Array);
    //     expect(response.body.length).toBeGreaterThan(0);
    //     expect(response.body[0].name).toEqual('Test Shop 1');
    //   });
    // });
    ```

### 1.2. Testing Specific Backend Logic
*   **OAuth Flows:** Mocking the external Google OAuth provider is complex. Focus on testing your internal logic:
    *   Does your `/api/oauth/google/initiate` endpoint correctly generate the Google consent URL?
    *   Does your `/api/oauth/google/callback` endpoint correctly handle a (mocked) successful code exchange, encrypt the token, and store it in Firestore (emulator)? Test error handling for invalid codes or states.
*   **Credential Encryption:** Unit test your `encrypt` and `decrypt` functions with known inputs and outputs.
*   **Stripe Integration:**
    *   Use Stripe's mock event data and test webhook signatures locally.
    *   Test your webhook handler logic by sending mock Stripe events to your emulated webhook function. Verify that it correctly updates Firestore.
    *   Stripe CLI can be used to forward webhook events to your local emulator.

## 2. Frontend Testing (React/Polaris Admin Dashboard)

### 2.1. Unit and Component Tests
*   **Frameworks:** Use **Jest** with **React Testing Library** for testing React components.
*   **Unit Tests:** Test individual React components in isolation, mocking props and verifying rendered output or callbacks.
*   **Component Tests:** Test interactions within components or simple compositions of components.
    ```jsx
    // Example (Conceptual React Testing Library test)
    // import { render, screen, fireEvent } from '@testing-library/react';
    // import MyShopifyPolarisComponent from '../MyComponent';

    // test('renders component and handles click', () => {
    //   const handleClickMock = jest.fn();
    //   render(<MyShopifyPolarisComponent onClick={handleClickMock} />);
    //   fireEvent.click(screen.getByText('Click Me'));
    //   expect(handleClickMock).toHaveBeenCalledTimes(1);
    // });
    ```

### 2.2. Integration Tests (Frontend)
*   Test interactions between multiple frontend components, routing, and state management.
*   Mock API calls to your backend (Cloud Functions) to isolate frontend logic. Use libraries like `msw` (Mock Service Worker) or `jest.mock`.

### 2.3. End-to-End (E2E) Testing (Optional but Recommended)
*   **Frameworks:** Consider tools like **Cypress** or **Playwright**.
*   These tests interact with your application like a real user, running in a browser.
*   They can test entire user flows: admin login -> navigate to shops list -> view shop details -> update a setting.
*   E2E tests can be run against your local Firebase Emulators (frontend served by hosting emulator, backend by functions emulator) or against a staging environment.
*   **Challenges:** Can be slower and more brittle than unit/integration tests, but provide high confidence.

## 3. Security Testing

(Derived from Grok Outline Section 7.2)

Security testing is critical for an admin dashboard handling sensitive data.

*   **Authentication & Authorization:**
    *   Manually test access controls: try to access admin routes without logging in, or with non-admin user tokens.
    *   Verify that one admin cannot access data or functions they are not permissioned for (if granular admin roles are implemented).
*   **Input Validation:**
    *   Attempt to submit invalid or malicious data to API endpoints (e.g., excessively long strings, script tags, unexpected data types) and verify that the backend handles it gracefully (e.g., returns appropriate error codes, doesn't crash).
*   **Credential Handling:**
    *   Verify that no sensitive credentials are ever exposed in API responses or frontend code.
    *   Confirm that encryption is correctly applied and keys are managed securely.
*   **Dependency Vulnerabilities:** Regularly run `npm audit` (for both frontend and backend) to check for known vulnerabilities in dependencies.
*   **Penetration Testing (Consider for Mature Apps):** For applications handling highly sensitive data or facing significant threats, consider engaging third-party security professionals for penetration testing.

## 4. Performance Testing

(Derived from Grok Outline Section 7.3, adapted for Firebase)

*   **Firestore Performance:**
    *   Analyze query performance using Firestore's query explain/analysis tools (if available through GCP console).
    *   Ensure efficient indexing for common queries made by the admin dashboard.
    *   Test how the application performs with a larger number of shops, subscriptions, or credentials in Firestore (use test data generation scripts).
*   **Cloud Function Performance:**
    *   Monitor function execution times, memory usage, and invocation counts in the Firebase/Google Cloud Console.
    *   Identify and optimize slow-running functions.
    *   Test cold start times and their impact on user experience for critical functions.
*   **Frontend Performance:**
    *   Use browser developer tools (Lighthouse, Performance tab) to analyze frontend load times, rendering performance, and identify bottlenecks in React components.
    *   Optimize bundle sizes and ensure efficient data fetching.

## 5. User Acceptance Testing (UAT)
*   Before full rollout, have intended admin users (e.g., your internal team) test the dashboard to ensure it meets their needs, is intuitive, and functions correctly for common administrative tasks.

A comprehensive testing strategy, combining local emulation with various types of automated and manual tests, will significantly improve the quality and reliability of your admin dashboard.

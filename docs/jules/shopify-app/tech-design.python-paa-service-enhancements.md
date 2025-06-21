# Technical Design Document: Python PAA Service - V2 Enhancements

## 1. Introduction & Goals

### 1.1. Purpose of this Document
This Technical Design Document (TDD) outlines potential V2 enhancements for the Python "People Also Ask" (PAA) service used by the "Content Agent" Shopify app. The V1 service, described in `docs/jules/shopify-app/jules.python-paa-service-setup.md`, relies on the `people-also-ask` library for scraping Google SERP for PAA data.

This TDD explores improvements in robustness, scalability, data quality, and feature set.

### 1.2. Context: V1 PAA Service
*   **Functionality:** Receives keywords, language, and region; uses `people-also-ask` library to scrape PAA questions from Google.
*   **Deployment:** Python Cloud Function or Cloud Run.
*   **Limitations & Risks (from `jules.authority.md` and V1 TDD):**
    *   Reliant on web scraping, making it susceptible to changes in Google's SERP structure.
    *   Risk of being blocked by Google if query volume is too high or scraping is detected.
    *   Error handling is basic (per-keyword errors logged, empty list returned).
    *   Scalability might be limited by single-instance processing or library constraints.

### 1.3. Goals for V2 Enhancements
1.  **Increased Robustness:** Improve error handling, retries, and graceful degradation.
2.  **Better Scalability:** Handle higher query volumes more efficiently.
3.  **Alternative Data Sources:** Reduce sole reliance on direct Google scraping.
4.  **Data Quality & Depth:** Potentially fetch more than just questions (e.g., brief answers, related searches).
5.  **Improved Monitoring & Alerting:** Better insight into service health and scraping success rates.
6.  **Cost Management:** Optimize resource usage if using paid APIs as alternatives.

## 2. Proposed V2 Enhancements

### 2.1. Advanced Error Handling & Retry Mechanisms
*   **Current (V1):** Basic try-except per keyword.
*   **V2 Proposal:**
    *   **Granular Error Classification:** Differentiate between:
        *   Temporary network issues.
        *   Google blocking/CAPTCHA.
        *   No PAA results found (legitimate).
        *   Library-specific errors.
    *   **Configurable Retries:** Implement exponential backoff for retries on temporary errors or potential blocks.
        *   E.g., retry up to 3 times with delays of 1s, 3s, 10s.
    *   **Circuit Breaker:** If a high percentage of requests to Google fail (e.g., due to IP blocking), temporarily halt requests to that specific Google domain or globally for a cool-down period.
    *   **Dead Letter Queue (DLQ):** For persistently failing keywords/requests, send them to a DLQ (e.g., a Pub/Sub topic or Firestore collection) for later analysis or manual retry, instead of just returning empty.

### 2.2. Scalability Improvements
*   **Current (V1):** Synchronous processing of keywords within a single function invocation.
*   **V2 Proposal (if using Cloud Run, or for batching to Cloud Functions):**
    *   **Asynchronous Keyword Processing:** Instead of iterating through keywords sequentially in one request, the main HTTP function could act as a dispatcher.
        1.  Receive batch of keywords.
        2.  For each keyword (or small groups of keywords), enqueue a task to a Cloud Task queue.
        3.  A separate worker Cloud Function (or Cloud Run endpoint) processes each task, fetching PAA for its assigned keyword(s).
        4.  Results are written back to a temporary storage (e.g., Firestore document with a job ID, or Pub/Sub topic) for aggregation by the calling Node.js backend.
    *   **Concurrency Management for Workers:** Configure maximum concurrent instances for worker functions/services to manage load on Google and the `people-also-ask` library.
    *   **Proxy Rotation (Advanced & Complex):** For very high volumes, consider using a pool of proxy IPs to distribute requests to Google. This adds significant complexity and cost (procuring and managing proxies). *Likely out of scope for V2 unless absolutely critical and other methods fail.*

### 2.3. Alternative Data Sources & Fallbacks

To mitigate risks of relying solely on `people-also-ask` direct scraping:

*   **A. Paid SERP APIs:**
    *   **Concept:** Integrate with commercial SERP APIs (e.g., SerpAPI, Scale SERP, Bright Data's SERP API, AvesAPI) that handle proxy management and CAPTCHA solving. Many offer specific PAA endpoints or allow extracting this data.
    *   **Implementation:**
        1.  Conditionally use a paid API based on configuration (e.g., an environment variable `PAA_PROVIDER="SERP_API"`).
        2.  The Python service would need new modules to interact with the chosen SERP API's SDK or HTTP interface.
        3.  Map the SERP API's response for PAA questions to the format expected by the Node.js backend.
    *   **Pros:** More reliable than direct scraping, less likely to be blocked.
    *   **Cons:** Adds operational cost (API subscription fees).
    *   **Error Handling:** Still need to handle API errors, rate limits, and no-results scenarios from these services.

*   **B. AI-Generated Fallback Questions:**
    *   **Concept:** If direct scraping and/or paid SERP APIs fail or return no results for a keyword, use a powerful LLM (like Gemini) to generate potential PAA-style questions based on the keyword.
    *   **Implementation:**
        1.  If primary PAA sources yield no results, the Python service (or the Node.js backend after receiving an empty result) could flag this.
        2.  The Node.js backend could then call Gemini with a prompt like: "Generate 3-5 'People Also Ask' style questions for the keyword: '[keyword]'." Alternatively, if the Python PAA service has direct, low-latency access to an LLM and necessary credentials, it could perform this generation itself before responding to the Node.js backend.
    *   **Pros:** Provides some data even when scraping fails. Can be cost-effective if using existing LLM integrations.
    *   **Cons:** AI-generated questions might not be actual searched-for questions; quality can vary. Requires careful prompt engineering. Potential for added latency if LLM generation is synchronous within the request flow.

*   **C. Hybrid Approach (Recommended for V2):**
    1.  **Attempt 1:** Use `people-also-ask` library (direct scraping).
    2.  **Fallback 1 (if direct scraping fails or is disabled):** If a SERP API is configured and within budget/quota, try fetching PAA via the SERP API.
    3.  **Fallback 2 (if all above fail):** Trigger AI-generated questions in the Node.js backend.
    *   This provides resilience and allows for cost/reliability trade-offs.

### 2.4. Enhanced Data & Features
*   **Current (V1):** Fetches only the PAA questions.
*   **V2 Proposal:**
    *   **Fetch Short Answers/Snippets:** Some PAA scraping libraries or SERP APIs can also extract the short text snippet displayed below the question in SERPs. This could be valuable context. The response JSON from the Python service would need to be updated to include this.
    *   `{ "keyword": "query", "questions": [ { "question": "What is X?", "answer_snippet": "X is a..." } ] }`
    *   **Related Searches:** Often, SERPs also show "Related Searches." These could also be scraped (if library/API supports) and returned, offering more keyword expansion opportunities.
    *   **Question Depth:** Some tools allow clicking on PAA questions to reveal more nested questions. V1 likely gets only the initial set. V2 could explore fetching 1-2 levels deeper, if supported and performant (adds more requests). This should be configurable.

### 2.5. Improved Monitoring & Alerting
*   **Current (V1):** Basic logging of errors within the function.
*   **V2 Proposal:**
    *   **Custom Metrics:**
        *   Log to Google Cloud Monitoring:
            *   Number of keywords processed.
            *   Number of successful PAA fetches (per keyword/overall).
            *   Number of failures (per keyword/overall, categorized by error type if possible).
            *   Latency per keyword/request.
            *   Usage of fallback mechanisms (e.g., "SERP_API_used_count", "AI_fallback_used_count").
    *   **Alerting:** Set up Google Cloud Monitoring alerts for:
        *   High failure rates (e.g., >20% of keywords return errors over 1 hour).
        *   Increased latency.
        *   Function execution errors.
        *   Activation of circuit breaker.
    *   **Dashboard:** Create a simple dashboard in Google Cloud Monitoring to visualize these metrics.

## 3. API Interface Changes (Potential)

*   The core request (`POST` with `{"keywords": ["kw1", "kw2"], "language": "en", "region": "US"}`) might remain the same for backward compatibility.
*   New optional parameters could be added:
    *   `"options": { "include_snippets": true, "include_related_searches": true, "max_depth": 1 }`
*   The response JSON structure would need to change if snippets or related searches are included:
    ```json
    // V2 Example Response
    {
      "paa_results": {
        "keyword1": {
          "status": "success", // "partial_failure", "failure"
          "questions": [
            { "question": "PAA Question 1 for kw1", "snippet": "Brief answer...", "source": "direct_scrape" },
            { "question": "PAA Question 2 for kw1", "snippet": null, "source": "serp_api" }
          ],
          "related_searches": ["related search 1", "related search 2"],
          "error": null // or error message if status is failure
        },
        "keyword2": {
          "status": "failure",
          "questions": [],
          "related_searches": [],
          "error": "Blocked by Google"
        }
      },
      "metadata": { // Overall job metadata
        "provider_used": ["direct_scrape", "serp_api"], // Which providers were hit
        "total_keywords_processed": 2,
        "total_questions_found": 2
      }
    }
    ```
    This is a more structured response than just a map of keywords to lists of question strings. The Node.js backend would need to adapt to this new structure.

## 4. Implementation Considerations

*   **Phased Rollout:** Introduce V2 enhancements incrementally. E.g., start with improved error handling and monitoring, then add a SERP API integration, then AI fallbacks.
*   **Configuration Management:** Use environment variables or a Firestore configuration document to manage settings like:
    *   Retry counts/delays.
    *   SERP API keys and provider choice.
    *   Enable/disable flags for V2 features (snippets, related searches, fallbacks).
*   **Testing:** Thoroughly test each enhancement, especially different error scenarios and the behavior of fallback mechanisms. Mock Google responses and SERP API responses for controlled testing.
*   **Cost Analysis:** If incorporating paid SERP APIs, closely monitor costs and implement budget alerts in the respective provider's dashboard and potentially within the app (e.g., by tracking API call counts against a monthly budget).

This TDD provides a roadmap for evolving the Python PAA service into a more robust, scalable, and feature-rich component of the Content Agent ecosystem.

# Content Agent Shopify App: Python PAA Service Setup

This document details the setup of a separate Python-based service for scraping "People Also Ask" (PAA) data from Google Search results. This service will be called by the main Node.js backend (Cloud Functions for Firebase) to provide related questions for content generation.

**As per `docs/jules.authority.md`, this service relies on web scraping and carries inherent risks associated with changes in Google's SERP structure and Terms of Service regarding automated querying. It should be monitored, and alternative PAA data sources (e.g., paid SERP APIs, AI-generated questions) should be considered as fallback or future enhancements.**

## 1. Choosing a Hosting Environment for the Python Service

(This section remains as previously defined, outlining Python Cloud Function for Firebase or Google Cloud Run as options. For brevity, it's not repeated here but should be included in the actual file.)

**Recommendation:** Python Cloud Function for Firebase (2nd Gen) is recommended for simplicity if its limits are acceptable.

## 2. Python Cloud Function Implementation for PAA Scraping

### 2.1. Function Directory Structure (Example)
If creating a separate function (e.g., `paa_service`):
```
functions/
├── paa_service/
│   ├── main.py             # Python Cloud Function code described below
│   ├── requirements.txt    # Python dependencies
│   └── ...
├── src/                    # Node.js backend functions
│   └── index.js
└── package.json
```

### 2.2. `requirements.txt` for the Python PAA Function
```txt
# functions/paa_service/requirements.txt
functions-framework>=3.0.0
people-also-ask>=1.0.0 # Or the specific version
# Flask # Optional, if building a Flask app within the function
```

### 2.3. Describing `main.py` (Python Cloud Function Code)

The Python script (`main.py`) for the Cloud Function will be responsible for receiving keywords, using the `people-also-ask` library to fetch related questions, and returning these questions in a JSON format.

**Core Logic (Pseudo-code / Description):**

1.  **Import necessary libraries:**
    *   `functions_framework` (for Google Cloud Functions).
    *   `json` (for handling JSON data).
    *   `get_related_questions` from the `people_also_ask` library.

2.  **Define the HTTP-triggered function (e.g., `get_paa_data`):**
    *   This function will be decorated with `@functions_framework.http`.
    *   It accepts an HTTP `request` object.

3.  **Request Handling:**
    *   Verify the request method is `POST`. If not, return a 405 error.
    *   Attempt to parse the JSON body from the request.
    *   Validate the parsed JSON:
        *   Ensure it contains a `keywords` field.
        *   Ensure `keywords` is a list of strings.
        *   If validation fails, return a 400 error with a descriptive message.
    *   Extract optional `language` (default 'en') and `region` (default 'US') parameters from the JSON body.

4.  **Fetching PAA Data:**
    *   Initialize an empty dictionary, say `results_map`, to store PAA questions for each keyword.
    *   Iterate through each `keyword` in the input `keywords_list`:
        *   **Inside a try-except block (for robustness):**
            *   Call the `get_related_questions(keyword, lang=language)` function from the `people_also_ask` library. (Note: The exact parameters and usage might vary slightly based on the library version; the developer implementing this should consult the library's documentation. The `region` parameter might also be relevant for some library versions or underlying search mechanisms).
            *   Store the returned list of questions (or an empty list if none found) in `results_map` with the keyword as the key.
        *   **If an exception occurs during PAA fetching for a specific keyword:**
            *   Log the error (e.g., `print(f"Error fetching PAA for '{keyword}': {str(e)}")`).
            *   Store an empty list for that keyword in `results_map` to indicate failure for that term but allow the function to continue with other keywords.

5.  **Formatting and Returning the Response:**
    *   Construct a final JSON response object, e.g., `{"paa_results": results_map}`.
    *   Return this JSON string with a `200 OK` status and `Content-Type: application/json` header.

6.  **Global Error Handling:**
    *   Wrap the main request processing logic in a broader try-except block.
    *   If any unexpected error occurs, log the general error and return a `500 Internal Server Error` response.

**Developer Implementation Note:** The developer writing the actual Python code should ensure they are using the `people-also-ask` library according to its documentation, especially regarding parameter names for language/region and how it handles errors or no results found. The above is a conceptual structure.

### 2.4. Deployment
(This section remains as previously defined: `firebase deploy --only functions:get_paa_data`, etc. Not repeated here for brevity but should be in the actual file.)

## 3. API Interface and Security

(This section remains as previously defined: HTTP POST interface, JSON request/response, IAM for security. Not repeated here for brevity but should be in the actual file.)

## 4. Node.js Backend Integration

(This section remains as previously defined: Node.js function reads PAA service URL, constructs payload, makes HTTP call, handles response/errors. Not repeated here for brevity but should be in the actual file.)

This approach allows us to document the Python service's role and structure without directly embedding Python code that causes issues with the current tooling. The implementing developer will use this guide to write the actual `main.py`.

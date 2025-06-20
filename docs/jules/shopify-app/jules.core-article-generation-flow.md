# Content Agent Shopify App: Core Article Generation Flow

This document details the core multi-step process within the "Content Agent" Shopify app that users interact with to generate SEO-enhanced blog articles. It covers both the frontend UI (React/Polaris) aspects and the corresponding backend Cloud Function interactions for each step.

## Overview of the Multi-Step Form

The primary user interaction for creating articles is a multi-step form. The number of steps and available options within each step can vary based on the user's subscription plan.

## Step 1: Product Selection

*   **Purpose:** User selects the Shopify product(s) that the article(s) will be based on.
*   **UI (React/Polaris):**
    *   A Polaris `Autocomplete` component for an instant-suggestion search field. Users start typing a product name.
    *   The app fetches matching products from the user's Shopify store via a backend call.
    *   **Single Product Plans (e.g., Free, Basic):** User selects one product.
    *   **Multi-Product Plans (e.g., Pro, Enterprise):**
        *   The UI allows selecting multiple products, often displayed like tags below the search field. Each selected product tag can be removed.
        *   A "Clear All" button appears if multiple products are selected (e.g., >3).
        *   **Category Selection:** A `Select` list or similar UI to choose a Shopify product category. If selected, the app fetches all products in that category. If the count is within the plan's `productProcessingLimit`, the user can add them all.
        *   **"All Products" Selection:** An option to select all products in the store, enabled only if the total product count is within the plan's `productProcessingLimit`.
*   **Backend Interaction (Cloud Function - e.g., `POST /api/shopify/products/search`):**
    *   Receives: Search query (for autocomplete), category ID (for category selection), or a flag for "all products". `shop_id` from verified session token.
    *   Logic: Uses the stored Shopify access token for the shop to call the Shopify Admin API (Products endpoint) to fetch matching products or products from a category.
    *   Returns: List of product data (ID, title, description, image URL, etc.).
*   **Data Flow:** Selected product(s) data (IDs, titles, descriptions) is stored in the frontend's state to be passed to subsequent steps.

## Step 2: Keyword Generation

*   **Purpose:** Generate relevant keywords based on the selected product(s) and validate them with Google search insights.
*   **UI (React/Polaris):**
    *   Displays a list of AI-generated keywords.
    *   Shows the top X keywords (e.g., 25, based on plan's `keywordGenerationLimit`) with the highest search volume (obtained from Google Ads API).
    *   Each keyword is presented with a `Checkbox` for user selection/deselection.
    *   A "Select All" / "Deselect All" option.
*   **Backend Interaction (Cloud Function - e.g., `POST /api/content/generate-keywords`):**
    *   Receives: List of selected product data (titles, descriptions), `shop_id`.
    *   Logic:
        1.  Sends product information to Gemini AI (or equivalent) to brainstorm an initial list of potential keywords.
        2.  Takes the AI-generated keywords and queries the Google Ads API (Keyword Planner - requires appropriate setup and potentially a linked manager account, or careful use of the API for keyword research) to get search volume and other insights.
        3.  Filters and ranks keywords based on search volume and relevance.
        4.  Returns the top X keywords (as per plan limit) to the frontend.
*   **Data Flow:** Selected keywords are stored in the frontend's state.

## Step 3: Related Questions (Plan-Dependent)

*   **Purpose:** Fetch "People Also Ask" (PAA) style questions related to the selected keywords to enrich the article content. This step is only available if enabled by the user's plan.
*   **UI (React/Polaris):**
    *   Displays a list of PAA questions fetched by the backend.
    *   Each question has a `Checkbox` for user selection/deselection.
    *   The number of questions displayed/selectable can be limited by the plan's `relatedQuestions.maxQuestionsToRetrieve` setting.
*   **Backend Interaction (Cloud Function - e.g., `POST /api/content/get-related-questions`):**
    *   Receives: List of selected keywords, `shop_id`.
    *   Logic:
        1.  Calls the internal Python PAA service (described in `docs/jules/shopify-app/jules.python-paa-service-setup.md`) with the keywords.
        2.  The Python service scrapes Google for PAA data and returns it.
        3.  The Node.js function processes the response from the Python service.
        4.  Returns the list of relevant questions to the frontend, respecting plan limits.
        5.  Handles errors from the PAA service gracefully (e.g., by returning an empty list or an error message).
*   **Data Flow:** Selected questions are stored in the frontend's state.

## Step 4: Article Outline Generation

*   **Purpose:** Create SEO-optimized article outlines based on the selected keywords and (if applicable) related questions.
*   **UI (React/Polaris):**
    *   Displays one or more generated article outlines. Each outline includes:
        *   Suggested H1 Title (editable `TextField`).
        *   A list of H2 headings (editable `TextField`s, reorderable, deletable).
        *   Potentially H3 subheadings under H2s (editable, reorderable, deletable).
        *   Text suggestions for what imagery might be appropriate for different sections.
    *   If multiple products were selected, the UI needs a way to manage outlines for each (e.g., tabs, an accordion, a list of articles to outline).
*   **Backend Interaction (Cloud Function - e.g., `POST /api/content/generate-outlines`):**
    *   Receives: Selected keywords, selected questions (if any), product data, `shop_id`.
    *   Logic:
        1.  Sends the inputs to Gemini AI (or equivalent) with a prompt engineered to create SEO-optimized article outlines (including target character counts for H1, structure for H2s/H3s, and image suggestions).
        2.  If multiple products/keyword clusters are involved, the AI might be prompted to generate multiple distinct outlines.
        3.  Returns the generated outline(s) data structure to the frontend.
*   **Data Flow:** User-edited and approved outline(s) are stored in the frontend's state.

## Multi-Article Flow Management (Interim Note)

If multiple products were selected or if the keyword/question selection naturally leads to multiple distinct article topics, the UI needs a clear way for the user to manage this from Step 4 onwards.

*   **Suggestion from Previous Discussion:** After "Article Outline Generation" for all selected inputs, the user might enter an "Article Editor/Generator Workspace."
    *   This workspace would list all articles for which outlines have been approved.
    *   The user selects one article from this list to proceed with "Article Construction" and "Image Generation" for that specific article.
    *   Once an article is fully processed (constructed, images generated/skipped), they return to this workspace to select the next article.

This approach avoids deeply nested steps within the main multi-step form and provides a clearer batch processing experience. The following steps (Construction, Imaging, Commit) would then apply to one article at a time, selected from this workspace.

This core flow sets the stage for content creation. The subsequent steps of article construction, image generation, and publishing will be detailed in the next document.

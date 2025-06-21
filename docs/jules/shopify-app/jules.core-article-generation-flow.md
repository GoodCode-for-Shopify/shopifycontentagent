# Content Agent Shopify App: Core Article Generation Flow (Initial Batch Steps)

This document details the initial core steps within the "Content Agent" Shopify app that users interact with to initiate the generation of SEO-enhanced blog articles. It covers the frontend UI (React/Polaris) aspects and the corresponding backend Cloud Function interactions for these initial phases, which can apply to a single item or a batch of items (e.g., multiple products or topics) depending on the user's plan.

This flow leads up to the generation of multiple article outlines, which are then managed and processed individually from an "Article Workspace."

## Overview of the Initial Multi-Step Process

The primary user interaction for initiating article creation is a multi-step process. The number of steps and available options within each step can vary based on the user's subscription plan. These initial steps are designed to gather the necessary inputs for a batch of articles.

## Step 1: Product/Topic Selection

*   **Purpose:** User selects the Shopify product(s) or defines the topics that the articles will be based on.
*   **UI (React/Polaris):**
    *   A Polaris `Autocomplete` component for product search.
    *   **Single Item Plans:** User selects one product or defines one topic.
    *   **Multi-Item Plans:**
        *   UI allows selecting multiple products (e.g., displayed as tags).
        *   May include options for category selection or "All Products" if within plan limits (`productProcessingLimit`).
        *   Alternatively, UI may allow defining multiple distinct topics or keyword clusters if not product-based.
*   **Backend Interaction (e.g., `POST /api/shopify/products/search` for products):**
    *   Fetches product data from Shopify if applicable.
*   **Data Flow:** Selected product(s) data or topic definitions are stored in the frontend's state for the batch.

## Step 2: Keyword Generation (for the Batch)

*   **Purpose:** Generate relevant keywords based on the selected product(s) or topics for the entire batch.
*   **UI (React/Polaris):**
    *   Displays AI-generated keywords relevant to the selected batch of items.
    *   Shows top X keywords (e.g., based on plan's `keywordGenerationLimit`) potentially with search volume insights.
    *   Checkboxes for user selection/deselection of keywords to apply to the batch.
*   **Backend Interaction (e.g., `POST /api/content/generate-keywords`):**
    *   Receives: Data for selected products/topics.
    *   Logic: Uses AI (e.g., Gemini) and potentially Google Ads API to generate, validate, and rank keywords for the batch.
    *   Returns: A list of keywords for user selection.
*   **Data Flow:** Keywords selected by the user for the batch are stored in frontend state.

## Step 3: Related Questions (for the Batch, Plan-Dependent)

*   **Purpose:** Fetch "People Also Ask" (PAA) style questions related to the selected keywords for the batch, enriching potential article content. This step is only available if enabled by the user's plan.
*   **UI (React/Polaris):**
    *   Displays a list of PAA questions relevant to the batch's keywords.
    *   Checkboxes for user selection, limited by `relatedQuestions.maxQuestionsToRetrieve` if applicable.
*   **Backend Interaction (e.g., `POST /api/content/get-related-questions`):**
    *   Receives: Selected keywords for the batch.
    *   Logic: Calls the internal Python PAA service (see `jules.python-paa-service-setup.md`) for the keywords.
    *   Returns: List of relevant questions, respecting plan limits.
*   **Data Flow:** Selected questions for the batch are stored in frontend state.

## Step 4: Batch Article Outline Generation

*   **Purpose:** Generate an initial article outline for *each* selected product or topic in the batch, using the finalized keywords and questions.
*   **UI (React/Polaris):**
    *   After keywords and questions are confirmed for the batch, the user initiates this step (e.g., by clicking "Generate All Outlines").
    *   A loading state is displayed while the backend processes the batch request.
*   **Backend Interaction (Cloud Function - `POST /api/content/initiate-multi-article-job`):**
    *   **Reference:** This process is detailed in `docs/jules/shopify-app/tech-design.multi-article-flow.md`.
    *   Receives:
        *   A list of items (selected products or defined topics).
        *   The finalized keywords and questions applicable to the batch.
        *   `shop_id` from the verified session token.
    *   Logic:
        1.  Creates an overall "job" to track the batch.
        2.  For each item in the batch, it triggers the AI-powered outline generation logic.
        3.  Each generated outline is stored as a distinct potential article linked to the job.
    *   Returns: A confirmation that the job has been initiated, along with identifiers for the job and the individual article outlines created (or stubs for them).
*   **Outcome & Next Step - The Article Workspace:**
    *   Upon successful completion of batch outline generation, the user is typically navigated to an "Article Workspace" (or a similar dashboard/list view as described in `tech-design.multi-article-flow.md`).
    *   This workspace displays all the outlines generated from the batch, each representing a potential article (e.g., "Outline for Product A," "Outline for Topic X"). Each will show a status like "Outline Ready for Review."
    *   **Crucially, detailed editing of a specific outline or proceeding to construct the full text for an article occurs *after* this step, by selecting an individual article/outline from the Article Workspace.**

## Transition to Individual Article Processing

Once the batch outline generation is complete and the outlines are available in the Article Workspace, the user transitions from batch setup to processing individual articles one by one.

The user will select a specific article outline from the workspace to:
1.  Review and optionally edit the outline in detail.
2.  Approve the outline.
3.  Proceed to "Article Construction" (generating the full text).
4.  Proceed to "Image Generation" (if applicable).
5.  Finally, commit/export the article.

These subsequent steps of processing a *single selected article* will be detailed in documents like `jules.article-construction-and-imaging.md` and `jules.publishing-and-export.md`, building upon the foundation laid by the `tech-design.multi-article-flow.md`.

This document focuses on the initial phase: gathering inputs for a potential batch of articles and generating all their outlines. The "Article Workspace" then serves as the central hub for the subsequent per-article workflows.

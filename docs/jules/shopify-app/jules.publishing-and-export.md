# Content Agent Shopify App: Publishing & Export (from Article Workspace)

This document covers the final stage of the "Content Agent" Shopify app's article generation workflow: committing generated content to the Shopify store's blog, or exporting it for use on external platforms. These actions are performed on one or more articles that have been fully processed (text constructed, images selected/skipped) and are marked as "Ready to Commit/Export" within the "Article Workspace."

This flow follows the per-article construction and imaging steps (detailed in `jules.article-construction-and-imaging.md`) and aligns with the multi-article management strategy in `docs/jules/shopify-app/tech-design.multi-article-flow.md`.

## Starting Context: Actioning Articles from the Workspace

The user is assumed to be in the "Article Workspace," which displays a list of all articles associated with a multi-article job. Some of these articles will have a status like "Ready to Commit/Export." The user can select one or more of these "Ready" articles to perform publishing or export actions. Each article has a unique `articleId`.

## Step 7: Commit Content & Export Options

*   **Purpose:** Allow the user to save finalized article(s) to their Shopify blog or export them, with rules to prevent duplicate content.
*   **Context:** User is in the Article Workspace and has selected one or more articles with a status indicating they are ready for publishing/export.

### 7.1. UI for Selecting and Actioning Articles (React/Polaris - Article Workspace)

*   **Article List in Workspace:**
    *   The Article Workspace displays all articles (e.g., using Polaris `ResourceList` or `DataTable`), each with its title, status (e.g., "Ready to Commit/Export", "Committed to Shopify", "Exported to WordPress"), and potentially other metadata.
    *   Checkboxes or a selection mechanism allow the user to select one or multiple articles that are in a "Ready to Commit/Export" state.
*   **Action Buttons (apply to selected articles):**
    *   "Commit Selected to Shopify Blog"
    *   "Export Selected..." (could open a sub-menu or modal with download/external platform options)
*   **Review Screen (Before Committing to Shopify - typically per article if bulk committing):**
    *   If committing multiple articles, the system might process them one by one, or show a summary. For a single article commit, a preview screen is appropriate.
    *   Display the full article content.
    *   Final confirmation.
    *   **Metadata Note:** Author tag "Content Agent for Shopify, by GoodCode.ca" will be added.
*   **Export Options Screen (for selected articles, if not committing to Shopify):**
    *   **Download Options:** "Download as Markdown (.md)", "Download as Plain Text (.txt)".
    *   **Direct API Publishing (Paid Plans):**
        *   Platform selection (e.g., WordPress).
        *   Credential handling as described in the original document.
        *   **Important Rule Display:** "Note: Publishing to an external platform means this article cannot also be published to your Shopify Blog via Content Agent to avoid duplicate content."

### 7.2. Backend Logic: Committing to Shopify Blog (Cloud Function)

*   **Endpoint:** e.g., `POST /api/content/commit-to-shopify`
*   **Receives:**
    *   `articleId`: (String) The ID of the specific article from `job_articles` to commit. (If supporting batch commits in one call, this could be `articleIds`: Array of Strings. For V1, per-article calls from a frontend loop might be simpler).
    *   `shop_id` (from verified session token).
    *   (Optionally, if not already part of the article data fetched by `articleId`: `author`, `tags`, `status: "draft"`)
*   **Logic:**
    1.  Fetches the article's data (title, HTML content, featured image URL etc.) from Firestore using the provided `articleId`.
    2.  Uses the shop's stored Shopify access token.
    3.  **Image Handling:** As described in the original document (upload to Shopify CDN, update HTML).
    4.  Calls the Shopify Admin API to create a new blog post, using the fetched article data. The author will be "Content Agent for Shopify, by GoodCode.ca".
    5.  Updates the article's status in Firestore for the given `articleId` to "Committed to Shopify" and stores the `shopifyArticleId`.
*   **Response:** Success message, updated article status. The Article Workspace UI should reflect this change.

### 7.3. Backend Logic: Downloadable Formats (Cloud Function)

*   **Endpoint:** e.g., `POST /api/content/download-article`
*   **Receives:**
    *   `articleId`: (String) The ID of the article to download.
    *   `format`: "markdown" or "plaintext".
    *   `shop_id`.
*   **Logic:**
    1.  Fetches the article's data (HTML content, title) from Firestore using `articleId`.
    2.  Converts HTML to Markdown or plain text as needed.
    3.  Sets appropriate headers for file download.
*   **Response:** The file content. (Status in Firestore might be updated to "Downloaded" if tracking this action is desired).

### 7.4. Backend Logic: API Publishing to External Platforms (Paid Plans)

Focus on **WordPress** example:
*   **Prerequisite:** WordPress credentials stored for the shop.
*   **Endpoint:** e.g., `POST /api/content/publish-to-wordpress`
*   **Receives:**
    *   `articleId`: (String) The ID of the article to publish.
    *   `shop_id`.
    *   (Optionally, `status` for WP post: "draft", "publish")
*   **Logic:**
    1.  Fetches the article's data (title, HTML content) from Firestore using `articleId`.
    2.  Retrieves and decrypts WordPress credentials.
    3.  Uses the WordPress REST API to create a new post with the fetched article data.
    4.  **Image Handling:** As described in the original document (upload to WP media library, update HTML).
    5.  Updates the article's status in Firestore for the given `articleId` to "Exported to WordPress" and stores the `externalPostId`.
*   **Response:** Success/failure message, updated article status. The Article Workspace UI reflects this.

### 7.5. Tracking Published State (per `articleId`)

*   The `job_articles` collection in Firestore is the source of truth for each article's state.
*   Each article document (identified by `articleId`) will have fields like:
    *   `status`: (e.g., "Ready to Commit/Export", "Committed to Shopify", "Exported to WordPress", "Error Publishing").
    *   `shopifyArticleId`: (String, if committed to Shopify).
    *   `externalPlatform`: (String, e.g., "wordpress").
    *   `externalPostId`: (String, ID on the external platform).
*   This per-`articleId` tracking prevents duplicate publishing and allows the Article Workspace UI to accurately display the current state of every article in the job. Users can see at a glance what has been done with each piece of content.

This concludes the primary content generation and output flow. Users manage articles from the Workspace and can see their final destinations.

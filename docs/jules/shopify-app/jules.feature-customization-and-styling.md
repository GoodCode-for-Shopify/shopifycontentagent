# Content Agent Shopify App: Feature Customization & Styling

This document describes how the "Content Agent" Shopify app allows users to customize the structure and basic styling of the blog articles it generates. This ensures the content aligns better with their store's theme and branding.

## 1. Article Structure Customization

The app aims to provide a well-structured HTML output for blog posts. To allow some level of customization and ensure compatibility with various Shopify themes, the app can offer a pre-defined structure that users can slightly influence or that can be injected into their theme.

### 1.1. Providing a Base Structure (HTML/Liquid Snippet)
*   **Concept:** The app can generate content within a defined HTML structure. This structure can be provided to the user as a Liquid snippet.
*   **Mechanism:**
    1.  **Shopify Metafields or Theme App Extensions:**
        *   **Theme App Extensions (Recommended for OS2.0 themes):** The app can use Theme App Extensions to create a custom "Content Agent Article" section or block that merchants can add to their blog post templates (`article.liquid`) via the Shopify Theme Editor. This block would render the app-generated content using a predefined Liquid snippet managed by the app. This is the most modern and user-friendly approach for Online Store 2.0 themes.
        *   **Metafields (Older approach or for data storage):** The app could save the generated HTML content to a Shopify Article Metafield. The merchant would then need to modify their `article.liquid` theme file to display this metafield's content. This is less user-friendly for setup.
    2.  **App-Generated HTML:** The AI-generated content (`body_html` when committing to Shopify) will already have HTML tags (H1, H2, P, etc.). The "structure" here refers more to wrapper elements or class names that the user might want for styling.
*   **Example Snippet (Conceptual - if providing a Liquid snippet for manual theme integration):**
    ```liquid
    <!-- Liquid snippet for Content Agent article -->
    <div class="content-agent-article {{ section.settings.custom_classes }}">
      <h1 class="content-agent-title">{{ article.title }}</h1>

      {% if article.image %}
        <img src="{{ article.image | img_url: 'master' }}" alt="{{ article.image.alt | escape }}" class="content-agent-featured-image">
      {% endif %}

      <div class="content-agent-body">
        {{ article.content }} {# This is where the AI-generated HTML goes #}
      </div>

      <div class="content-agent-cta" style="background-color: {{ section.settings.cta_background_color }};">
        <p>{{ section.settings.cta_text }}</p>
        <a href="{{ section.settings.cta_link }}" class="content-agent-cta-button" style="background-color: {{ section.settings.cta_button_color }}; color: {{ section.settings.cta_button_text_color }};">
          {{ section.settings.cta_button_text }}
        </a>
      </div>

      <p class="content-agent-author-meta">
        Author: {{ article.author }}
      </p>
    </div>
    ```
    This snippet includes placeholders for styling classes and potential custom section settings.

### 1.2. User Configuration for Structure
*   **Within the App UI (Settings Page):**
    *   The app might offer limited options to toggle certain structural elements on/off if the structure is complex (e.g., "Include Author Bio Section," "Show Related Products Section" - though these are more content features than pure structure).
    *   Primarily, the structure will be defined by the app for consistency and SEO best practices. The main customization comes from styling.

## 2. Basic Color Selection Features

To help the generated articles fit the store's branding, the app can allow users to select basic colors for key elements of the generated content, especially if a specific "Content Agent Article" theme section/block is used.

### 2.1. UI for Color Selection (React/Polaris)
*   **Location:** A "Styling" or "Appearance" tab within the Content Agent app's settings page in the Shopify admin.
*   **Components:**
    *   Use Polaris `ColorPicker` components or simpler `TextField` inputs where users can enter hex color codes.
    *   Provide clear labels for each selectable element:
        *   "Call-to-Action Background Color"
        *   "Call-to-Action Button Color"
        *   "Call-to-Action Button Text Color"
        *   "Article Link Color" (for contextual links within the content)
        *   "Heading Color (H2)" (optional, could be global or per heading type)
    *   A "Preview" area showing sample elements with the selected colors applied would be highly beneficial.
*   **Saving Settings:**
    *   Color selections are saved via a backend API call.
    *   The backend stores these preferences in Firestore, associated with the `shop_id` (e.g., in `shops/{shop_id}/stylingPreferences`).

### 2.2. Applying Styles
There are several ways these color selections can be applied:

1.  **Inline Styles (Simpler, More Direct):**
    *   When the backend Cloud Function generates the final HTML for the Shopify blog post (or for export), it can directly embed `style` attributes with the user's selected colors into the relevant HTML elements.
    *   Example (within the AI-generated content or the wrapper snippet):
        `<a href="..." style="color: ${shopStylingPrefs.link_color};">Product Link</a>`
    *   **Pros:** Easy to implement.
    *   **Cons:** Can be overridden by theme CSS if theme uses `!important`. Less flexible for global style changes. Can make HTML more verbose.

2.  **CSS Variables (More Flexible, OS2.0 Friendly):**
    *   If using a Theme App Extension section/block:
        *   The Liquid snippet for the section can define CSS custom properties (variables) based on the section's settings (which the app would populate from Firestore).
        *   Your app's CSS (scoped to the section) would then use these variables.
        *   The app would need to save the user's color choices as settings for that Theme App Extension section via Shopify's Section API or by guiding the user to set them in the Theme Editor.
    *   Example (Liquid for Theme App Extension section):
        ```liquid
        {%- liquid
          assign cta_bg_color = section.settings.cta_background_color | default: '#f0f0f0'
          assign link_color = section.settings.link_color | default: '#007bff'
        -%}
        <style>
          .content-agent-article-{{ section.id }} .content-agent-cta {
            --cta-background: {{ cta_bg_color }};
            background-color: var(--cta-background);
          }
          .content-agent-article-{{ section.id }} .content-agent-body a {
            --link-color: {{ link_color }};
            color: var(--link-color);
          }
        </style>
        <div class="content-agent-article-{{ section.id }}">
          ... content ...
        </div>
        ```
    *   The app would populate `section.settings.cta_background_color` etc., by interacting with Shopify's APIs for section settings if possible, or by instructing the user.
    *   **Pros:** Cleaner HTML, respects theme styling cascade better, more aligned with modern theme development.
    *   **Cons:** More complex to set up initially, relies on OS2.0 theme capabilities.

3.  **Injecting a `<style>` Block (Less Recommended):**
    *   The app could generate a `<style>` block with CSS rules using the selected colors and include this within the `body_html` of the article.
    *   **Pros:** Works even without Theme App Extensions.
    *   **Cons:** Can be messy, might conflict with theme CSS, not always robust.

**Recommendation:**
*   For **structure**, leverage **Theme App Extensions** if possible, providing a well-defined section/block.
*   For **styling**, using **CSS Variables within that Theme App Extension section** is the most flexible and modern approach. The app's UI would guide the user to the Theme Editor to make these color choices if direct API control over section settings is limited for app-owned sections without user interaction in the editor. Alternatively, if the app generates HTML directly, inline styles are the most straightforward way to apply user-defined colors from app settings.

### 2.3. Call-to-Action (CTA) Customization
*   Beyond colors, the admin dashboard (for plan features) or the app settings could allow users to define default text and link for a final Call-to-Action section in their articles.
*   **UI in App Settings:**
    *   `TextField` for "Default CTA Text" (e.g., "Check out our amazing [Product Name]!")
    *   `TextField` for "Default CTA Button Text" (e.g., "Shop Now")
    *   `TextField` for "Default CTA Link Suffix" (e.g., if product pages are always `/products/[product-handle]`, they might just specify a part of it, or the app intelligently links to the selected product).
*   The AI would then be instructed to include a placeholder for this CTA, and the backend would populate it with the user's configured defaults or contextually relevant product links.

By providing these customization options, "Content Agent" can help merchants generate blog content that not only performs well for SEO and conversion but also integrates more seamlessly with their store's unique branding.

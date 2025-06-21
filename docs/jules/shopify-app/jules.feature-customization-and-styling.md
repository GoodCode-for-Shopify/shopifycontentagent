# Content Agent Shopify App: Feature Customization & Styling

This document describes how the "Content Agent" Shopify app allows users to customize the structure and basic styling of the blog articles it generates. This ensures the content aligns better with their store's theme and branding.

## 1. Article Structure Customization

The app aims to provide a well-structured HTML output for blog posts. To allow some level of customization and ensure compatibility with various Shopify themes, the app can offer a pre-defined structure that users can slightly influence or that can be injected into their theme.

### 1.1. Providing a Base Structure (HTML/Liquid Snippet)
*   **Concept:** The app can generate content within a defined HTML structure. This structure can be provided to the user as a Liquid snippet.
*   **Mechanism:**
    1.  **Theme App Extensions (Recommended for OS2.0 themes):** The app can use Theme App Extensions to create a custom "Content Agent Article" section or block that merchants can add to their blog post templates (`article.liquid`) via the Shopify Theme Editor. This block would render the app-generated content using a predefined Liquid snippet managed by the app. This is the most modern and user-friendly approach for Online Store 2.0 themes.
    2.  **Metafields (Older approach or for data storage):** The app could save the generated HTML content to a Shopify Article Metafield. The merchant would then need to modify their `article.liquid` theme file to display this metafield's content. This is less user-friendly for setup.
    3.  **App-Generated HTML:** The AI-generated content (`body_html` when committing to Shopify) will already have HTML tags (H1, H2, P, etc.). The "structure" here refers more to wrapper elements or class names that the user might want for styling.
*   **Example Snippet (Conceptual - if providing a Liquid snippet for manual theme integration or within a Theme App Extension):**
    ```liquid
    <!-- Liquid snippet for Content Agent article -->
    <div class="content-agent-article {{ section.settings.custom_classes | default: 'default-ca-classes' }}">
      <h1 class="content-agent-title">{{ article.title }}</h1>

      {% if article.image %}
        <img src="{{ article.image | img_url: 'master' }}" alt="{{ article.image.alt | escape | default: article.title }}" class="content-agent-featured-image">
      {% endif %}

      <div class="content-agent-body">
        {{ article.content }} {# This is where the AI-generated HTML (body_html) goes #}
      </div>

      {%- comment -%}
      CTA Section - Content for this could be user-defined in app settings
      and inserted by the backend when article.content is finalized,
      or this structure is provided and user populates via theme editor if it's a theme section.
      {%- endcomment -%}
      {% if section.settings.show_cta %}
        <div class="content-agent-cta" style="background-color: {{ section.settings.cta_background_color | default: '#f9f9f9' }};">
          <p>{{ section.settings.cta_text | default: "Check out this amazing product!" }}</p>
          <a href="{{ section.settings.cta_link | default: article.metafields.custom.product_link | default: '#' }}" class="content-agent-cta-button" style="background-color: {{ section.settings.cta_button_color | default: '#007bff' }}; color: {{ section.settings.cta_button_text_color | default: '#ffffff' }};">
            {{ section.settings.cta_button_text | default: "Shop Now" }}
          </a>
        </div>
      {% endif %}

      <p class="content-agent-author-meta">
        Author: {{ article.author }}
      </p>
    </div>

    {% schema %}
    {
      "name": "Content Agent Article",
      "target": "article", // Or section if it's a standalone section for articles
      "settings": [
        {
          "type": "text",
          "id": "custom_classes",
          "label": "Custom CSS Classes for Article Container"
        },
        {
          "type": "checkbox",
          "id": "show_cta",
          "label": "Show Call-to-Action Section",
          "default": true
        },
        {
          "type": "text",
          "id": "cta_text",
          "label": "CTA Text",
          "default": "Check out this amazing product!"
        },
        {
          "type": "url",
          "id": "cta_link",
          "label": "CTA Link (Defaults to product if linked)"
        },
        {
          "type": "text",
          "id": "cta_button_text",
          "label": "CTA Button Text",
          "default": "Shop Now"
        },
        {
          "type": "color",
          "id": "cta_background_color",
          "label": "CTA Background Color",
          "default": "#f9f9f9"
        },
        {
          "type": "color",
          "id": "cta_button_color",
          "label": "CTA Button Background Color",
          "default": "#007bff"
        },
        {
          "type": "color",
          "id": "cta_button_text_color",
          "label": "CTA Button Text Color",
          "default": "#ffffff"
        },
        {
          "type": "color",
          "id": "link_color",
          "label": "Article Body Link Color"
        }
      ]
    }
    {% endschema %}
    ```
    This snippet includes placeholders for styling classes and potential custom section settings that a user could configure via the Shopify Theme Editor if this were part of a Theme App Extension.

### 1.2. User Configuration for Structure
*   **Within the App UI (Settings Page):**
    *   The app might offer limited options to toggle certain structural elements on/off (e.g., "Include Author Meta," "Enable Call-to-Action Section by default").
    *   Primarily, the structure will be defined by the app for consistency and SEO best practices. The main customization comes from styling or theme editor settings if using Theme App Extensions.

## 2. Basic Color Selection Features

To help the generated articles fit the store's branding, the app can allow users to select basic colors for key elements.

### 2.1. UI for Color Selection (React/Polaris in App Settings)
*   **Location:** A "Styling" or "Appearance" tab within the Content Agent app's settings page in the Shopify admin.
*   **Components:**
    *   Use Polaris `ColorPicker` components or `TextField` inputs (type="color") for color selection.
    *   Provide clear labels for each selectable element (these would correspond to settings in the Theme App Extension schema if that approach is taken):
        *   "Default Call-to-Action Background Color"
        *   "Default Call-to-Action Button Color"
        *   "Default Call-to-Action Button Text Color"
        *   "Default Article Link Color"
    *   A "Preview" area showing sample elements with the selected colors applied.
*   **Saving Settings:**
    *   Color selections are saved via a backend API call.
    *   The backend stores these preferences in Firestore, associated with the `shop_id` (e.g., in `shops/{shop_id}/stylingPreferences`). These saved preferences can act as defaults when a new "Content Agent Article" Theme App Extension block is added by the user, or if direct HTML generation with inline styles is used.

### 2.2. Applying Styles

1.  **Theme App Extensions with Section Settings (Recommended):**
    *   The user adds the "Content Agent Article" block/section to their blog post template via the Theme Editor.
    *   The settings defined in the schema (like `cta_background_color`, `link_color` in the example above) are directly configurable by the user *in the Theme Editor*.
    *   The Liquid snippet uses these `section.settings` values to apply styles, often via CSS custom properties.
    *   The app can *pre-populate* these settings when the section is first added, or provide defaults, based on what's saved in the app's settings in Firestore.
    *   **Pros:** Most Shopify-native, user has direct control via Theme Editor, styles are scoped and theme-friendly.
    *   **Cons:** Relies on OS2.0 themes.

2.  **Inline Styles in App-Generated HTML (Fallback/Simpler):**
    *   If not using Theme App Extensions, or for elements directly within the AI-generated `body_html`:
    *   When the backend Cloud Function generates the final HTML for the Shopify blog post, it can directly embed `style` attributes with the user's selected colors (from `shops/{shop_id}/stylingPreferences` in Firestore) into the relevant HTML elements.
    *   Example (within the AI-generated content):
        `<a href="..." style="color: ${shopStylingPrefs.link_color};">Product Link</a>`
    *   **Pros:** Works independently of theme structure.
    *   **Cons:** Can be overridden by theme CSS if theme uses `!important`. Less flexible for global style changes. Can make HTML more verbose.

### 2.3. Call-to-Action (CTA) Content Customization
*   Beyond colors, the app settings should allow users to define default text and link behavior for a Call-to-Action section.
*   **UI in App Settings (React/Polaris):**
    *   `TextField` for "Default CTA Headline Text" (e.g., "Check out our amazing [Product Name]!") - `[Product Name]` could be a placeholder the backend replaces.
    *   `TextField` for "Default CTA Button Text" (e.g., "Shop Now").
    *   Guidance on how CTA links will be generated (e.g., "CTA will automatically link to the primary product selected for the article. You can override this in the Theme Editor if using our Theme Section.").
*   The AI would be instructed to include a placeholder for this CTA, or the Theme App Extension would have fields for it. The backend or Liquid snippet would populate it with the user's configured defaults or contextually relevant product links.

By providing these customization options, "Content Agent" can help merchants generate blog content that not only performs well for SEO and conversion but also integrates more seamlessly with their store's unique branding.

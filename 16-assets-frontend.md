# 16 — Asset Pipeline & Frontend Conventions

---

## 16.1 Default Stack (Rails 7.1+)

```
# Rails defaults:
# - Importmap for JavaScript (no Node.js required)
# - Propshaft for asset pipeline (replaces Sprockets)
# - Turbo for navigation (replaces Turbolinks)
# - Stimulus for JavaScript behavior
# - Tailwind CSS or plain CSS (no Sass by default)

# Alternative stacks (via rails new flags):
# rails new myapp --css=tailwind
# rails new myapp --css=bootstrap
# rails new myapp --javascript=esbuild
# rails new myapp --javascript=webpack (legacy)
```

---

## 16.2 Importmap Conventions

```ruby
# config/importmap.rb
pin "application"                         # app/javascript/application.js
pin "@hotwired/turbo-rails", to: "turbo.min.js"
pin "@hotwired/stimulus", to: "stimulus.min.js"
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"

pin_all_from "app/javascript/controllers", under: "controllers"

# Pin from CDN
pin "lodash", to: "https://cdn.jsdelivr.net/npm/lodash@4.17.21/lodash.min.js"

# Pin local vendor libraries
pin_all_from "vendor/javascript"
```

```javascript
// app/javascript/application.js
import "@hotwired/turbo-rails"
import "controllers"
```

---

## 16.3 Stimulus Controller Conventions

```javascript
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

// Naming: <name>_controller.js → data-controller="<name>"
export default class extends Controller {
  // Targets: DOM elements to interact with
  static targets = ["content", "button"]

  // Values: typed data attributes
  static values = {
    open: { type: Boolean, default: false },
    animationDuration: { type: Number, default: 300 }
  }

  // Classes: CSS class mappings
  static classes = ["active", "hidden"]

  // Lifecycle callbacks
  connect() {
    // Called when controller connects to DOM
  }

  disconnect() {
    // Called when controller disconnects from DOM
  }

  // Actions
  toggle() {
    this.openValue = !this.openValue
  }

  // Value changed callback
  openValueChanged() {
    if (this.openValue) {
      this.contentTarget.classList.remove(this.hiddenClass)
    } else {
      this.contentTarget.classList.add(this.hiddenClass)
    }
  }
}
```

```erb
<%# Usage in views %>
<div data-controller="toggle" data-toggle-open-value="false" data-toggle-hidden-class="hidden">
  <button data-action="click->toggle#toggle" data-toggle-target="button">
    Toggle Content
  </button>
  <div data-toggle-target="content" class="hidden">
    Content here
  </div>
</div>
```

### Stimulus Naming Conventions

```
File name:         clipboard_controller.js
Controller name:   data-controller="clipboard"
Action:            data-action="click->clipboard#copy"
Target:            data-clipboard-target="source"
Value:             data-clipboard-content-value="Hello"
Class:             data-clipboard-active-class="bg-green"

# Multi-word: kebab-case in HTML, camelCase in JS
File:              search_form_controller.js
HTML:              data-controller="search-form"
JS:                this.queryValue (from data-search-form-query-value)
```

---

## 16.4 Turbo Conventions

### Turbo Drive (Page Navigation)

```erb
<%# Turbo Drive is enabled by default — all link clicks and form submissions are Turbo %>

<%# Disable Turbo for specific links %>
<%= link_to "External", "https://example.com", data: { turbo: false } %>

<%# Disable Turbo for specific forms %>
<%= form_with(model: @user, data: { turbo: false }) do |f| %>
```

### Turbo Frames

```erb
<%# Lazy-loaded frame %>
<%= turbo_frame_tag "notifications", src: notifications_path, loading: :lazy do %>
  <p>Loading...</p>
<% end %>

<%# Frame that updates on navigation %>
<%= turbo_frame_tag @user do %>
  <%= render @user %>
  <%= link_to "Edit", edit_user_path(@user) %>
<% end %>

<%# Break out of frame %>
<%= link_to "Full page", user_path(@user), data: { turbo_frame: "_top" } %>
```

### Turbo Streams

```erb
<%# app/views/users/create.turbo_stream.erb %>
<%= turbo_stream.prepend "users" do %>
  <%= render @user %>
<% end %>

<%= turbo_stream.update "user_count" do %>
  <%= User.count %> users
<% end %>

<%= turbo_stream.remove "empty-state" %>
```

```ruby
# From models (broadcast automatically)
class Comment < ApplicationRecord
  belongs_to :post
  after_create_commit -> { broadcast_prepend_to post, target: "comments" }
  after_update_commit -> { broadcast_replace_to post }
  after_destroy_commit -> { broadcast_remove_to post }
end
```

---

## 16.5 CSS Conventions

```
# With Tailwind CSS (recommended)
app/assets/stylesheets/application.tailwind.css

# With plain CSS
app/assets/stylesheets/application.css

# Convention: Component-based organization
app/assets/stylesheets/
├── application.tailwind.css
├── components/
│   ├── buttons.css
│   ├── cards.css
│   └── forms.css
└── utilities/
    └── animations.css
```

---

## 16.6 Propshaft (Asset Pipeline)

```ruby
# Propshaft replaces Sprockets in Rails 7.1+
# Simpler: just serves files from app/assets/ with digest fingerprints
# No compilation, no concatenation — relies on HTTP/2 and importmaps

# Asset helpers in views:
<%= stylesheet_link_tag "application" %>
<%= image_tag "logo.png" %>
<%= javascript_importmap_tags %>
```

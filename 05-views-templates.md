# 05 — Views & Templates Conventions

---

## 5.1 Template Engine Conventions

```
# Default: ERB (Embedded Ruby)
app/views/users/index.html.erb     # HTML + ERB
app/views/users/index.json.jbuilder # JSON (Jbuilder)
app/views/users/index.turbo_stream.erb # Turbo Stream

# Alternative: Haml (requires haml-rails gem)
app/views/users/index.html.haml

# Alternative: Slim (requires slim-rails gem)
app/views/users/index.html.slim

# Convention: Stick with ONE template engine per project
```

---

## 5.2 ERB Tag Conventions

```erb
<%# This is a comment — not rendered in HTML %>

<% # Logic tag — executes Ruby but doesn't output %>
<% if @user.admin? %>
  <span>Admin</span>
<% end %>

<%= # Output tag — executes Ruby and outputs the result (auto-escaped) %>
<%= @user.name %>

<%== # Raw output — UNESCAPED (use with extreme caution) %>
<%== @user.bio_html %>  <!-- Only for pre-sanitized content -->

<%= # Safe alternatives to raw %>
<%= sanitize @user.bio_html %>
<%= @user.bio_html.html_safe %>  <!-- Only if YOU generated the HTML -->
```

---

## 5.3 Layout Conventions

```erb
<!-- app/views/layouts/application.html.erb -->
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for?(:title) ? yield(:title) : "My App" %></title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
  </head>
  <body>
    <%= render "shared/navbar" %>
    <%= render "shared/flash_messages" %>

    <main class="container">
      <%= yield %>
    </main>

    <%= render "shared/footer" %>
  </body>
</html>
```

### Specifying Layouts

```ruby
# Controller-level layout
class AdminController < ApplicationController
  layout "admin"
end

# Action-level layout
def show
  render layout: "minimal"
end

# Conditional layout
layout :determine_layout

private

def determine_layout
  current_user&.admin? ? "admin" : "application"
end
```

### Content For / Yield

```erb
<!-- In layout -->
<head>
  <%= yield :head %>
</head>
<body>
  <%= yield :sidebar if content_for?(:sidebar) %>
  <%= yield %>
</body>

<!-- In view -->
<% content_for :head do %>
  <%= stylesheet_link_tag "special_page" %>
<% end %>

<% content_for :sidebar do %>
  <nav>Sidebar content</nav>
<% end %>

<h1>Main content</h1>
```

---

## 5.4 Partial Conventions

### Naming

```
# Partials always start with underscore in filename
app/views/users/_form.html.erb
app/views/users/_user.html.erb          # Collection partial
app/views/shared/_navbar.html.erb
app/views/shared/_flash_messages.html.erb
```

### Rendering

```erb
<%# Basic partial %>
<%= render "form" %>
<%= render partial: "form" %>

<%# With local variables %>
<%= render "form", user: @user %>
<%= render partial: "form", locals: { user: @user } %>

<%# From another directory %>
<%= render "shared/navbar" %>

<%# Collection rendering (most efficient for lists) %>
<%= render @users %>
<!-- Rails looks for app/views/users/_user.html.erb -->
<!-- Automatically passes each user as local variable `user` -->

<%# Explicit collection rendering %>
<%= render partial: "user", collection: @users %>

<%# Collection with custom variable name %>
<%= render partial: "user", collection: @users, as: :person %>

<%# Collection with spacer %>
<%= render partial: "user", collection: @users, spacer_template: "user_divider" %>

<%# Collection with counter %>
<!-- In _user.html.erb, user_counter is automatically available -->
<%= user_counter %> <!-- 0, 1, 2, ... -->

<%# Empty collection fallback %>
<%= render(@users) || "No users found." %>
```

### Partial Best Practices

```erb
<%# ✅ DO: Use locals, not instance variables in partials %>
<!-- In controller -->
<%= render "user_card", user: @user %>

<!-- In _user_card.html.erb -->
<div class="card"><%= user.name %></div>  <!-- ✅ Local variable -->
<div class="card"><%= @user.name %></div> <!-- ❌ Instance variable -->

<%# ✅ DO: Declare expected locals (Rails 7.1+) %>
<!-- _user_card.html.erb -->
<%# locals: (user:, show_avatar: true) %>
<div class="card">
  <%= image_tag user.avatar if show_avatar %>
  <%= user.name %>
</div>

<%# ✅ DO: Keep partials focused on one thing %>
<%# ❌ DON'T: Create deeply nested partials (max 2-3 levels) %>
```

---

## 5.5 Helper Conventions

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  # Page title helper
  def page_title(title = nil)
    base = "My App"
    title.present? ? "#{title} | #{base}" : base
  end

  # Conditional CSS classes
  def active_link_class(path)
    current_page?(path) ? "active" : ""
  end

  # Format currency
  def format_currency(amount)
    number_to_currency(amount, unit: "₹", delimiter: ",")
  end

  # Time ago in words (with tooltip)
  def time_ago_tag(time)
    tag.time(time_ago_in_words(time) + " ago",
             datetime: time.iso8601,
             title: time.strftime("%B %d, %Y at %I:%M %p"))
  end

  # Boolean display
  def yes_no(value)
    value ? "Yes" : "No"
  end
end

# Convention: Helpers are per-controller but available globally
# Put shared helpers in ApplicationHelper
# Put controller-specific helpers in their own helper file
```

---

## 5.6 Form Conventions

```erb
<%# Rails 7+ form_with (preferred over form_for and form_tag) %>
<%= form_with(model: @user) do |form| %>
  <%# Errors display %>
  <% if @user.errors.any? %>
    <div id="error_explanation">
      <h2><%= pluralize(@user.errors.count, "error") %> prohibited this user from being saved:</h2>
      <ul>
        <% @user.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div class="field">
    <%= form.label :name %>
    <%= form.text_field :name, required: true %>
  </div>

  <div class="field">
    <%= form.label :email %>
    <%= form.email_field :email, required: true %>
  </div>

  <div class="field">
    <%= form.label :role %>
    <%= form.select :role, User::ROLES, include_blank: "Select role" %>
  </div>

  <div class="field">
    <%= form.label :bio %>
    <%= form.text_area :bio, rows: 5 %>
  </div>

  <div class="field">
    <%= form.label :active %>
    <%= form.check_box :active %>
  </div>

  <div class="field">
    <%= form.label :avatar %>
    <%= form.file_field :avatar, accept: "image/*", direct_upload: true %>
  </div>

  <div class="actions">
    <%= form.submit %>
  </div>
<% end %>

<%# Nested attributes %>
<%= form_with(model: @order) do |form| %>
  <%= form.fields_for :line_items do |li_form| %>
    <%= li_form.text_field :product_name %>
    <%= li_form.number_field :quantity %>
    <%= li_form.check_box :_destroy %>
  <% end %>
<% end %>
```

### form_with Defaults

```ruby
# form_with defaults:
# - local: false (submits via Turbo in Rails 7+)
# - Generates proper action URL based on model state (new vs persisted)
# - Generates proper HTTP method (POST for new, PATCH for existing)

# Force local form (no Turbo/Ajax)
form_with(model: @user, local: true)

# Custom URL
form_with(model: @user, url: custom_path)

# Custom method
form_with(url: search_path, method: :get)
```

---

## 5.7 View Components (Rails 7+)

```ruby
# app/components/user_card_component.rb
class UserCardComponent < ViewComponent::Base
  def initialize(user:, show_avatar: true)
    @user = user
    @show_avatar = show_avatar
  end
end

# app/components/user_card_component.html.erb
<div class="user-card">
  <% if @show_avatar %>
    <%= image_tag @user.avatar_url %>
  <% end %>
  <h3><%= @user.name %></h3>
  <p><%= @user.email %></p>
</div>

# Usage in views:
<%= render UserCardComponent.new(user: @user) %>
```

---

## 5.8 Turbo Frames & Streams

```erb
<%# Turbo Frame — lazy loading %>
<%= turbo_frame_tag "user_#{@user.id}" do %>
  <div class="user-card">
    <%= @user.name %>
    <%= link_to "Edit", edit_user_path(@user) %>
  </div>
<% end %>

<%# Turbo Frame — lazy loaded from another URL %>
<%= turbo_frame_tag "recent_posts", src: posts_path, loading: :lazy %>

<%# Turbo Stream response (app/views/users/create.turbo_stream.erb) %>
<%= turbo_stream.prepend "users" do %>
  <%= render @user %>
<% end %>

<%= turbo_stream.update "flash" do %>
  <div class="notice">User created!</div>
<% end %>

<%# Turbo Stream actions: append, prepend, replace, update, remove, before, after %>
```

---

## 5.9 I18n in Views

```erb
<%# Convention: Use I18n for all user-facing text %>
<h1><%= t(".title") %></h1>
<!-- Looks up: en.users.index.title -->

<%= t("activerecord.errors.models.user.attributes.email.blank") %>

<%# With interpolation %>
<%= t(".welcome", name: @user.name) %>
<!-- en.yml: welcome: "Welcome, %{name}!" -->

<%# Pluralization %>
<%= t(".items_count", count: @items.size) %>
<!-- en.yml:
  items_count:
    zero: "No items"
    one: "1 item"
    other: "%{count} items"
-->
```

---

## 5.10 Asset Helpers

```erb
<%= image_tag "logo.png", alt: "Company Logo", class: "logo" %>
<%= stylesheet_link_tag "application" %>
<%= javascript_include_tag "application" %>
<%= favicon_link_tag "favicon.ico" %>
<%= video_tag "intro.mp4", controls: true %>
<%= audio_tag "notification.mp3" %>

<%# Active Storage images %>
<%= image_tag @user.avatar if @user.avatar.attached? %>
<%= image_tag @user.avatar.variant(resize_to_limit: [200, 200]) %>
```

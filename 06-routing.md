# 06 — Routing Conventions

---

## 6.1 RESTful Resource Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Full resource (7 actions: index, show, new, create, edit, update, destroy)
  resources :users

  # Limited actions
  resources :users, only: [:index, :show]
  resources :users, except: [:destroy]

  # Singular resource (no index, no :id in URL — profile belongs to current user)
  resource :profile, only: [:show, :edit, :update]

  # Root route
  root "pages#home"
end
```

### Generated Routes for `resources :users`

| Helper | HTTP Verb | Path | Controller#Action |
|--------|-----------|------|-------------------|
| `users_path` | GET | /users | users#index |
| `new_user_path` | GET | /users/new | users#new |
| `users_path` | POST | /users | users#create |
| `user_path(:id)` | GET | /users/:id | users#show |
| `edit_user_path(:id)` | GET | /users/:id/edit | users#edit |
| `user_path(:id)` | PATCH/PUT | /users/:id | users#update |
| `user_path(:id)` | DELETE | /users/:id | users#destroy |

---

## 6.2 Nested Resources

```ruby
# Convention: Nest only 1 level deep (max 2 in rare cases)
resources :users do
  resources :posts, only: [:index, :create, :new]
end
# /users/:user_id/posts      → posts#index
# /users/:user_id/posts/new  → posts#new
# /users/:user_id/posts      → posts#create

# Show/edit/delete don't need parent context
resources :posts, only: [:show, :edit, :update, :destroy]
# /posts/:id                 → posts#show

# ❌ BAD: Deep nesting
resources :users do
  resources :posts do
    resources :comments do
      resources :likes  # 🚫 Way too deep!
    end
  end
end

# ✅ GOOD: Shallow nesting
resources :users do
  resources :posts, shallow: true
end
# Creates: /users/:user_id/posts (index, new, create)
#          /posts/:id             (show, edit, update, destroy)
```

---

## 6.3 Namespaces

```ruby
# Namespace — adds URL prefix AND module/directory nesting
namespace :admin do
  resources :users       # Admin::UsersController → /admin/users
  resources :reports     # Admin::ReportsController → /admin/reports
  root "dashboard#index" # Admin::DashboardController → /admin
end

# Scope — adds URL prefix only (no module nesting)
scope :admin do
  resources :users       # UsersController → /admin/users
end

# Module — adds module nesting only (no URL prefix)
scope module: :admin do
  resources :users       # Admin::UsersController → /users
end
```

---

## 6.4 API Versioning Routes

```ruby
namespace :api do
  namespace :v1 do
    resources :users, only: [:index, :show, :create, :update, :destroy]
    resources :posts, only: [:index, :show]
  end

  namespace :v2 do
    resources :users, only: [:index, :show, :create, :update, :destroy]
  end
end
# Api::V1::UsersController → /api/v1/users
# Api::V2::UsersController → /api/v2/users
```

---

## 6.5 Member & Collection Routes

```ruby
resources :users do
  # Member routes — operate on a SINGLE resource (has :id)
  member do
    post :activate      # POST /users/:id/activate
    post :deactivate    # POST /users/:id/deactivate
    get :followers      # GET  /users/:id/followers
  end

  # Collection routes — operate on the COLLECTION (no :id)
  collection do
    get :search         # GET /users/search
    post :import        # POST /users/import
    get :export         # GET /users/export
  end
end

# Shorthand (single routes)
resources :users do
  post :activate, on: :member
  get :search, on: :collection
end

# ✅ Preferred: Extract to separate controller instead
resources :users do
  resource :activation, only: [:create, :destroy], controller: "users/activations"
end
```

---

## 6.6 Route Constraints

```ruby
# Segment constraints
get "users/:id", to: "users#show", constraints: { id: /\d+/ }

# Subdomain constraints
constraints subdomain: "api" do
  resources :users
end

# Custom constraint class
class AdminConstraint
  def matches?(request)
    user = User.find_by(id: request.session[:user_id])
    user&.admin?
  end
end

constraints AdminConstraint.new do
  mount Sidekiq::Web, at: "/sidekiq"
end

# IP constraints
constraints(ip: /192\.168\./) do
  resources :debug_tools
end
```

---

## 6.7 Route Concerns

```ruby
# DRY up repeated route patterns
concern :commentable do
  resources :comments, only: [:index, :create, :destroy]
end

concern :taggable do
  resources :tags, only: [:index, :create, :destroy]
end

resources :posts, concerns: [:commentable, :taggable]
resources :photos, concerns: [:commentable]
# /posts/:post_id/comments
# /posts/:post_id/tags
# /photos/:photo_id/comments
```

---

## 6.8 Redirects & Catch-all

```ruby
# Redirects
get "/old-path", to: redirect("/new-path")
get "/old-path/:id", to: redirect("/new-path/%{id}")
get "/legacy", to: redirect { |params, request| "/new?ref=#{request.host}" }

# Catch-all route (MUST be last)
get "*path", to: "errors#not_found", via: :all

# Health check (Rails 7.1+)
get "up", to: "rails/health#show", as: :rails_health_check
```

---

## 6.9 Route Naming Best Practices

```ruby
# Custom named routes
get "dashboard", to: "pages#dashboard", as: :dashboard
get "login", to: "sessions#new", as: :login
post "login", to: "sessions#create"
delete "logout", to: "sessions#destroy", as: :logout

# Convention:
# - Use meaningful names that describe the page/action
# - _path for relative URLs: dashboard_path → "/dashboard"
# - _url for absolute URLs: dashboard_url → "http://example.com/dashboard"
# - Use _url in mailers, _path everywhere else
```

---

## 6.10 Route File Organization

```ruby
# For large apps, split routes into separate files
# config/routes.rb
Rails.application.routes.draw do
  root "pages#home"

  # Core routes
  draw(:authentication)  # config/routes/authentication.rb
  draw(:admin)           # config/routes/admin.rb
  draw(:api)             # config/routes/api.rb

  resources :users
  resources :posts
end

# config/routes/admin.rb
namespace :admin do
  root "dashboard#index"
  resources :users
  resources :settings
end

# config/routes/api.rb
namespace :api do
  namespace :v1 do
    resources :users
  end
end
```

---

## 6.11 Direct & Resolve Routes

```ruby
# Direct routes — custom URL helpers
direct :homepage do
  "https://www.example.com"
end
# homepage_url → "https://www.example.com"

# Resolve — customize polymorphic routing
resolve("User") { [:profile] }
# url_for(@user) → /profile instead of /users/:id
```

# 01 — Naming Conventions

Naming is the single most important convention in Rails. The entire autoloading system, database mapping, and routing engine depend on consistent naming.

---

## 1.1 General Naming Rules

| Context | Convention | Example |
|---------|-----------|---------|
| Classes & Modules | `PascalCase` (CamelCase) | `UserAccount`, `Admin::Dashboard` |
| Methods & Variables | `snake_case` | `first_name`, `calculate_total` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT`, `API_BASE_URL` |
| Predicate methods | End with `?` | `active?`, `admin?`, `valid?` |
| Dangerous methods | End with `!` | `save!`, `destroy!`, `update!` |
| Setter methods | End with `=` | `name=`, `status=` |
| Private methods | No leading underscore | `process_payment` (not `_process_payment`) |
| Files & directories | `snake_case` | `user_account.rb`, `admin/dashboard.rb` |
| Database tables | `snake_case`, plural | `users`, `user_accounts`, `line_items` |
| Database columns | `snake_case`, singular | `first_name`, `created_at`, `user_id` |
| Foreign keys | `singular_table_id` | `user_id`, `category_id` |
| Join tables | Alphabetical, plural | `categories_products` (not `products_categories`) |

---

## 1.2 Model Naming

```
# Model class → Table name → File name → Foreign key
User          → users          → app/models/user.rb          → user_id
LineItem      → line_items     → app/models/line_item.rb     → line_item_id
Person        → people         → app/models/person.rb        → person_id
Category      → categories     → app/models/category.rb      → category_id
Mouse         → mice           → app/models/mouse.rb         → mouse_id
AdminUser     → admin_users    → app/models/admin_user.rb    → admin_user_id
```

### Namespaced Models

```ruby
# app/models/admin/user.rb
# Table: admin_users
module Admin
  class User < ApplicationRecord
    self.table_name = "admin_users"
  end
end
```

### STI (Single Table Inheritance) Naming

```ruby
# All stored in `vehicles` table with `type` column
class Vehicle < ApplicationRecord; end
class Car < Vehicle; end          # type = "Car"
class Truck < Vehicle; end        # type = "Truck"
class ElectricCar < Car; end      # type = "ElectricCar"
```

### Polymorphic Naming

```ruby
# Convention: <association_name>_type and <association_name>_id
# Columns: commentable_type, commentable_id
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
end
```

---

## 1.3 Controller Naming

```
# Controller class         → File name                          → URL path
UsersController            → app/controllers/users_controller.rb            → /users
Admin::UsersController     → app/controllers/admin/users_controller.rb      → /admin/users
Api::V1::UsersController   → app/controllers/api/v1/users_controller.rb     → /api/v1/users
SessionsController         → app/controllers/sessions_controller.rb         → /sessions
LineItemsController        → app/controllers/line_items_controller.rb       → /line_items
```

**Rules:**
- Always **plural** (matches the resource name)
- Suffix with `Controller`
- Inherit from `ApplicationController` (or `ActionController::API` for API-only)
- One controller per resource (don't combine unrelated actions)

---

## 1.4 View Naming

```
# Controller#action          → View template path
UsersController#index        → app/views/users/index.html.erb
UsersController#show         → app/views/users/show.html.erb
UsersController#new          → app/views/users/new.html.erb
UsersController#edit         → app/views/users/edit.html.erb
Admin::UsersController#index → app/views/admin/users/index.html.erb
```

### Partials

```
# Partials always start with underscore
app/views/users/_form.html.erb       → render "form"
app/views/users/_user.html.erb       → render @user (collection partial)
app/views/shared/_navbar.html.erb    → render "shared/navbar"
app/views/layouts/_header.html.erb   → render "layouts/header"
```

### Layouts

```
app/views/layouts/application.html.erb   → Default layout
app/views/layouts/admin.html.erb         → Admin layout
app/views/layouts/mailer.html.erb        → Mailer layout
app/views/layouts/mailer.text.erb        → Plain text mailer layout
```

---

## 1.5 Route & URL Naming

```ruby
# Resource            → Helpers generated
resources :users
# users_path          → /users              (index)
# user_path(@user)    → /users/1            (show)
# new_user_path       → /users/new          (new)
# edit_user_path(@u)  → /users/1/edit       (edit)

# Nested resource
resources :users do
  resources :posts
end
# user_posts_path(@user)        → /users/1/posts
# user_post_path(@user, @post)  → /users/1/posts/2

# Singular resource (no index, no ID in URL)
resource :profile
# profile_path         → /profile
# new_profile_path     → /profile/new
# edit_profile_path    → /profile/edit
```

**Route naming rules:**
- Use `snake_case` for custom route names
- Path helpers end with `_path` (relative) or `_url` (absolute)
- Named routes use `as:` option: `get "dashboard", to: "pages#dashboard", as: :dashboard`

---

## 1.6 Database Column Naming

### Timestamp Columns (Automatic)

```ruby
# Rails automatically manages these:
created_at    # DateTime — set on creation
updated_at    # DateTime — set on every save
```

### Boolean Columns

```ruby
# Prefix with is_/has_ OR use adjective form
# Convention: use adjective form (Rails generates `active?` method)
add_column :users, :active, :boolean, default: false
add_column :users, :admin, :boolean, default: false
add_column :users, :verified, :boolean, default: false
add_column :users, :email_confirmed, :boolean, default: false

# AVOID: is_active, has_verified (Rails already provides ? methods)
```

### Counter Cache Columns

```ruby
# Convention: <plural_association>_count
add_column :posts, :comments_count, :integer, default: 0
# Usage in model:
belongs_to :post, counter_cache: true
```

### Type Columns (for STI)

```ruby
add_column :vehicles, :type, :string
# Rails uses this automatically for Single Table Inheritance
```

### Position/Ordering Columns

```ruby
add_column :items, :position, :integer
# Used with acts_as_list or manual ordering
```

### Token/Slug Columns

```ruby
add_column :users, :token, :string            # Authentication tokens
add_column :articles, :slug, :string           # URL-friendly identifiers
add_column :users, :confirmation_token, :string
add_column :users, :reset_password_token, :string
```

---

## 1.7 Migration Naming

```ruby
# Convention: <timestamp>_<action>_<table/description>.rb
# The name describes what the migration DOES

# Creating tables
20240101120000_create_users.rb
20240101120001_create_line_items.rb

# Adding columns
20240102120000_add_email_to_users.rb
20240102120001_add_status_and_role_to_users.rb

# Removing columns
20240103120000_remove_legacy_field_from_users.rb

# Renaming
20240104120000_rename_name_to_full_name_in_users.rb

# Adding indexes
20240105120000_add_index_to_users_email.rb

# Creating join tables
20240106120000_create_join_table_categories_products.rb

# Data migrations (use descriptive names)
20240107120000_backfill_user_slugs.rb
```

---

## 1.8 Mailer Naming

```ruby
# Class                     → File                                    → Views directory
UserMailer                  → app/mailers/user_mailer.rb              → app/views/user_mailer/
OrderMailer                 → app/mailers/order_mailer.rb             → app/views/order_mailer/
Admin::NotificationMailer   → app/mailers/admin/notification_mailer.rb

# Method names describe the email being sent
class UserMailer < ApplicationMailer
  def welcome_email       # → app/views/user_mailer/welcome_email.html.erb
  def password_reset      # → app/views/user_mailer/password_reset.html.erb
  def confirmation        # → app/views/user_mailer/confirmation.html.erb
end
```

---

## 1.9 Job Naming

```ruby
# Class                          → File
UserCleanupJob                   → app/jobs/user_cleanup_job.rb
SendWelcomeEmailJob              → app/jobs/send_welcome_email_job.rb
ProcessPaymentJob                → app/jobs/process_payment_job.rb
Admin::GenerateReportJob         → app/jobs/admin/generate_report_job.rb

# Convention: <Verb><Noun>Job — describes what the job does
# Always suffix with "Job"
```

---

## 1.10 Helper Naming

```ruby
# Helper module                → File
UsersHelper                    → app/helpers/users_helper.rb
ApplicationHelper              → app/helpers/application_helper.rb
Admin::DashboardHelper         → app/helpers/admin/dashboard_helper.rb

# Convention: matches controller name, available in corresponding views
# ApplicationHelper is available everywhere
```

---

## 1.11 Concern Naming

```ruby
# Model concerns — named as adjectives/traits
module Archivable        # app/models/concerns/archivable.rb
module Searchable        # app/models/concerns/searchable.rb
module Sluggable         # app/models/concerns/sluggable.rb
module Trackable         # app/models/concerns/trackable.rb
module HasAddress        # app/models/concerns/has_address.rb

# Controller concerns — named as verbs/behaviors
module Authenticatable   # app/controllers/concerns/authenticatable.rb
module Paginatable       # app/controllers/concerns/paginatable.rb
module ErrorHandling     # app/controllers/concerns/error_handling.rb
```

---

## 1.12 Serializer / Decorator Naming

```ruby
# Serializer (for API responses)
UserSerializer              → app/serializers/user_serializer.rb
PostSerializer              → app/serializers/post_serializer.rb

# Decorator/Presenter
UserDecorator               → app/decorators/user_decorator.rb
UserPresenter               → app/presenters/user_presenter.rb
```

---

## 1.13 Service Object Naming

```ruby
# Convention: <Verb><Noun> or <Noun><Verb>er
Users::Create               → app/services/users/create.rb
Users::SendWelcomeEmail     → app/services/users/send_welcome_email.rb
Payments::ProcessCharge     → app/services/payments/process_charge.rb
Orders::Calculator          → app/services/orders/calculator.rb

# Alternative flat naming:
CreateUser                  → app/services/create_user.rb
ProcessPayment              → app/services/process_payment.rb
```

---

## 1.14 Test File Naming

```ruby
# Minitest (default Rails)
test/models/user_test.rb
test/controllers/users_controller_test.rb
test/integration/user_flows_test.rb
test/system/users_test.rb
test/mailers/user_mailer_test.rb
test/jobs/user_cleanup_job_test.rb

# RSpec
spec/models/user_spec.rb
spec/requests/users_spec.rb        # Preferred over controller specs
spec/system/users_spec.rb
spec/mailers/user_mailer_spec.rb
spec/jobs/user_cleanup_job_spec.rb
spec/services/users/create_spec.rb
```

---

## 1.15 Gem & Engine Naming

```ruby
# Gem name            → Module         → File structure
my_gem                → MyGem          → lib/my_gem.rb, lib/my_gem/
blorgh                → Blorgh         → lib/blorgh.rb, lib/blorgh/

# Rails Engine
module Blorgh
  class Engine < ::Rails::Engine
    isolate_namespace Blorgh
  end
end
```

---

## 1.16 Inflection Gotchas

Rails uses `ActiveSupport::Inflector` for automatic name transformations. Be aware of:

```ruby
"Person".tableize          # => "people"     (irregular plural)
"Mouse".tableize           # => "mice"
"Octopus".tableize         # => "octopi"
"Datum".tableize           # => "data"
"HTTPSConnection".underscore # => "https_connection"
"API".underscore           # => "api"

# Custom inflections in config/initializers/inflections.rb
ActiveSupport::Inflector.inflections(:en) do |inflect|
  inflect.acronym "API"
  inflect.acronym "SMS"
  inflect.acronym "HTML"
  inflect.irregular "person", "people"
  inflect.uncountable %w[fish sheep species]
end
```

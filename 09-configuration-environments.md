# 09 — Configuration & Environments

---

## 9.1 Environment Configuration

```ruby
# config/environments/development.rb
Rails.application.configure do
  config.cache_classes = false                    # Reload code on change
  config.eager_load = false                       # Don't eager load
  config.consider_all_requests_local = true       # Show detailed errors
  config.action_controller.perform_caching = false
  config.active_storage.service = :local
  config.action_mailer.delivery_method = :letter_opener  # View emails in browser
  config.action_mailer.default_url_options = { host: "localhost", port: 3000 }
  config.active_record.migration_error = :page_load
  config.active_support.deprecation = :log
end

# config/environments/production.rb
Rails.application.configure do
  config.cache_classes = true
  config.eager_load = true
  config.consider_all_requests_local = false
  config.action_controller.perform_caching = true
  config.active_storage.service = :amazon
  config.force_ssl = true
  config.log_level = :info
  config.log_tags = [:request_id]
  config.action_mailer.delivery_method = :smtp
  config.action_mailer.default_url_options = { host: "example.com" }
  config.active_support.deprecation = :notify
  config.active_record.dump_schema_after_migration = false
end

# config/environments/test.rb
Rails.application.configure do
  config.cache_classes = true
  config.eager_load = ENV["CI"].present?          # Eager load in CI
  config.consider_all_requests_local = true
  config.action_controller.perform_caching = false
  config.action_dispatch.show_exceptions = :rescuable
  config.action_mailer.delivery_method = :test
  config.active_storage.service = :test
end
```

---

## 9.2 Credentials & Secrets

```bash
# Rails 7.1+ credentials (encrypted)
# Edit credentials:
EDITOR="code --wait" bin/rails credentials:edit

# Environment-specific credentials:
EDITOR="code --wait" bin/rails credentials:edit --environment production
```

```yaml
# config/credentials.yml.enc (decrypted view)
secret_key_base: abc123...

aws:
  access_key_id: AKIA...
  secret_access_key: secret...
  bucket: my-app-production

stripe:
  publishable_key: pk_live_...
  secret_key: sk_live_...

smtp:
  address: smtp.example.com
  port: 587
  user_name: user
  password: pass
```

```ruby
# Accessing credentials:
Rails.application.credentials.aws[:access_key_id]
Rails.application.credentials.stripe[:secret_key]
Rails.application.credentials.dig(:aws, :access_key_id)

# Environment-specific:
Rails.application.credentials.dig(:smtp, :password)

# Convention:
# - NEVER commit master.key or production.key
# - Share keys securely with team (1Password, Vault, etc.)
# - Use environment variables as fallback: ENV["SECRET_KEY_BASE"]
```

---

## 9.3 Custom Configuration

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    # Custom config namespace
    config.x.payment.retry_count = 3
    config.x.payment.timeout = 30
    config.x.support_email = "support@example.com"
  end
end

# Access:
Rails.configuration.x.payment.retry_count

# Alternative: config_for (loads YAML)
# config/payment.yml
development:
  gateway: sandbox
  retry_count: 3
production:
  gateway: live
  retry_count: 5

# config/application.rb
config.payment = config_for(:payment)
# Access: Rails.configuration.payment.gateway
```

---

## 9.4 Initializers

```ruby
# Convention: One concern per initializer file
# config/initializers/ — runs at boot time

# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "*"
    resource "*", headers: :any, methods: [:get, :post, :patch, :put, :delete, :options]
  end
end

# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0") }
end

# config/initializers/filter_parameter_logging.rb
# ALWAYS filter sensitive parameters
Rails.application.config.filter_parameters += [
  :passw, :secret, :token, :_key, :crypt, :salt,
  :certificate, :otp, :ssn, :credit_card
]

# config/initializers/inflections.rb
ActiveSupport::Inflector.inflections(:en) do |inflect|
  inflect.acronym "API"
  inflect.acronym "HTML"
  inflect.acronym "SMS"
  inflect.acronym "CSV"
end

# config/initializers/time_zone.rb
Rails.application.config.time_zone = "UTC"  # Convention: Always use UTC

# config/initializers/active_model_serializers.rb
ActiveModelSerializers.config.adapter = :json_api
```

---

## 9.5 I18n / Locales

```yaml
# config/locales/en.yml
en:
  activerecord:
    models:
      user: "User"
      post: "Article"
    attributes:
      user:
        name: "Full Name"
        email: "Email Address"
    errors:
      models:
        user:
          attributes:
            email:
              taken: "is already registered"
              invalid: "doesn't look like a valid email"

  helpers:
    submit:
      create: "Create %{model}"
      update: "Update %{model}"

  flash:
    users:
      create:
        success: "Account created successfully!"
      update:
        success: "Profile updated."

  mailers:
    user_mailer:
      welcome:
        subject: "Welcome to MyApp!"

  views:
    pagination:
      previous: "← Previous"
      next: "Next →"
```

### Organizing Locale Files

```
config/locales/
├── en.yml                    # General English
├── en/
│   ├── activerecord.en.yml   # Model translations
│   ├── devise.en.yml         # Devise translations
│   ├── flash.en.yml          # Flash messages
│   ├── mailers.en.yml        # Mailer translations
│   └── views.en.yml          # View translations
├── ta.yml                    # Tamil
└── ta/
    ├── activerecord.ta.yml
    └── views.ta.yml
```

```ruby
# config/application.rb
config.i18n.load_path += Dir[Rails.root.join("config/locales/**/*.yml")]
config.i18n.default_locale = :en
config.i18n.available_locales = [:en, :ta]
config.i18n.fallbacks = true
```

---

## 9.6 Logging Conventions

```ruby
# Convention: Use Rails.logger (not puts or p)
Rails.logger.debug "Debug info: #{variable}"
Rails.logger.info "User #{user.id} logged in"
Rails.logger.warn "Deprecation: method X will be removed"
Rails.logger.error "Failed to process payment: #{error.message}"
Rails.logger.fatal "Database connection lost!"

# Tagged logging
Rails.logger.tagged("PaymentService") do
  Rails.logger.info "Processing payment for order #{order.id}"
end

# Convention: Log levels
# debug — Detailed debugging info (development only)
# info  — General operational info (default in production)
# warn  — Unusual situations that might become errors
# error — Error conditions that were handled
# fatal — Unrecoverable errors

# Custom log formatting
# config/environments/production.rb
config.log_formatter = ::Logger::Formatter.new
config.log_tags = [:request_id]  # Tag logs with request ID
```

---

## 9.7 Environment Variables

```ruby
# Convention: Use ENV.fetch with defaults for required vars
database_url = ENV.fetch("DATABASE_URL") { raise "DATABASE_URL not set!" }
redis_url = ENV.fetch("REDIS_URL", "redis://localhost:6379/0")
port = ENV.fetch("PORT", 3000).to_i

# For development, use dotenv-rails or credentials
# .env (development only, never commit)
DATABASE_URL=postgres://localhost/myapp_dev
REDIS_URL=redis://localhost:6379/0
STRIPE_SECRET_KEY=sk_test_...

# Convention: Use SCREAMING_SNAKE_CASE for env var names
# Convention: Prefix app-specific vars: MYAPP_API_KEY
```

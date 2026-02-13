# 02 — Project Structure & File Organization

---

## 2.1 Standard Rails Directory Structure

```
myapp/
├── app/                          # Core application code
│   ├── assets/                   # Static assets (images, stylesheets)
│   │   ├── images/
│   │   └── stylesheets/
│   │       └── application.css
│   ├── channels/                 # Action Cable channels
│   │   └── application_cable/
│   │       ├── channel.rb
│   │       └── connection.rb
│   ├── controllers/              # Request handling
│   │   ├── concerns/             # Shared controller modules
│   │   └── application_controller.rb
│   ├── helpers/                  # View helper methods
│   │   └── application_helper.rb
│   ├── javascript/               # JavaScript (importmaps or bundled)
│   │   ├── controllers/          # Stimulus controllers
│   │   └── application.js
│   ├── jobs/                     # Background jobs (Active Job)
│   │   └── application_job.rb
│   ├── mailers/                  # Email sending
│   │   └── application_mailer.rb
│   ├── models/                   # Business logic & data access
│   │   ├── concerns/             # Shared model modules
│   │   └── application_record.rb
│   └── views/                    # Templates
│       ├── layouts/              # Layout templates
│       │   ├── application.html.erb
│       │   ├── mailer.html.erb
│       │   └── mailer.text.erb
│       └── shared/               # Shared partials
├── bin/                          # Executable scripts
│   ├── bundle
│   ├── dev                       # Development server (Rails 8)
│   ├── docker-entrypoint
│   ├── rails
│   ├── rake
│   └── setup
├── config/                       # Configuration
│   ├── environments/             # Per-environment config
│   │   ├── development.rb
│   │   ├── production.rb
│   │   └── test.rb
│   ├── initializers/             # Boot-time configuration
│   │   ├── assets.rb
│   │   ├── content_security_policy.rb
│   │   ├── filter_parameter_logging.rb
│   │   ├── inflections.rb
│   │   └── permissions_policy.rb
│   ├── locales/                  # I18n translation files
│   │   └── en.yml
│   ├── application.rb            # Application-level config
│   ├── boot.rb                   # Bundler setup
│   ├── cable.yml                 # Action Cable config
│   ├── credentials.yml.enc       # Encrypted credentials
│   ├── database.yml              # Database config
│   ├── environment.rb            # Environment loader
│   ├── master.key                # Encryption key (NEVER commit)
│   ├── puma.rb                   # Web server config
│   ├── routes.rb                 # URL routing
│   └── storage.yml               # Active Storage config
├── db/                           # Database files
│   ├── migrate/                  # Migration files
│   ├── schema.rb                 # Current schema (auto-generated)
│   └── seeds.rb                  # Seed data
├── lib/                          # Extended modules & tasks
│   ├── assets/                   # Library-level assets
│   └── tasks/                    # Custom Rake tasks (.rake files)
├── log/                          # Log files (never commit)
├── public/                       # Static files served directly
│   ├── 404.html
│   ├── 422.html
│   ├── 500.html
│   ├── favicon.ico
│   └── robots.txt
├── storage/                      # Active Storage files (dev)
├── test/ (or spec/)              # Tests
│   ├── controllers/
│   ├── fixtures/                 # Test data (YAML)
│   ├── helpers/
│   ├── integration/
│   ├── jobs/
│   ├── mailers/
│   ├── models/
│   ├── system/                   # Browser tests
│   ├── application_system_test_case.rb
│   └── test_helper.rb
├── tmp/                          # Temporary files
├── vendor/                       # Third-party code
├── .gitignore
├── .ruby-version                 # Ruby version specification
├── Dockerfile                    # Container config (Rails 7.1+)
├── Gemfile                       # Gem dependencies
├── Gemfile.lock                  # Locked gem versions (always commit)
├── Procfile.dev                  # Dev process manager
├── Rakefile                      # Rake configuration
└── README.md                     # Project documentation
```

---

## 2.2 Custom Directories (Common Additions)

```
app/
├── builders/                # Builder pattern objects
├── components/              # ViewComponent classes
├── decorators/              # Draper decorators or custom
├── forms/                   # Form objects
├── notifiers/               # Notification dispatchers
├── policies/                # Pundit authorization policies
├── presenters/              # Presenter/ViewModel objects
├── queries/                 # Query objects
├── serializers/             # API serializers (AMS, Blueprinter, Alba)
├── services/                # Service objects
│   ├── users/
│   │   ├── create.rb
│   │   └── send_welcome_email.rb
│   └── payments/
│       └── process_charge.rb
├── validators/              # Custom validators
└── value_objects/           # Value objects (Money, Address, etc.)
```

**Convention:** Any directory under `app/` is automatically added to the autoload path. You do NOT need to manually configure autoloading for new directories inside `app/`.

---

## 2.3 Autoloading with Zeitwerk

Rails 7+ uses Zeitwerk exclusively. Key conventions:

### File-to-Constant Mapping

```ruby
# File path                          → Constant
app/models/user.rb                   → User
app/models/admin/user.rb             → Admin::User
app/controllers/api/v1/users_controller.rb → Api::V1::UsersController
app/services/payments/process_charge.rb    → Payments::ProcessCharge
```

### Rules

```ruby
# 1. One constant per file (matching the file name)
# app/models/user.rb
class User < ApplicationRecord    # ✅ Correct
end

# 2. Directory names become module namespaces
# app/models/admin/user.rb
module Admin
  class User < ApplicationRecord  # ✅ Correct
  end
end

# 3. Acronyms must be configured
# config/initializers/inflections.rb
ActiveSupport::Inflector.inflections(:en) do |inflect|
  inflect.acronym "API"           # Allows Api::V1 → api/v1/
  inflect.acronym "HTML"
  inflect.acronym "SMS"
end

# 4. Concerns directory is a "collapsed" directory
# app/models/concerns/searchable.rb → Searchable (NOT Concerns::Searchable)
# app/controllers/concerns/authenticatable.rb → Authenticatable

# 5. Never use require/require_relative for app/ code
# Zeitwerk handles autoloading automatically
require "some_gem"              # ✅ For gems
require_relative "../lib/util"  # ✅ For lib/ code (if not autoloaded)
require "user"                  # ❌ Never for app/ code
```

### Configuring Autoload Paths

```ruby
# config/application.rb
# lib/ is NOT autoloaded by default in Rails 7+
# To autoload lib/:
config.autoload_lib(ignore: %w[assets tasks])

# Custom autoload paths (rarely needed since app/ dirs auto-register):
config.autoload_paths << Rails.root.join("extras")
```

---

## 2.4 The `lib/` Directory

```
lib/
├── assets/                    # Assets for libs (rarely used)
├── tasks/                     # Custom Rake tasks
│   ├── db.rake                # Database tasks
│   └── maintenance.rake       # Maintenance tasks
├── middleware/                 # Custom Rack middleware
│   └── rate_limiter.rb
├── extensions/                # Core class extensions
│   └── string_extensions.rb
└── my_app/                    # App-specific library code
    ├── csv_importer.rb
    └── pdf_generator.rb
```

**Convention for Rake tasks:**

```ruby
# lib/tasks/users.rake
namespace :users do
  desc "Clean up inactive users older than 90 days"
  task cleanup: :environment do
    User.inactive.where("last_login_at < ?", 90.days.ago).destroy_all
  end
end

# Run with: bin/rails users:cleanup
```

---

## 2.5 Configuration File Conventions

```ruby
# config/application.rb — Global settings
# config/environments/development.rb — Dev-only settings
# config/environments/production.rb — Prod-only settings
# config/environments/test.rb — Test-only settings

# Convention: environment-specific configs override application.rb

# Custom configuration
# config/application.rb
config.x.payment_processing.retry_count = 3
config.x.payment_processing.timeout = 30

# Access anywhere:
Rails.configuration.x.payment_processing.retry_count
```

---

## 2.6 Initializer Ordering

Initializers run alphabetically. Use numeric prefixes if order matters:

```
config/initializers/
├── 01_core_extensions.rb
├── 02_redis.rb
├── assets.rb
├── content_security_policy.rb
├── cors.rb
├── filter_parameter_logging.rb
├── inflections.rb
├── permissions_policy.rb
└── sidekiq.rb
```

**Convention:** Only use numeric prefixes when initialization order is genuinely important.

---

## 2.7 `.gitignore` Conventions

These files should NEVER be committed:

```gitignore
# Ignore master key for decrypting credentials
config/master.key
config/credentials/*.key

# Ignore log files
/log/*
!/log/.keep

# Ignore tmp files
/tmp/*
!/tmp/.keep

# Ignore storage files (development uploads)
/storage/*
!/storage/.keep

# Ignore node_modules (if using Node-based bundling)
/node_modules

# Ignore OS files
.DS_Store
Thumbs.db

# Ignore IDE files
.idea/
.vscode/
*.swp

# Ignore environment files (if using dotenv)
.env
.env.local
.env.*.local
```

These files SHOULD be committed:

```
Gemfile.lock        # Ensures consistent gem versions
db/schema.rb        # Current database schema
.ruby-version       # Ruby version
yarn.lock           # If using Yarn
```

---

## 2.8 Engines & Mountable Engines

```
# Full engine structure
my_engine/
├── app/
│   ├── controllers/my_engine/
│   ├── models/my_engine/
│   └── views/my_engine/
├── config/
│   └── routes.rb
├── db/
│   └── migrate/
├── lib/
│   ├── my_engine.rb
│   └── my_engine/
│       ├── engine.rb
│       └── version.rb
├── test/
├── Gemfile
├── my_engine.gemspec
└── Rakefile
```

**Convention:** Mountable engines isolate their namespace to avoid conflicts with the host app.

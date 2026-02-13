# 07 — Database & Migration Conventions

---

## 7.1 Migration File Conventions

```ruby
# Generate migrations via CLI
# rails generate migration CreateUsers name:string email:string
# rails generate migration AddEmailToUsers email:string:index
# rails generate migration RemoveAgeFromUsers age:integer

# Convention: Migration names describe the change
# Create:  Create<TableName>
# Add:     Add<Column>To<Table>
# Remove:  Remove<Column>From<Table>
# Rename:  Rename<Old>To<New>In<Table>
# Change:  Change<Column>In<Table>
```

### Creating Tables

```ruby
class CreateUsers < ActiveRecord::Migration[7.1]
  def change
    create_table :users do |t|
      t.string :name, null: false
      t.string :email, null: false
      t.string :password_digest, null: false
      t.string :role, null: false, default: "member"
      t.boolean :active, null: false, default: true
      t.text :bio
      t.date :date_of_birth
      t.integer :login_count, null: false, default: 0
      t.decimal :balance, precision: 10, scale: 2, default: 0.0
      t.references :organization, null: false, foreign_key: true
      t.timestamps  # Adds created_at and updated_at
    end

    add_index :users, :email, unique: true
    add_index :users, :role
    add_index :users, [:organization_id, :email], unique: true
  end
end
```

### Column Types

| Rails Type | PostgreSQL | MySQL | Use For |
|-----------|-----------|-------|---------|
| `:string` | varchar(255) | varchar(255) | Short text (name, email) |
| `:text` | text | text | Long text (bio, description) |
| `:integer` | integer | int(11) | Whole numbers |
| `:bigint` | bigint | bigint | Large numbers, foreign keys |
| `:float` | float | float | Approximate decimals |
| `:decimal` | decimal | decimal | Exact decimals (money!) |
| `:boolean` | boolean | tinyint(1) | True/false |
| `:date` | date | date | Date only |
| `:datetime` | timestamp | datetime | Date and time |
| `:time` | time | time | Time only |
| `:binary` | bytea | blob | Binary data |
| `:json` | json | json | JSON data |
| `:jsonb` | jsonb | N/A | Indexed JSON (PostgreSQL) |
| `:uuid` | uuid | char(36) | UUIDs |
| `:inet` | inet | N/A | IP addresses |
| `:cidr` | cidr | N/A | Network addresses |
| `:daterange` | daterange | N/A | Date ranges |
| `:tsrange` | tsrange | N/A | Timestamp ranges |
| `:hstore` | hstore | N/A | Key-value pairs |

---

## 7.2 Migration Best Practices

### Reversible Migrations

```ruby
# ✅ Use `change` method when Rails can auto-reverse
class AddStatusToOrders < ActiveRecord::Migration[7.1]
  def change
    add_column :orders, :status, :string, null: false, default: "pending"
    add_index :orders, :status
  end
end

# ✅ Use `up` and `down` when Rails can't auto-reverse
class ChangeStatusColumnType < ActiveRecord::Migration[7.1]
  def up
    change_column :orders, :status, :integer, using: "status::integer"
  end

  def down
    change_column :orders, :status, :string
  end
end

# ✅ Use `reversible` for mixed operations
class AddFullTextSearch < ActiveRecord::Migration[7.1]
  def change
    reversible do |dir|
      dir.up do
        execute <<-SQL
          CREATE INDEX users_search_idx ON users
          USING gin(to_tsvector('english', name || ' ' || email));
        SQL
      end
      dir.down do
        execute "DROP INDEX users_search_idx;"
      end
    end
  end
end
```

### Data Integrity Conventions

```ruby
# ✅ ALWAYS add NOT NULL constraints for required fields
add_column :users, :email, :string, null: false

# ✅ ALWAYS add default values for boolean and counter columns
add_column :users, :active, :boolean, null: false, default: true
add_column :posts, :comments_count, :integer, null: false, default: 0

# ✅ ALWAYS add foreign key constraints
add_reference :posts, :user, null: false, foreign_key: true

# ✅ ALWAYS add unique indexes for uniqueness validations
add_index :users, :email, unique: true
add_index :users, [:organization_id, :email], unique: true

# ✅ Add indexes for columns used in WHERE, ORDER BY, and JOIN
add_index :users, :role
add_index :orders, :status
add_index :posts, :created_at
add_index :posts, [:user_id, :created_at]

# ✅ Use check constraints (Rails 6.1+)
add_check_constraint :orders, "total >= 0", name: "orders_total_non_negative"
add_check_constraint :users, "age >= 0 AND age <= 150", name: "users_age_valid"
```

### Never in Migrations

```ruby
# ❌ NEVER reference models in migrations (they may change or be deleted)
class BackfillUserSlugs < ActiveRecord::Migration[7.1]
  def up
    User.find_each { |u| u.update!(slug: u.name.parameterize) }  # ❌
  end
end

# ✅ Use raw SQL or inline class
class BackfillUserSlugs < ActiveRecord::Migration[7.1]
  class User < ApplicationRecord; end  # Inline class, frozen in time

  def up
    User.find_each do |user|
      user.update_column(:slug, user.name.parameterize)
    end
  end
end

# ❌ NEVER modify existing migrations that have been run in production
# ✅ Create a new migration to fix issues

# ❌ NEVER use `change_column` without checking for data loss
# ✅ Always handle data transformation explicitly
```

---

## 7.3 Index Conventions

```ruby
# Single column index
add_index :users, :email

# Unique index
add_index :users, :email, unique: true

# Composite index (order matters — put most selective first)
add_index :orders, [:user_id, :created_at]

# Partial index (PostgreSQL) — index only matching rows
add_index :users, :email, where: "active = true", name: "index_active_users_on_email"

# Covering index (PostgreSQL 11+)
add_index :users, :email, include: [:name, :role]

# GIN index for JSONB
add_index :users, :metadata, using: :gin

# Expression index
add_index :users, "lower(email)", unique: true, name: "index_users_on_lower_email"

# Naming convention for indexes:
# Default: index_<table>_on_<column>
# Custom: Use descriptive names for complex indexes
```

---

## 7.4 Foreign Key Conventions

```ruby
# Via references (preferred)
t.references :user, null: false, foreign_key: true

# Explicit foreign key
add_foreign_key :posts, :users
add_foreign_key :posts, :users, column: :author_id

# With cascade options
add_foreign_key :comments, :posts, on_delete: :cascade
add_foreign_key :line_items, :orders, on_delete: :cascade

# Options:
# on_delete: :cascade    — delete child when parent deleted
# on_delete: :nullify    — set FK to NULL when parent deleted
# on_delete: :restrict   — prevent parent deletion if children exist (default)
```

---

## 7.5 Schema.rb Conventions

```ruby
# db/schema.rb is AUTO-GENERATED — never edit manually
# It represents the current state of the database
# ALWAYS commit schema.rb to version control

# To regenerate from database:
# rails db:schema:dump

# For complex schemas (custom SQL, functions, triggers):
# Use structure.sql instead:
# config/application.rb
config.active_record.schema_format = :sql
# This generates db/structure.sql instead
```

---

## 7.6 Seed Data Conventions

```ruby
# db/seeds.rb
# Convention: Seeds should be idempotent (safe to run multiple times)

# ✅ Use find_or_create_by
admin = User.find_or_create_by!(email: "admin@example.com") do |user|
  user.name = "Admin"
  user.password = "password"
  user.role = "admin"
end

# ✅ Environment-specific seeds
if Rails.env.development?
  50.times do |i|
    User.find_or_create_by!(email: "user#{i}@example.com") do |user|
      user.name = Faker::Name.name
      user.password = "password"
    end
  end
end

# ✅ Organize large seed files
# db/seeds/01_roles.rb
# db/seeds/02_users.rb
# db/seeds/03_products.rb

# In db/seeds.rb:
Dir[Rails.root.join("db/seeds/*.rb")].sort.each { |seed| load seed }
```

---

## 7.7 UUID Primary Keys

```ruby
# config/initializers/generators.rb (for entire app)
Rails.application.config.generators do |g|
  g.orm :active_record, primary_key_type: :uuid
end

# Per-migration
class CreateOrders < ActiveRecord::Migration[7.1]
  def change
    create_table :orders, id: :uuid do |t|
      t.references :user, type: :uuid, null: false, foreign_key: true
      t.timestamps
    end
  end
end

# Enable pgcrypto extension (PostgreSQL)
class EnableUuidExtension < ActiveRecord::Migration[7.1]
  def change
    enable_extension "pgcrypto"
  end
end
```

---

## 7.8 Database Configuration

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: myapp_development

test:
  <<: *default
  database: myapp_test

production:
  <<: *default
  url: <%= ENV["DATABASE_URL"] %>

# Convention:
# - Database names: <app_name>_<environment>
# - Use environment variables for production credentials
# - Never hardcode passwords in database.yml
```

---

## 7.9 Multiple Databases (Rails 6+)

```yaml
# config/database.yml
production:
  primary:
    database: myapp_production
    adapter: postgresql
  primary_replica:
    database: myapp_production
    adapter: postgresql
    replica: true
  animals:
    database: myapp_animals_production
    adapter: postgresql
    migrations_paths: db/animals_migrate
```

```ruby
# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class
  connects_to database: { writing: :primary, reading: :primary_replica }
end

# app/models/animals_record.rb
class AnimalsRecord < ApplicationRecord
  self.abstract_class = true
  connects_to database: { writing: :animals }
end
```

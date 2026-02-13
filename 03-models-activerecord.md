# 03 — Models & Active Record Conventions

---

## 3.1 Model File Structure Convention

Every model should follow this ordering:

```ruby
class User < ApplicationRecord
  # 1. Constants
  ROLES = %w[admin moderator member].freeze
  MAX_LOGIN_ATTEMPTS = 5

  # 2. Attribute declarations
  enum :status, { active: 0, inactive: 1, suspended: 2 }
  enum :role, { member: 0, admin: 1, moderator: 2 }

  # 3. Associations (in this order)
  belongs_to :organization
  belongs_to :created_by, class_name: "User", optional: true

  has_one :profile, dependent: :destroy
  has_one :address, through: :profile

  has_many :posts, dependent: :destroy
  has_many :comments, dependent: :destroy
  has_many :commented_posts, through: :comments, source: :post

  has_and_belongs_to_many :roles  # Avoid; prefer has_many :through

  has_one_attached :avatar
  has_many_attached :documents

  # 4. Delegations
  delegate :city, :state, :zip, to: :address, prefix: true, allow_nil: true
  delegate :name, to: :organization, prefix: true

  # 5. Validations
  validates :email, presence: true,
                    uniqueness: { case_sensitive: false },
                    format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :name, presence: true, length: { maximum: 100 }
  validates :role, inclusion: { in: ROLES }

  # 6. Callbacks (in lifecycle order)
  before_validation :normalize_email
  before_save :set_defaults
  after_create :send_welcome_email
  after_commit :sync_to_external_service, on: [:create, :update]

  # 7. Scopes
  scope :active, -> { where(status: :active) }
  scope :admins, -> { where(role: :admin) }
  scope :recent, -> { order(created_at: :desc) }
  scope :created_after, ->(date) { where("created_at > ?", date) }
  scope :search, ->(query) { where("name ILIKE ?", "%#{query}%") }

  # 8. Class methods
  def self.find_by_credentials(email, password)
    user = find_by(email: email.downcase)
    user&.authenticate(password) ? user : nil
  end

  # 9. Instance methods (public)
  def full_name
    "#{first_name} #{last_name}".strip
  end

  def admin?
    role == "admin"
  end

  # 10. Private methods
  private

  def normalize_email
    self.email = email&.downcase&.strip
  end

  def set_defaults
    self.role ||= "member"
  end

  def send_welcome_email
    UserMailer.welcome_email(self).deliver_later
  end

  def sync_to_external_service
    SyncUserJob.perform_later(id)
  end
end
```

---

## 3.2 Association Conventions

### belongs_to

```ruby
# Convention: singular, references foreign key <association>_id
belongs_to :user                                          # user_id column
belongs_to :author, class_name: "User"                    # author_id column
belongs_to :parent, class_name: "Category", optional: true # parent_id, allows nil
belongs_to :commentable, polymorphic: true                 # commentable_type + commentable_id

# Rails 7+: belongs_to is required by default
# Use optional: true to allow nil foreign keys
belongs_to :manager, optional: true
```

### has_one

```ruby
has_one :profile, dependent: :destroy
has_one :address, through: :profile
has_one :latest_post, -> { order(created_at: :desc) }, class_name: "Post"

# Convention: ALWAYS specify dependent option
# :destroy  — call destroy on associated record (runs callbacks)
# :delete   — delete directly (skips callbacks)
# :nullify  — set foreign key to NULL
# :restrict_with_error — prevent deletion if associated record exists
# :restrict_with_exception — raise error if associated record exists
```

### has_many

```ruby
has_many :posts, dependent: :destroy
has_many :published_posts, -> { where(published: true) }, class_name: "Post"
has_many :comments, as: :commentable  # Polymorphic
has_many :active_comments, -> { where(active: true) }, class_name: "Comment"

# Through associations
has_many :taggings, dependent: :destroy
has_many :tags, through: :taggings

# Self-referential
has_many :subordinates, class_name: "Employee", foreign_key: "manager_id"
belongs_to :manager, class_name: "Employee", optional: true
```

### has_and_belongs_to_many (HABTM)

```ruby
# AVOID HABTM — prefer has_many :through
# HABTM requires a join table with NO model

# Only use for pure many-to-many with NO additional attributes
# Join table naming: alphabetical order, both plural
# Table: categories_products (NOT products_categories)

# Migration for join table:
create_join_table :categories, :products do |t|
  t.index [:category_id, :product_id]
  t.index [:product_id, :category_id]
end
```

### Preferred: has_many :through

```ruby
# Always preferred over HABTM
class User < ApplicationRecord
  has_many :user_roles, dependent: :destroy
  has_many :roles, through: :user_roles
end

class UserRole < ApplicationRecord
  belongs_to :user
  belongs_to :role
  # Can have additional attributes: granted_at, granted_by, etc.
end

class Role < ApplicationRecord
  has_many :user_roles, dependent: :destroy
  has_many :users, through: :user_roles
end
```

---

## 3.3 Validation Conventions

```ruby
# Presence
validates :name, presence: true
validates :email, presence: { message: "must be provided" }

# Uniqueness (always add database-level unique index too!)
validates :email, uniqueness: { case_sensitive: false, scope: :organization_id }
validates :slug, uniqueness: true

# Length
validates :name, length: { minimum: 2, maximum: 100 }
validates :bio, length: { maximum: 500 }
validates :password, length: { in: 8..128 }

# Numericality
validates :age, numericality: { only_integer: true, greater_than: 0 }
validates :price, numericality: { greater_than_or_equal_to: 0 }
validates :quantity, numericality: { only_integer: true, in: 1..999 }

# Format
validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
validates :phone, format: { with: /\A\+?[\d\s\-()]+\z/ }
validates :slug, format: { with: /\A[a-z0-9\-]+\z/, message: "only allows lowercase letters, numbers, and hyphens" }

# Inclusion / Exclusion
validates :role, inclusion: { in: %w[admin moderator member] }
validates :status, exclusion: { in: %w[deleted] }

# Conditional validations
validates :company_name, presence: true, if: :business_account?
validates :phone, presence: true, unless: -> { email.present? }
validates :reason, presence: true, if: :cancelling?

# Custom validator class
validates :email, email_format: true  # app/validators/email_format_validator.rb

# Custom validation method
validate :end_date_after_start_date

private

def end_date_after_start_date
  return if end_date.blank? || start_date.blank?
  errors.add(:end_date, "must be after start date") if end_date <= start_date
end
```

### Custom Validator Convention

```ruby
# app/validators/email_format_validator.rb
class EmailFormatValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    unless value =~ URI::MailTo::EMAIL_REGEXP
      record.errors.add(attribute, options[:message] || "is not a valid email")
    end
  end
end
```

---

## 3.4 Callback Conventions

### Lifecycle Order

```ruby
# Creation:  before_validation → after_validation → before_save → around_save →
#            before_create → around_create → after_create → after_save → after_commit

# Update:   before_validation → after_validation → before_save → around_save →
#            before_update → around_update → after_update → after_save → after_commit

# Destroy:  before_destroy → around_destroy → after_destroy → after_commit
```

### Best Practices

```ruby
# ✅ DO: Use callbacks for data normalization
before_validation :normalize_email
before_save :generate_slug

# ✅ DO: Use after_commit for external side effects
after_commit :sync_to_search_index, on: [:create, :update]
after_commit :send_notification, on: :create

# ❌ DON'T: Use callbacks for complex business logic
# Move to service objects instead
after_create :charge_credit_card           # ❌ Too complex for callback
after_create :create_stripe_customer       # ❌ External service call

# ❌ DON'T: Use callbacks that depend on other models' state
after_save :update_order_total             # ❌ Fragile, hard to test

# ❌ DON'T: Skip callbacks with update_column/update_all
# If you need to skip callbacks, your design may need rethinking

# ✅ DO: Use after_commit (not after_save) for jobs and external calls
# after_save runs inside the transaction; if it fails, the transaction rolls back
after_commit :process_payment_async, on: :create  # ✅ Runs after transaction commits
```

### Conditional Callbacks

```ruby
before_save :set_published_at, if: :publishing?
after_create :notify_admin, unless: :test_account?

private

def publishing?
  status_changed? && status == "published"
end
```

---

## 3.5 Scope Conventions

```ruby
# Convention: Use scopes for reusable query logic
# Scopes should always return an ActiveRecord::Relation

# ✅ Simple scopes (lambda syntax)
scope :active, -> { where(active: true) }
scope :recent, -> { order(created_at: :desc) }
scope :published, -> { where(status: "published") }

# ✅ Scopes with arguments
scope :created_before, ->(date) { where("created_at < ?", date) }
scope :by_status, ->(status) { where(status: status) }
scope :search, ->(term) { where("name ILIKE ?", "%#{sanitize_sql_like(term)}%") }

# ✅ Scopes that chain
scope :active_and_recent, -> { active.recent }

# ✅ Default scope (use sparingly — it affects ALL queries)
default_scope { where(deleted_at: nil) }  # Soft delete — use with caution

# ❌ DON'T use scopes that return single records (use class methods instead)
# BAD:
scope :latest, -> { order(created_at: :desc).first }  # Returns object, not relation

# ✅ GOOD:
def self.latest
  order(created_at: :desc).first
end
```

---

## 3.6 Enum Conventions

```ruby
# Rails 7+ syntax (keyword arguments)
class Order < ApplicationRecord
  enum :status, {
    pending: 0,
    confirmed: 1,
    processing: 2,
    shipped: 3,
    delivered: 4,
    cancelled: 5,
    refunded: 6
  }

  # With prefix/suffix to avoid method name collisions
  enum :payment_status, {
    pending: 0,
    paid: 1,
    failed: 2
  }, prefix: true  # payment_status_pending?, payment_status_paid?

  enum :delivery_method, {
    standard: 0,
    express: 1,
    overnight: 2
  }, suffix: :delivery  # standard_delivery?, express_delivery?
end

# Generated methods:
order.pending?          # Check status
order.confirmed!        # Update status (with save!)
Order.pending           # Scope: Order.where(status: :pending)
order.status            # Returns string: "pending"

# ✅ ALWAYS use integers for enum values (not strings)
# ✅ ALWAYS explicitly assign integer values (don't rely on array position)
# ✅ Use prefix/suffix when multiple enums might have same values
# ❌ NEVER remove or reorder existing enum values in production
# ✅ Only append new values at the end with new integer values
```

---

## 3.7 Query Conventions

```ruby
# ✅ Use Active Record query interface
User.where(active: true)
User.where("created_at > ?", 1.week.ago)
User.where(role: [:admin, :moderator])
User.where.not(status: :banned)

# ✅ Use find_by for single records (returns nil if not found)
User.find_by(email: "user@example.com")

# ✅ Use find for lookup by ID (raises RecordNotFound)
User.find(params[:id])

# ✅ Use exists? instead of present? for checking existence
User.where(email: email).exists?  # ✅ SELECT 1 ... LIMIT 1
User.where(email: email).present? # ❌ Loads all records into memory

# ✅ Use pluck for extracting single columns
User.active.pluck(:email)         # Returns array of strings
User.active.pluck(:id, :email)    # Returns array of arrays

# ✅ Use select for limiting loaded columns
User.select(:id, :name, :email).where(active: true)

# ✅ Use find_each / find_in_batches for large datasets
User.find_each(batch_size: 1000) do |user|
  user.process_something
end

# ❌ NEVER interpolate user input into SQL strings
User.where("name = '#{params[:name]}'")       # ❌ SQL INJECTION!
User.where("name = ?", params[:name])          # ✅ Parameterized
User.where(name: params[:name])                # ✅ Hash conditions

# ✅ Use Arel for complex queries
users = User.arel_table
User.where(users[:age].gt(18).and(users[:status].eq("active")))
```

---

## 3.8 ActiveRecord Callbacks vs Service Objects

| Use Callbacks For | Use Service Objects For |
|---|---|
| Data normalization (downcase email) | Complex business logic |
| Setting defaults | Multi-model operations |
| Generating slugs/tokens | External API calls |
| Updating counter caches | Payment processing |
| Simple derived attributes | Sending emails (debatable) |

---

## 3.9 Model Concerns

```ruby
# app/models/concerns/searchable.rb
module Searchable
  extend ActiveSupport::Concern

  included do
    scope :search, ->(query) {
      where("name ILIKE :q OR description ILIKE :q", q: "%#{sanitize_sql_like(query)}%")
    }
  end

  class_methods do
    def searchable_columns
      %i[name description]
    end
  end

  def matching_search_term?(term)
    searchable_columns.any? { |col| send(col)&.downcase&.include?(term.downcase) }
  end
end

# Usage
class Product < ApplicationRecord
  include Searchable
end
```

### Common Concern Patterns

```ruby
# Soft Delete
module SoftDeletable
  extend ActiveSupport::Concern

  included do
    scope :kept, -> { where(deleted_at: nil) }
    scope :deleted, -> { where.not(deleted_at: nil) }
  end

  def soft_delete
    update(deleted_at: Time.current)
  end

  def restore
    update(deleted_at: nil)
  end

  def deleted?
    deleted_at.present?
  end
end

# Sluggable
module Sluggable
  extend ActiveSupport::Concern

  included do
    before_validation :generate_slug, on: :create
    validates :slug, presence: true, uniqueness: true
  end

  private

  def generate_slug
    self.slug = name&.parameterize
  end
end

# Tokenable
module Tokenable
  extend ActiveSupport::Concern

  included do
    before_create :generate_token
  end

  private

  def generate_token
    self.token = SecureRandom.urlsafe_base64(32)
  end
end
```

---

## 3.10 Transactions

```ruby
# ✅ Use transactions for multi-record operations that must succeed together
ActiveRecord::Base.transaction do
  order = Order.create!(order_params)
  order.line_items.create!(line_item_params)
  order.process_payment!
end

# ✅ Use with_lock for optimistic/pessimistic locking
account.with_lock do
  account.balance -= amount
  account.save!
end

# Convention: Use bang methods (save!, create!, update!) inside transactions
# They raise exceptions, which trigger automatic rollback
```

---

## 3.11 Secure Token & has_secure_password

```ruby
class User < ApplicationRecord
  # Built-in password hashing (requires bcrypt gem and password_digest column)
  has_secure_password

  # Auto-generated unique tokens
  has_secure_token :api_key
  has_secure_token :confirmation_token

  # Generates:
  # user.api_key                    # Access the token
  # user.regenerate_api_key         # Generate a new one
  # User.generate_unique_secure_token  # Class method
end
```

---

## 3.12 Rich Text & Action Text

```ruby
class Article < ApplicationRecord
  has_rich_text :body
  has_rich_text :excerpt
end

# Convention: Rich text fields are stored in action_text_rich_texts table
# Access: article.body.to_s, article.body.to_plain_text
# Eager load: Article.all.with_rich_text_body
```

---

## 3.13 ApplicationRecord

```ruby
# app/models/application_record.rb
# All models inherit from this (not ActiveRecord::Base directly)
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class

  # Shared model logic goes here
  def self.ransackable_attributes(auth_object = nil)
    authorizable_ransackable_attributes
  end
end
```

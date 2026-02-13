# 17 — Code Style & Ruby Conventions

---

## 17.1 Formatting Rules

```ruby
# Indentation: 2 spaces (NEVER tabs)
class User
  def full_name
    "#{first_name} #{last_name}"
  end
end

# Line length: Max 120 characters (soft limit)
# Method length: Max 10-15 lines (guideline)
# Class length: Max 100-150 lines (guideline, extract if longer)

# Trailing commas in multi-line hashes/arrays (easier diffs)
{
  name: "John",
  email: "john@example.com",
  role: "admin",        # ← trailing comma
}

# String quoting: Double quotes for interpolation, single or double otherwise
name = "John"
greeting = "Hello, #{name}!"
status = "active"       # Both are acceptable
status = 'active'       # Both are acceptable

# Frozen string literal (opt-in, recommended)
# frozen_string_literal: true
```

---

## 17.2 Naming Style

```ruby
# Methods and variables: snake_case
def calculate_total; end
total_amount = 0

# Classes and modules: PascalCase
class UserAccount; end
module Admin; end

# Constants: SCREAMING_SNAKE_CASE
MAX_RETRIES = 3
DEFAULT_TIMEOUT = 30

# Predicate methods: end with ?
def active?; end
def valid_email?; end

# Dangerous methods (modify self or raise): end with !
def save!; end
def normalize!; end

# Setters: end with =
def name=(value); end

# Boolean variables: use adjective or is_/has_ prefix
active = true
is_valid = true
has_children = false

# Avoid: Hungarian notation, type prefixes
str_name = "bad"     # ❌
arr_items = []       # ❌
```

---

## 17.3 Method Design

```ruby
# ✅ Small, focused methods (Single Responsibility)
def process_order(order)
  validate_order(order)
  calculate_total(order)
  charge_payment(order)
  send_confirmation(order)
end

# ✅ Use keyword arguments for clarity
def create_user(name:, email:, role: "member")
  User.create!(name: name, email: email, role: role)
end

# ✅ Prefer early returns (guard clauses)
def process(order)
  return unless order.valid?
  return if order.processed?
  # ... main logic
end

# ❌ AVOID deeply nested conditionals
def process(order)
  if order.valid?
    if !order.processed?
      if order.items.any?
        # ... deeply nested
      end
    end
  end
end

# ✅ Use &. (safe navigation) instead of try
user&.profile&.avatar_url    # ✅
user.try(:profile).try(:avatar_url)  # ❌ (use &. instead)

# ✅ Use fetch with default for hash access
options.fetch(:timeout, 30)
params.fetch(:page, 1)
```

---

## 17.4 Ruby Idioms

```ruby
# String interpolation (not concatenation)
"Hello, #{name}!"         # ✅
"Hello, " + name + "!"    # ❌

# Symbols for hash keys
{ name: "John", age: 30 }     # ✅ (Ruby 3.1+ shorthand: { name:, age: })
{ "name" => "John" }          # ❌ (unless keys are dynamic strings)

# Array/Hash literals
%w[admin moderator member]   # ✅ Array of strings
%i[name email role]          # ✅ Array of symbols
[1, 2, 3].freeze             # ✅ Frozen constant array

# Conditional assignment
@users ||= User.all          # Memoization
name ||= "Default"           # Default value

# Ternary for simple conditions
status = active? ? "Active" : "Inactive"    # ✅
# Use if/else for complex conditions        # ✅

# Unless (for negative conditions — single clause only)
return unless valid?          # ✅
unless valid?                 # ✅ (simple case)
  raise InvalidError
end
# ❌ NEVER use unless with else
unless valid?
  # ...
else    # ❌ Confusing! Use if instead
  # ...
end

# each vs map
users.each { |u| send_email(u) }        # Side effects
names = users.map { |u| u.name }        # Transformation
names = users.map(&:name)               # Shorthand

# Block style
# Single line: { }
users.map { |u| u.name }
# Multi line: do...end
users.each do |user|
  process(user)
  notify(user)
end

# Enumerable methods (prefer over manual loops)
users.select(&:active?)
users.reject(&:admin?)
users.find { |u| u.email == email }
users.any?(&:admin?)
users.all?(&:verified?)
users.none?(&:banned?)
users.count(&:active?)
users.sum(&:balance)
users.group_by(&:role)
users.sort_by(&:name)
users.flat_map(&:posts)
users.each_with_object({}) { |u, h| h[u.id] = u.name }
```

---

## 17.5 Error Handling

```ruby
# ✅ Rescue specific exceptions
begin
  process_payment(order)
rescue Stripe::CardError => e
  order.update!(status: :payment_failed, error_message: e.message)
rescue Stripe::RateLimitError => e
  retry_later(order)
rescue StandardError => e
  Rails.logger.error("Unexpected error: #{e.message}")
  raise  # Re-raise unexpected errors
end

# ❌ NEVER rescue Exception (catches system exits, interrupts)
rescue Exception => e   # ❌

# ❌ AVOID bare rescue (catches StandardError — too broad)
rescue => e             # ⚠️ Only when you truly want all StandardErrors

# ✅ Custom exception classes
module Payments
  class Error < StandardError; end
  class InsufficientFundsError < Error; end
  class InvalidCardError < Error; end
  class GatewayTimeoutError < Error; end
end

# ✅ Use ensure for cleanup
def process_file(path)
  file = File.open(path)
  parse(file)
rescue ParseError => e
  Rails.logger.error("Parse failed: #{e.message}")
ensure
  file&.close
end
```

---

## 17.6 RuboCop Configuration

```yaml
# .rubocop.yml
require:
  - rubocop-rails
  - rubocop-rspec
  - rubocop-performance

AllCops:
  TargetRubyVersion: 3.3
  NewCops: enable
  Exclude:
    - "db/schema.rb"
    - "db/migrate/*"
    - "bin/*"
    - "vendor/**/*"
    - "node_modules/**/*"

# Customize rules
Layout/LineLength:
  Max: 120

Metrics/MethodLength:
  Max: 15

Metrics/ClassLength:
  Max: 150

Style/Documentation:
  Enabled: false

Style/FrozenStringLiteralComment:
  EnforcedStyle: always

Rails/HasAndBelongsToMany:
  Enabled: true  # Encourage has_many :through

Rails/SkipsModelValidations:
  Enabled: true  # Warn on update_column, etc.
```

---

## 17.7 Git Conventions

```bash
# Commit message format
# <type>: <subject>
#
# Types: feat, fix, refactor, docs, test, chore, style, perf

feat: Add user registration with email confirmation
fix: Resolve N+1 query in posts index
refactor: Extract payment processing to service object
docs: Update API documentation for v2
test: Add system tests for checkout flow
chore: Update Rails to 7.1.3
perf: Add database indexes for user search

# Branch naming
feature/user-registration
bugfix/payment-timeout
hotfix/security-patch
chore/update-dependencies

# Convention: Small, focused commits
# Convention: One logical change per commit
# Convention: Write commit messages in imperative mood
```

---

## 17.8 Comments Convention

```ruby
# ✅ Comments explain WHY, not WHAT
# Discount expires after 30 days per business requirement #1234
discount_expiry = 30.days.from_now

# ✅ TODO/FIXME/HACK with context
# TODO(vignesh): Replace with proper rate limiter after v2 launch
# FIXME: Race condition when two users update simultaneously
# HACK: Workaround for Stripe API bug #5678

# ❌ DON'T state the obvious
# Get the user
user = User.find(id)  # ❌ Comment adds nothing

# ✅ Use YARD-style for public API documentation
# Calculates the total price including tax and shipping.
#
# @param order [Order] the order to calculate
# @param include_tax [Boolean] whether to include tax (default: true)
# @return [BigDecimal] the total amount
# @raise [InvalidOrderError] if order has no line items
def calculate_total(order, include_tax: true)
end
```

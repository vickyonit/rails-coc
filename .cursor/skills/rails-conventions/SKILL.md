---
name: rails-conventions
description: Follow Rails conventions and best practices when writing Ruby on Rails code. Use when working on Rails projects, creating models, controllers, views, migrations, routes, or any Rails code. Ensures code follows Rails naming conventions, project structure, RESTful patterns, and Rails 7.1+ best practices.
---

# Rails Conventions Guide

Follow Rails conventions and best practices when writing Ruby on Rails code. This skill ensures your code adheres to Rails naming conventions, project structure, RESTful patterns, and modern Rails practices.

## Quick Reference

### Naming Conventions

| Context | Convention | Example |
|---------|-----------|---------|
| Classes & Modules | `PascalCase` | `UserAccount`, `Admin::Dashboard` |
| Methods & Variables | `snake_case` | `first_name`, `calculate_total` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Predicate methods | End with `?` | `active?`, `admin?` |
| Dangerous methods | End with `!` | `save!`, `destroy!` |
| Database tables | `snake_case`, plural | `users`, `line_items` |
| Database columns | `snake_case`, singular | `first_name`, `user_id` |
| Foreign keys | `singular_table_id` | `user_id`, `category_id` |

### Model File Structure Order

```ruby
class User < ApplicationRecord
  # 1. Constants
  ROLES = %w[admin moderator member].freeze
  
  # 2. Attribute declarations (enums)
  enum :status, { active: 0, inactive: 1 }
  
  # 3. Associations (belongs_to → has_one → has_many → has_many :through)
  belongs_to :organization
  has_one :profile, dependent: :destroy
  has_many :posts, dependent: :destroy
  
  # 4. Delegations
  delegate :name, to: :organization, prefix: true
  
  # 5. Validations
  validates :email, presence: true, uniqueness: true
  
  # 6. Callbacks (lifecycle order)
  before_validation :normalize_email
  after_create :send_welcome_email
  
  # 7. Scopes
  scope :active, -> { where(status: :active) }
  
  # 8. Class methods
  def self.find_by_email(email)
    find_by(email: email.downcase)
  end
  
  # 9. Instance methods
  def full_name
    "#{first_name} #{last_name}"
  end
  
  # 10. Private methods
  private
  
  def normalize_email
    self.email = email.downcase.strip
  end
end
```

### Controller Conventions

- Use RESTful actions: `index`, `show`, `new`, `create`, `edit`, `update`, `destroy`
- Keep controllers thin - business logic belongs in models or service objects
- Use strong parameters for mass assignment
- Follow the 7 RESTful routes pattern

```ruby
class UsersController < ApplicationController
  before_action :set_user, only: [:show, :edit, :update, :destroy]
  
  def index
    @users = User.all
  end
  
  def show
  end
  
  def new
    @user = User.new
  end
  
  def create
    @user = User.new(user_params)
    if @user.save
      redirect_to @user, notice: 'User was successfully created.'
    else
      render :new, status: :unprocessable_entity
    end
  end
  
  private
  
  def set_user
    @user = User.find(params[:id])
  end
  
  def user_params
    params.require(:user).permit(:name, :email, :role)
  end
end
```

### Routing Conventions

- Use `resources` for RESTful routes
- Nest resources only when necessary (max 1 level deep)
- Use `member` and `collection` routes sparingly

```ruby
# Good
resources :users
resources :posts do
  resources :comments, only: [:create, :destroy]
end

# Avoid deep nesting
resources :users do
  resources :posts do
    resources :comments do
      resources :replies  # Too deep!
    end
  end
end
```

### Migration Conventions

- Use reversible migrations
- Add indexes for foreign keys and frequently queried columns
- Use `change` method when possible
- Add `null: false` and `default` values appropriately

```ruby
class CreateUsers < ActiveRecord::Migration[7.1]
  def change
    create_table :users do |t|
      t.string :email, null: false, index: { unique: true }
      t.string :name, null: false
      t.timestamps
    end
  end
end
```

## Golden Rules

1. **Convention over Configuration** - Follow Rails defaults unless you have a strong reason not to
2. **DRY (Don't Repeat Yourself)** - Extract shared logic into concerns, helpers, or service objects
3. **Fat Models, Skinny Controllers** - Business logic belongs in models, not controllers
4. **RESTful Design** - Think in resources, not arbitrary actions
5. **Principle of Least Surprise** - Code should do what a reader expects
6. **Fail Fast, Fail Loud** - Use bang methods and validations to catch errors early
7. **Test Everything That Could Break** - Especially edge cases and business logic

## When Writing Rails Code

### Models
- ✅ Use proper naming (PascalCase class, snake_case file)
- ✅ Follow the model file structure order above
- ✅ Use `dependent: :destroy` or `dependent: :nullify` appropriately
- ✅ Add validations for data integrity
- ✅ Use scopes for common queries
- ✅ Prefer `has_many :through` over `has_and_belongs_to_many`

### Controllers
- ✅ Keep actions focused and thin
- ✅ Use strong parameters
- ✅ Use `before_action` filters appropriately
- ✅ Return appropriate HTTP status codes
- ✅ Use `redirect_to` after successful creates/updates
- ✅ Use `render` with status for validation errors

### Views
- ✅ Use partials for repeated markup
- ✅ Keep logic in helpers or presenters
- ✅ Use semantic HTML
- ✅ Follow Rails view conventions (ERB/Haml)

### Routes
- ✅ Use `resources` for RESTful routes
- ✅ Avoid deep nesting (max 1 level)
- ✅ Use `only:` and `except:` to limit routes
- ✅ Use namespaces for admin/API sections

### Migrations
- ✅ Make migrations reversible
- ✅ Add indexes for foreign keys
- ✅ Use `change` method when possible
- ✅ Add `null: false` constraints where appropriate

## Common Patterns

### Service Objects
For complex business logic that doesn't fit in models:

```ruby
class UserRegistrationService
  def initialize(user_params)
    @user_params = user_params
  end
  
  def call
    user = User.create(@user_params)
    send_welcome_email(user) if user.persisted?
    user
  end
  
  private
  
  def send_welcome_email(user)
    UserMailer.welcome(user).deliver_later
  end
end
```

### Query Objects
For complex queries:

```ruby
class ActiveUsersQuery
  def initialize(scope = User.all)
    @scope = scope
  end
  
  def call
    @scope.where(status: :active)
         .where('last_login_at > ?', 30.days.ago)
         .order(:name)
  end
end
```

## Additional Resources

For complete Rails conventions documentation, see the Rails CoC files in this repository:

- [01-naming-conventions.md](../01-naming-conventions.md) - Complete naming guide
- [02-project-structure.md](../02-project-structure.md) - Directory layout and organization
- [03-models-activerecord.md](../03-models-activerecord.md) - Model conventions and patterns
- [04-controllers.md](../04-controllers.md) - Controller best practices
- [05-views-templates.md](../05-views-templates.md) - View conventions
- [06-routing.md](../06-routing.md) - Routing patterns
- [07-database-migrations.md](../07-database-migrations.md) - Migration conventions
- [08-testing-conventions.md](../08-testing-conventions.md) - Testing best practices
- [09-configuration-environments.md](../09-configuration-environments.md) - Configuration patterns
- [10-security.md](../10-security.md) - Security conventions
- [11-api-conventions.md](../11-api-conventions.md) - API development
- [12-background-jobs.md](../12-background-jobs.md) - Active Job patterns
- [13-action-mailer.md](../13-action-mailer.md) - Mailer conventions
- [14-action-cable.md](../14-action-cable.md) - WebSocket patterns
- [15-active-storage.md](../15-active-storage.md) - File handling
- [16-assets-frontend.md](../16-assets-frontend.md) - Frontend asset management
- [17-code-style.md](../17-code-style.md) - Ruby code style
- [18-advanced-patterns.md](../18-advanced-patterns.md) - Advanced Rails patterns
- [19-performance.md](../19-performance.md) - Performance optimization
- [20-deployment-devops.md](../20-deployment-devops.md) - Deployment practices

## Rails Version

This guide targets **Rails 7.1+** with notes for **Rails 8.0** where applicable.

# 18 — Advanced Patterns

---

## 18.1 Service Objects

The most important pattern for keeping controllers and models clean.

```ruby
# Convention: app/services/<namespace>/<verb>.rb
# Single public method: .call or .perform

# app/services/users/create.rb
module Users
  class Create
    def initialize(params, current_user: nil)
      @params = params
      @current_user = current_user
    end

    def call
      user = User.new(@params)

      ActiveRecord::Base.transaction do
        user.save!
        create_profile!(user)
        send_welcome_email(user)
        track_signup(user)
      end

      Result.new(success: true, user: user)
    rescue ActiveRecord::RecordInvalid => e
      Result.new(success: false, user: e.record, errors: e.record.errors)
    end

    private

    def create_profile!(user)
      user.create_profile!(bio: "", visibility: "public")
    end

    def send_welcome_email(user)
      UserMailer.welcome_email(user).deliver_later
    end

    def track_signup(user)
      Analytics.track("user_signed_up", user_id: user.id)
    end
  end
end

# Result object (simple struct)
Result = Struct.new(:success, :user, :errors, keyword_init: true) do
  def success? = success
  def failure? = !success
end

# Usage in controller:
class UsersController < ApplicationController
  def create
    result = Users::Create.new(user_params).call

    if result.success?
      redirect_to result.user, notice: "Welcome!"
    else
      @user = result.user
      render :new, status: :unprocessable_entity
    end
  end
end
```

### Alternative: Callable Pattern

```ruby
# app/services/base_service.rb
class BaseService
  def self.call(...)
    new(...).call
  end
end

# app/services/process_payment.rb
class ProcessPayment < BaseService
  def initialize(order)
    @order = order
  end

  def call
    # ...
  end
end

# Usage:
ProcessPayment.call(order)
```

---

## 18.2 Form Objects

For forms that span multiple models or need complex validation.

```ruby
# app/forms/registration_form.rb
class RegistrationForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :name, :string
  attribute :email, :string
  attribute :password, :string
  attribute :company_name, :string
  attribute :terms_accepted, :boolean

  validates :name, presence: true
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 }
  validates :company_name, presence: true
  validates :terms_accepted, acceptance: true

  def save
    return false unless valid?

    ActiveRecord::Base.transaction do
      organization = Organization.create!(name: company_name)
      user = organization.users.create!(
        name: name,
        email: email,
        password: password,
        role: :admin
      )
    end

    true
  rescue ActiveRecord::RecordInvalid => e
    errors.merge!(e.record.errors)
    false
  end
end

# Usage in controller:
class RegistrationsController < ApplicationController
  def new
    @form = RegistrationForm.new
  end

  def create
    @form = RegistrationForm.new(registration_params)
    if @form.save
      redirect_to dashboard_path, notice: "Welcome!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def registration_params
    params.require(:registration_form).permit(:name, :email, :password, :company_name, :terms_accepted)
  end
end
```

---

## 18.3 Query Objects

For complex database queries that don't belong in scopes.

```ruby
# app/queries/users/search_query.rb
module Users
  class SearchQuery
    def initialize(relation = User.all)
      @relation = relation
    end

    def call(params)
      result = @relation
      result = filter_by_name(result, params[:name]) if params[:name].present?
      result = filter_by_role(result, params[:role]) if params[:role].present?
      result = filter_by_status(result, params[:status]) if params[:status].present?
      result = filter_by_date_range(result, params[:from], params[:to])
      result = sort_by(result, params[:sort], params[:direction])
      result
    end

    private

    def filter_by_name(relation, name)
      relation.where("name ILIKE ?", "%#{ActiveRecord::Base.sanitize_sql_like(name)}%")
    end

    def filter_by_role(relation, role)
      relation.where(role: role)
    end

    def filter_by_status(relation, status)
      relation.where(status: status)
    end

    def filter_by_date_range(relation, from, to)
      relation = relation.where("created_at >= ?", from) if from.present?
      relation = relation.where("created_at <= ?", to) if to.present?
      relation
    end

    def sort_by(relation, column, direction)
      allowed = %w[name email created_at]
      return relation.order(created_at: :desc) unless allowed.include?(column)

      relation.order(column => (direction == "asc" ? :asc : :desc))
    end
  end
end

# Usage:
users = Users::SearchQuery.new.call(params.permit(:name, :role, :status, :from, :to, :sort, :direction))
```

---

## 18.4 Presenter / Decorator Pattern

```ruby
# app/presenters/user_presenter.rb
class UserPresenter < SimpleDelegator
  def display_name
    name.presence || email.split("@").first
  end

  def avatar_url
    if avatar.attached?
      Rails.application.routes.url_helpers.url_for(avatar.variant(:thumb))
    else
      "https://ui-avatars.com/api/?name=#{CGI.escape(display_name)}"
    end
  end

  def member_since
    created_at.strftime("%B %Y")
  end

  def role_badge_class
    case role
    when "admin" then "badge-red"
    when "moderator" then "badge-yellow"
    else "badge-gray"
    end
  end
end

# Usage in controller:
def show
  @user = UserPresenter.new(User.find(params[:id]))
end

# In view:
<%= @user.display_name %>
<span class="<%= @user.role_badge_class %>"><%= @user.role %></span>
```

---

## 18.5 Value Objects

```ruby
# app/value_objects/money.rb
class Money
  include Comparable

  attr_reader :amount, :currency

  def initialize(amount, currency = "USD")
    @amount = BigDecimal(amount.to_s)
    @currency = currency
    freeze
  end

  def +(other)
    raise "Currency mismatch" unless currency == other.currency
    self.class.new(amount + other.amount, currency)
  end

  def -(other)
    raise "Currency mismatch" unless currency == other.currency
    self.class.new(amount - other.amount, currency)
  end

  def *(multiplier)
    self.class.new(amount * multiplier, currency)
  end

  def <=>(other)
    amount <=> other.amount
  end

  def to_s
    "$#{'%.2f' % amount}"
  end

  def zero?
    amount.zero?
  end
end
```

---

## 18.6 Policy Objects (Beyond Pundit)

```ruby
# app/policies/order_cancellation_policy.rb
class OrderCancellationPolicy
  def initialize(order, user)
    @order = order
    @user = user
  end

  def allowed?
    return true if @user.admin?
    return false if @order.shipped?
    return false if @order.created_at < 24.hours.ago

    @order.user == @user
  end

  def reason_not_allowed
    return "Order has been shipped" if @order.shipped?
    return "Cancellation window has passed" if @order.created_at < 24.hours.ago
    return "Not your order" unless @order.user == @user
  end
end
```

---

## 18.7 Observer / Event Pattern

```ruby
# Using ActiveSupport::Notifications
# Publishing events
ActiveSupport::Notifications.instrument("order.completed", order: order) do
  order.update!(status: :completed)
end

# Subscribing to events
# config/initializers/event_subscribers.rb
ActiveSupport::Notifications.subscribe("order.completed") do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  order = event.payload[:order]
  OrderMailer.confirmation(order).deliver_later
  InventoryService.decrement(order.line_items)
  Analytics.track("order_completed", order_id: order.id)
end
```

---

## 18.8 Interactor Pattern

```ruby
# For complex multi-step business operations
# app/interactors/place_order.rb
class PlaceOrder
  def self.call(user:, cart:, payment_method:)
    new(user: user, cart: cart, payment_method: payment_method).call
  end

  def initialize(user:, cart:, payment_method:)
    @user = user
    @cart = cart
    @payment_method = payment_method
  end

  def call
    ActiveRecord::Base.transaction do
      order = create_order
      charge_payment(order)
      update_inventory(order)
      send_notifications(order)
      OpenStruct.new(success: true, order: order)
    end
  rescue PaymentError => e
    OpenStruct.new(success: false, error: e.message)
  end

  private

  def create_order
    Orders::Create.call(user: @user, cart: @cart)
  end

  def charge_payment(order)
    Payments::Charge.call(order: order, method: @payment_method)
  end

  def update_inventory(order)
    order.line_items.each { |li| Inventory::Decrement.call(product: li.product, quantity: li.quantity) }
  end

  def send_notifications(order)
    OrderMailer.confirmation(order).deliver_later
    AdminNotifier.new_order(order).deliver_later
  end
end
```

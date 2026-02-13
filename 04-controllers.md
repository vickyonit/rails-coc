# 04 — Controller Conventions

---

## 4.1 Controller Structure Convention

```ruby
class UsersController < ApplicationController
  # 1. Callbacks / Filters (in order of execution)
  before_action :authenticate_user!
  before_action :set_user, only: [:show, :edit, :update, :destroy]
  before_action :authorize_user!, only: [:edit, :update, :destroy]
  after_action :track_activity, only: [:create, :update]
  around_action :wrap_in_transaction, only: [:create, :update]

  # 2. RESTful actions (in conventional order)
  # GET /users
  def index
    @users = User.active.order(:name).page(params[:page])
  end

  # GET /users/:id
  def show
  end

  # GET /users/new
  def new
    @user = User.new
  end

  # GET /users/:id/edit
  def edit
  end

  # POST /users
  def create
    @user = User.new(user_params)
    if @user.save
      redirect_to @user, notice: "User was successfully created."
    else
      render :new, status: :unprocessable_entity
    end
  end

  # PATCH/PUT /users/:id
  def update
    if @user.update(user_params)
      redirect_to @user, notice: "User was successfully updated."
    else
      render :edit, status: :unprocessable_entity
    end
  end

  # DELETE /users/:id
  def destroy
    @user.destroy!
    redirect_to users_url, notice: "User was successfully destroyed."
  end

  # 3. Private methods
  private

  def set_user
    @user = User.find(params[:id])
  end

  def user_params
    params.require(:user).permit(:name, :email, :role)
  end

  def authorize_user!
    redirect_to root_path, alert: "Not authorized" unless current_user.admin?
  end
end
```

---

## 4.2 The Seven RESTful Actions

| Action | HTTP Verb | Path | Purpose |
|--------|-----------|------|---------|
| `index` | GET | /users | List all resources |
| `show` | GET | /users/:id | Show a single resource |
| `new` | GET | /users/new | Form for creating |
| `create` | POST | /users | Process creation |
| `edit` | GET | /users/:id/edit | Form for editing |
| `update` | PATCH/PUT | /users/:id | Process update |
| `destroy` | DELETE | /users/:id | Delete resource |

**Convention:** Stick to these seven actions. If you need more, consider creating a new controller for the sub-resource.

```ruby
# ❌ BAD: Adding custom actions to existing controllers
class UsersController < ApplicationController
  def activate    # Custom action
  def deactivate  # Custom action
  def export      # Custom action
end

# ✅ GOOD: Extract to separate controllers
class Users::ActivationsController < ApplicationController
  def create    # Activate user
  def destroy   # Deactivate user
end

class Users::ExportsController < ApplicationController
  def show      # Export user data
end
```

---

## 4.3 Strong Parameters

```ruby
# Convention: Define a private method named <model>_params
private

def user_params
  params.require(:user).permit(
    :name, :email, :password, :password_confirmation,
    :avatar, :bio,
    address_attributes: [:street, :city, :state, :zip],
    tag_ids: [],
    preferences: {}
  )
end

# Nested attributes
def order_params
  params.require(:order).permit(
    :customer_name, :shipping_method,
    line_items_attributes: [:id, :product_id, :quantity, :_destroy]
  )
end

# Conditional params based on role
def user_params
  permitted = [:name, :email, :bio]
  permitted += [:role, :admin] if current_user.admin?
  params.require(:user).permit(permitted)
end
```

---

## 4.4 Before Actions (Filters)

```ruby
class ApplicationController < ActionController::Base
  # Authentication — applied to ALL controllers
  before_action :authenticate_user!

  # Skip for specific controllers/actions
  skip_before_action :authenticate_user!, only: [:index, :show]

  private

  def authenticate_user!
    redirect_to login_path, alert: "Please log in" unless current_user
  end

  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
  helper_method :current_user  # Make available in views
end

class Admin::BaseController < ApplicationController
  before_action :require_admin!

  private

  def require_admin!
    redirect_to root_path, alert: "Access denied" unless current_user&.admin?
  end
end
```

---

## 4.5 Rendering Conventions

```ruby
# Convention: Rails automatically renders the template matching the action
def show
  @user = User.find(params[:id])
  # Automatically renders app/views/users/show.html.erb
end

# Explicit rendering
def create
  @user = User.new(user_params)
  if @user.save
    redirect_to @user, notice: "Created!"           # Redirect on success
  else
    render :new, status: :unprocessable_entity       # Re-render form on failure
  end
end

# ✅ Always set status codes on error renders
render :new, status: :unprocessable_entity           # 422
render :edit, status: :unprocessable_entity           # 422

# JSON responses
render json: @user
render json: @user, status: :created
render json: { error: "Not found" }, status: :not_found

# Turbo Stream responses (Rails 7+)
respond_to do |format|
  format.html { redirect_to @user }
  format.turbo_stream
  format.json { render json: @user }
end
```

---

## 4.6 Flash Messages

```ruby
# Convention: Use :notice for success and :alert for errors
redirect_to users_path, notice: "User created successfully."
redirect_to users_path, alert: "Something went wrong."

# In the action (for render, not redirect)
flash.now[:notice] = "Form has errors."

# Custom flash types
add_flash_types :success, :warning, :info

# Then:
redirect_to @user, success: "Profile updated!"
```

---

## 4.7 Respond To / Format

```ruby
def index
  @users = User.all

  respond_to do |format|
    format.html                                # Renders index.html.erb
    format.json { render json: @users }        # Renders JSON
    format.csv { send_data @users.to_csv }     # Downloads CSV
    format.turbo_stream                        # Renders index.turbo_stream.erb
    format.pdf do
      pdf = UserPdf.new(@users)
      send_data pdf.render, filename: "users.pdf", type: "application/pdf"
    end
  end
end
```

---

## 4.8 Error Handling

```ruby
class ApplicationController < ActionController::Base
  # Rescue from common errors
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActiveRecord::RecordInvalid, with: :unprocessable_entity
  rescue_from ActionController::ParameterMissing, with: :bad_request
  rescue_from Pundit::NotAuthorizedError, with: :forbidden

  private

  def not_found
    respond_to do |format|
      format.html { render "errors/not_found", status: :not_found }
      format.json { render json: { error: "Not found" }, status: :not_found }
    end
  end

  def unprocessable_entity(exception)
    respond_to do |format|
      format.html { render "errors/unprocessable", status: :unprocessable_entity }
      format.json { render json: { errors: exception.record.errors }, status: :unprocessable_entity }
    end
  end

  def forbidden
    respond_to do |format|
      format.html { redirect_to root_path, alert: "Access denied" }
      format.json { render json: { error: "Forbidden" }, status: :forbidden }
    end
  end

  def bad_request(exception)
    render json: { error: exception.message }, status: :bad_request
  end
end
```

---

## 4.9 Controller Concerns

```ruby
# app/controllers/concerns/paginatable.rb
module Paginatable
  extend ActiveSupport::Concern

  private

  def page
    (params[:page] || 1).to_i
  end

  def per_page
    [(params[:per_page] || 25).to_i, 100].min  # Max 100
  end
end

# app/controllers/concerns/error_handling.rb
module ErrorHandling
  extend ActiveSupport::Concern

  included do
    rescue_from ActiveRecord::RecordNotFound, with: :not_found
    rescue_from ActiveRecord::RecordInvalid, with: :unprocessable_entity
  end

  private

  def not_found
    render json: { error: "Resource not found" }, status: :not_found
  end

  def unprocessable_entity(exception)
    render json: { errors: exception.record.errors.full_messages }, status: :unprocessable_entity
  end
end
```

---

## 4.10 Skinny Controller Rules

```ruby
# ❌ BAD: Fat controller with business logic
class OrdersController < ApplicationController
  def create
    @order = Order.new(order_params)
    @order.total = calculate_total(order_params[:line_items])
    @order.tax = @order.total * 0.08
    @order.shipping = calculate_shipping(@order)

    if @order.save
      StripeService.charge(@order.total + @order.tax + @order.shipping)
      OrderMailer.confirmation(@order).deliver_later
      InventoryService.decrement(@order.line_items)
      redirect_to @order
    else
      render :new
    end
  end
end

# ✅ GOOD: Thin controller, logic in service object
class OrdersController < ApplicationController
  def create
    result = Orders::Create.call(order_params, current_user)

    if result.success?
      redirect_to result.order, notice: "Order placed!"
    else
      @order = result.order
      render :new, status: :unprocessable_entity
    end
  end
end
```

---

## 4.11 Streaming & Downloads

```ruby
# File downloads
def download
  send_file Rails.root.join("storage", "report.pdf"),
            filename: "report.pdf",
            type: "application/pdf",
            disposition: "attachment"
end

# Data streaming
def export
  headers["Content-Type"] = "text/csv"
  headers["Content-Disposition"] = 'attachment; filename="users.csv"'

  response.status = 200
  self.response_body = Enumerator.new do |yielder|
    yielder << CSV.generate_line(["Name", "Email"])
    User.find_each do |user|
      yielder << CSV.generate_line([user.name, user.email])
    end
  end
end
```

---

## 4.12 ApplicationController Base

```ruby
class ApplicationController < ActionController::Base
  # CSRF protection (enabled by default for non-API apps)
  protect_from_forgery with: :exception

  # Common helper methods available in views
  helper_method :current_user, :logged_in?

  # Locale setting
  around_action :switch_locale

  private

  def switch_locale(&action)
    locale = params[:locale] || current_user&.locale || I18n.default_locale
    I18n.with_locale(locale, &action)
  end

  def current_user
    @current_user ||= User.find_by(id: session[:user_id]) if session[:user_id]
  end

  def logged_in?
    current_user.present?
  end

  def require_login
    redirect_to login_path, alert: "Please log in" unless logged_in?
  end
end
```

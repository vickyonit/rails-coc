# 11 — API Conventions

---

## 11.1 API-Only Mode

```ruby
# Generate API-only app
# rails new myapp --api

# Or convert existing app:
# config/application.rb
config.api_only = true

# API controllers inherit from ActionController::API (no views, CSRF, cookies)
class ApplicationController < ActionController::API
end
```

---

## 11.2 API Controller Structure

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ActionController::API
      include ActionController::HttpAuthentication::Token::ControllerMethods

      before_action :authenticate!
      rescue_from ActiveRecord::RecordNotFound, with: :not_found
      rescue_from ActiveRecord::RecordInvalid, with: :unprocessable_entity
      rescue_from ActionController::ParameterMissing, with: :bad_request

      private

      def authenticate!
        authenticate_or_request_with_http_token do |token, _options|
          @current_user = User.find_by(api_token: token)
        end
      end

      def current_user
        @current_user
      end

      def not_found
        render json: { error: "Resource not found" }, status: :not_found
      end

      def unprocessable_entity(exception)
        render json: {
          error: "Validation failed",
          details: exception.record.errors.full_messages
        }, status: :unprocessable_entity
      end

      def bad_request(exception)
        render json: { error: exception.message }, status: :bad_request
      end
    end
  end
end
```

```ruby
# app/controllers/api/v1/users_controller.rb
module Api
  module V1
    class UsersController < BaseController
      def index
        users = User.active.page(params[:page]).per(params[:per_page] || 25)
        render json: users, each_serializer: UserSerializer,
               meta: pagination_meta(users)
      end

      def show
        user = User.find(params[:id])
        render json: user, serializer: UserSerializer
      end

      def create
        user = User.new(user_params)
        user.save!
        render json: user, serializer: UserSerializer, status: :created
      end

      def update
        user = User.find(params[:id])
        user.update!(user_params)
        render json: user, serializer: UserSerializer
      end

      def destroy
        user = User.find(params[:id])
        user.destroy!
        head :no_content
      end

      private

      def user_params
        params.require(:user).permit(:name, :email, :role)
      end

      def pagination_meta(collection)
        {
          current_page: collection.current_page,
          total_pages: collection.total_pages,
          total_count: collection.total_count,
          per_page: collection.limit_value
        }
      end
    end
  end
end
```

---

## 11.3 Response Format Conventions

```ruby
# Convention: Consistent JSON structure

# Success (single resource)
{
  "data": {
    "id": 1,
    "type": "user",
    "attributes": {
      "name": "John",
      "email": "john@example.com"
    }
  }
}

# Success (collection)
{
  "data": [...],
  "meta": {
    "current_page": 1,
    "total_pages": 5,
    "total_count": 100
  }
}

# Error
{
  "error": "Validation failed",
  "details": ["Email can't be blank", "Name is too short"]
}

# HTTP Status Codes Convention:
# 200 OK          — Successful GET, PUT, PATCH
# 201 Created     — Successful POST (resource created)
# 204 No Content  — Successful DELETE
# 400 Bad Request — Invalid parameters
# 401 Unauthorized — Authentication required
# 403 Forbidden   — Authenticated but not authorized
# 404 Not Found   — Resource doesn't exist
# 422 Unprocessable Entity — Validation errors
# 429 Too Many Requests — Rate limited
# 500 Internal Server Error — Server error
```

---

## 11.4 Serializers

```ruby
# Using Alba (recommended for new projects)
# app/serializers/user_serializer.rb
class UserSerializer
  include Alba::Resource

  attributes :id, :name, :email, :role, :created_at

  has_many :posts, serializer: PostSerializer
  has_one :profile, serializer: ProfileSerializer

  attribute :avatar_url do |user|
    user.avatar.attached? ? Rails.application.routes.url_helpers.url_for(user.avatar) : nil
  end
end

# Using Blueprinter
class UserBlueprint < Blueprinter::Base
  identifier :id
  fields :name, :email, :role
  field :created_at, datetime_format: "%Y-%m-%d"

  view :detailed do
    association :posts, blueprint: PostBlueprint
  end
end
# Usage: UserBlueprint.render(user, view: :detailed)

# Using Jbuilder (built-in)
# app/views/api/v1/users/show.json.jbuilder
json.data do
  json.id @user.id
  json.type "user"
  json.attributes do
    json.extract! @user, :name, :email, :role, :created_at
  end
end
```

---

## 11.5 Versioning Conventions

```ruby
# URL versioning (most common)
namespace :api do
  namespace :v1 do
    resources :users
  end
  namespace :v2 do
    resources :users
  end
end

# Header versioning (alternative)
# Accept: application/vnd.myapp.v1+json
class Api::BaseController < ActionController::API
  before_action :set_api_version

  private

  def set_api_version
    @api_version = request.headers["Accept"]&.match(/vnd\.myapp\.v(\d+)/)&.captures&.first || "1"
  end
end

# Convention: Never break existing API versions
# Convention: Deprecate old versions with sunset headers
# Convention: Support at least 2 versions simultaneously
```

---

## 11.6 Pagination

```ruby
# Using Kaminari
users = User.page(params[:page]).per(25)

# Using Pagy (faster)
pagy, users = pagy(User.active, items: 25)

# Convention: Always paginate collection endpoints
# Convention: Use Link headers or meta for pagination info
# Convention: Default per_page = 25, max = 100

# Link header convention:
response.headers["Link"] = [
  "<#{users_url(page: 2)}>; rel=\"next\"",
  "<#{users_url(page: 5)}>; rel=\"last\""
].join(", ")
```

---

## 11.7 CORS Configuration

```ruby
# Gemfile
gem "rack-cors"

# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "https://myapp.com", "https://staging.myapp.com"

    resource "/api/*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      expose: ["Authorization", "Link"],
      max_age: 600
  end
end
```

---

## 11.8 API Authentication Patterns

```ruby
# JWT Authentication
class Api::AuthController < Api::BaseController
  skip_before_action :authenticate!, only: [:login]

  def login
    user = User.find_by(email: params[:email])
    if user&.authenticate(params[:password])
      token = JWT.encode({ user_id: user.id, exp: 24.hours.from_now.to_i }, Rails.application.credentials.secret_key_base)
      render json: { token: token }
    else
      render json: { error: "Invalid credentials" }, status: :unauthorized
    end
  end
end

# API Key Authentication
class Api::BaseController < ActionController::API
  before_action :authenticate_with_api_key!

  private

  def authenticate_with_api_key!
    api_key = request.headers["X-Api-Key"]
    @current_api_client = ApiClient.find_by(key: api_key, active: true)
    render json: { error: "Invalid API key" }, status: :unauthorized unless @current_api_client
  end
end
```

# 10 — Security Conventions

---

## 10.1 CSRF Protection

```ruby
# Enabled by default in ApplicationController
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception  # Raises error on CSRF mismatch

  # Options:
  # :exception — Raise ActionController::InvalidAuthenticityToken
  # :reset_session — Reset the session
  # :null_session — Empty session for the request (best for APIs)
end

# For API controllers:
class Api::BaseController < ActionController::API
  # No CSRF protection needed (uses token auth instead)
end

# Convention: ALWAYS include CSRF meta tags in layouts
<%= csrf_meta_tags %>

# Convention: ALWAYS use form_with (includes CSRF token automatically)
<%= form_with(model: @user) do |f| %>
```

---

## 10.2 SQL Injection Prevention

```ruby
# ❌ NEVER interpolate user input into SQL
User.where("name = '#{params[:name]}'")                    # ❌ VULNERABLE
User.where("name = " + params[:name])                      # ❌ VULNERABLE
User.where("role IN (#{params[:roles].join(',')})")         # ❌ VULNERABLE

# ✅ ALWAYS use parameterized queries
User.where("name = ?", params[:name])                      # ✅ Safe
User.where(name: params[:name])                            # ✅ Safe
User.where("name LIKE ?", "%#{User.sanitize_sql_like(params[:q])}%")  # ✅ Safe
User.where(role: params[:roles])                           # ✅ Safe (array)

# ✅ For dynamic column names (use allowlist)
SORTABLE_COLUMNS = %w[name email created_at].freeze
column = SORTABLE_COLUMNS.include?(params[:sort]) ? params[:sort] : "created_at"
User.order(column => :asc)

# ✅ Use sanitize methods
ActiveRecord::Base.sanitize_sql_like(user_input)
ActiveRecord::Base.sanitize_sql_array(["name = ?", name])
```

---

## 10.3 XSS (Cross-Site Scripting) Prevention

```erb
<%# Rails auto-escapes output by default %>
<%= @user.name %>          <%# ✅ Auto-escaped %>
<%= @user.bio %>           <%# ✅ Auto-escaped %>

<%# ❌ DANGEROUS: Bypasses escaping %>
<%= raw @user.bio %>              <%# ❌ Never with user input %>
<%= @user.bio.html_safe %>        <%# ❌ Never with user input %>
<%== @user.bio %>                 <%# ❌ Never with user input %>

<%# ✅ When you need to render HTML, use sanitize %>
<%= sanitize @user.bio, tags: %w[p br strong em a], attributes: %w[href] %>

<%# ✅ For JSON in script tags %>
<script>
  var userData = <%= @user.to_json.html_safe %>;  <%# Safe because to_json escapes %>
</script>

<%# ✅ Better: Use data attributes %>
<div data-user="<%= @user.to_json %>">
```

```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self, :https
    policy.font_src    :self, :https, :data
    policy.img_src     :self, :https, :data
    policy.object_src  :none
    policy.script_src  :self, :https
    policy.style_src   :self, :https, :unsafe_inline
    policy.connect_src :self, :https, "wss://#{ENV['DOMAIN']}"

    # Report violations
    policy.report_uri "/csp-violation-report-endpoint"
  end

  # Nonce for inline scripts
  config.content_security_policy_nonce_generator = ->(_request) { SecureRandom.base64(16) }
  config.content_security_policy_nonce_directives = %w[script-src style-src]
end
```

---

## 10.4 Authentication Conventions

```ruby
# Convention: Use has_secure_password (built-in)
class User < ApplicationRecord
  has_secure_password
  # Requires: password_digest column
  # Provides: password=, password_confirmation=, authenticate methods
end

# Session-based auth
class SessionsController < ApplicationController
  def create
    user = User.find_by(email: params[:email])
    if user&.authenticate(params[:password])
      session[:user_id] = user.id
      redirect_to root_path, notice: "Logged in!"
    else
      flash.now[:alert] = "Invalid email or password"
      render :new, status: :unprocessable_entity
    end
  end

  def destroy
    session.delete(:user_id)
    redirect_to root_path, notice: "Logged out!"
  end
end

# Convention: Store minimal data in session (just user ID)
# Convention: Use bcrypt for password hashing (via has_secure_password)
# Convention: Never store plaintext passwords anywhere
```

### Token-based Auth (APIs)

```ruby
class Api::BaseController < ActionController::API
  before_action :authenticate_api_user!

  private

  def authenticate_api_user!
    token = request.headers["Authorization"]&.remove("Bearer ")
    @current_user = User.find_by(api_token: token)
    render json: { error: "Unauthorized" }, status: :unauthorized unless @current_user
  end
end
```

---

## 10.5 Authorization Conventions

```ruby
# Convention: Use Pundit or Action Policy for authorization
# Pundit example:
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  def update?
    user.admin? || record.user == user
  end

  def destroy?
    user.admin?
  end

  class Scope < Scope
    def resolve
      if user.admin?
        scope.all
      else
        scope.where(user: user)
      end
    end
  end
end

# In controller:
class PostsController < ApplicationController
  def update
    @post = Post.find(params[:id])
    authorize @post
    # ...
  end

  def index
    @posts = policy_scope(Post)
  end
end
```

---

## 10.6 Parameter Filtering

```ruby
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [
  :passw, :secret, :token, :_key, :crypt, :salt,
  :certificate, :otp, :ssn, :credit_card,
  :cvv, :card_number, :api_key, :access_token
]

# These parameters will show as [FILTERED] in logs
# Parameters: {"user"=>{"email"=>"john@example.com", "password"=>"[FILTERED]"}}
```

---

## 10.7 Mass Assignment Protection

```ruby
# Convention: ALWAYS use strong parameters
# ❌ NEVER use params.permit!
def user_params
  params.permit!  # ❌ Allows ALL parameters — DANGEROUS

  # ✅ Explicitly whitelist each parameter
  params.require(:user).permit(:name, :email, :bio)
end

# ❌ NEVER allow role/admin params from user input without authorization
def user_params
  if current_user.admin?
    params.require(:user).permit(:name, :email, :role, :admin)
  else
    params.require(:user).permit(:name, :email)
  end
end
```

---

## 10.8 HTTP Security Headers

```ruby
# config/initializers/permissions_policy.rb
Rails.application.config.permissions_policy do |policy|
  policy.camera      :none
  policy.gyroscope   :none
  policy.microphone  :none
  policy.usb         :none
  policy.fullscreen  :self
  policy.payment     :self, "https://stripe.com"
end

# Force SSL in production
# config/environments/production.rb
config.force_ssl = true

# Additional headers via middleware or Rack
config.action_dispatch.default_headers = {
  "X-Frame-Options" => "SAMEORIGIN",
  "X-XSS-Protection" => "0",  # Disabled (CSP is better)
  "X-Content-Type-Options" => "nosniff",
  "X-Permitted-Cross-Domain-Policies" => "none",
  "Referrer-Policy" => "strict-origin-when-cross-origin"
}
```

---

## 10.9 File Upload Security

```ruby
# ✅ Validate content type
class User < ApplicationRecord
  has_one_attached :avatar

  validates :avatar, content_type: { in: ["image/png", "image/jpeg", "image/gif"],
                                      message: "must be a PNG, JPEG, or GIF" },
                     size: { less_than: 5.megabytes,
                             message: "must be less than 5MB" }
end

# ✅ Don't serve uploads from your domain (use CDN or separate domain)
# ✅ Scan uploads for malware in production
# ✅ Randomize upload filenames (Active Storage does this by default)
```

---

## 10.10 Rate Limiting (Rails 8)

```ruby
# Rails 8 built-in rate limiting
class SessionsController < ApplicationController
  rate_limit to: 10, within: 3.minutes, only: :create, with: -> {
    redirect_to new_session_path, alert: "Try again later."
  }
end

# For older Rails, use rack-attack gem
# config/initializers/rack_attack.rb
Rack::Attack.throttle("requests by ip", limit: 100, period: 1.minute) do |request|
  request.ip
end

Rack::Attack.throttle("login attempts", limit: 5, period: 20.seconds) do |request|
  request.path == "/login" && request.post? && request.ip
end
```

---

## 10.11 Secrets Checklist

```
✅ master.key is in .gitignore
✅ Credentials are encrypted (credentials.yml.enc)
✅ No hardcoded passwords/tokens in code
✅ API keys use environment variables or credentials
✅ Database passwords not in plain text
✅ Session secret is unique per environment
✅ Production uses different keys than development
✅ Old/rotated secrets are revoked
```

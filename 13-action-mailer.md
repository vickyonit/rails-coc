# 13 — Action Mailer Conventions

---

## 13.1 Mailer Structure

```ruby
# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  default from: "noreply@example.com",
          reply_to: "support@example.com"
  layout "mailer"

  # Shared helper for all mailers
  helper :application
  helper :mailer  # app/helpers/mailer_helper.rb
end

# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  def welcome_email(user)
    @user = user
    @login_url = login_url
    mail(to: @user.email, subject: "Welcome to MyApp!")
  end

  def password_reset(user)
    @user = user
    @reset_url = edit_password_url(token: @user.reset_token)
    mail(to: @user.email, subject: "Password Reset Instructions")
  end

  def order_confirmation(order)
    @order = order
    @user = order.user
    attachments["receipt.pdf"] = ReceiptPdf.new(@order).render
    mail(to: @user.email, subject: "Order ##{@order.number} Confirmed")
  end
end
```

---

## 13.2 Mailer Views

```
app/views/user_mailer/
├── welcome_email.html.erb       # HTML version
├── welcome_email.text.erb       # Plain text version (always provide both!)
├── password_reset.html.erb
└── password_reset.text.erb
```

```erb
<%# app/views/layouts/mailer.html.erb %>
<!DOCTYPE html>
<html>
  <head>
    <style>
      /* Inline CSS for email clients */
      body { font-family: Arial, sans-serif; }
    </style>
  </head>
  <body>
    <%= yield %>
    <footer>
      <p>© <%= Date.current.year %> MyApp. All rights reserved.</p>
    </footer>
  </body>
</html>

<%# app/views/user_mailer/welcome_email.html.erb %>
<h1>Welcome, <%= @user.name %>!</h1>
<p>Thanks for joining. Get started by visiting your
   <%= link_to "dashboard", @login_url %>.</p>
```

---

## 13.3 Sending Conventions

```ruby
# ✅ ALWAYS use deliver_later (async via Active Job)
UserMailer.welcome_email(user).deliver_later

# ✅ With scheduling
UserMailer.reminder(user).deliver_later(wait: 1.hour)
UserMailer.digest(user).deliver_later(wait_until: Date.tomorrow.morning)

# ❌ AVOID deliver_now (blocks the request) except in rare cases
UserMailer.welcome_email(user).deliver_now  # Only for critical/immediate emails

# Convention: Use _url helpers (not _path) in mailers — emails need full URLs
# Convention: Always provide both HTML and plain text versions
# Convention: Use deliver_later for all non-critical emails
```

---

## 13.4 Mailer Previews

```ruby
# test/mailers/previews/user_mailer_preview.rb
class UserMailerPreview < ActionMailer::Preview
  def welcome_email
    UserMailer.welcome_email(User.first)
  end

  def password_reset
    user = User.first
    user.reset_token = "preview-token-123"
    UserMailer.password_reset(user)
  end
end

# Visit: http://localhost:3000/rails/mailers
```

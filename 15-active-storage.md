

---

# 15 — Active Storage Conventions

---

## 15.1 Attachment Declarations

```ruby
class User < ApplicationRecord
  has_one_attached :avatar
  has_many_attached :documents
end

class User < ApplicationRecord
  has_one_attached :avatar do |attachable|
    attachable.variant :thumb, resize_to_limit: [100, 100]
    attachable.variant :medium, resize_to_limit: [300, 300]
  end
end
```

---

## 15.2 Validations

```ruby
# Using active_storage_validations gem
class User < ApplicationRecord
  has_one_attached :avatar

  validates :avatar,
    content_type: { in: ["image/png", "image/jpeg", "image/webp"], message: "must be an image" },
    size: { less_than: 5.megabytes, message: "must be less than 5MB" },
    dimension: { width: { in: 100..4000 }, height: { in: 100..4000 } }
end
```

---

## 15.3 Service Configuration

```yaml
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>

amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: us-east-1
  bucket: my-app-production
```

```ruby
# config/environments/production.rb
config.active_storage.service = :amazon
```

---

## 15.4 Usage Conventions

```ruby
# Attaching files
user.avatar.attach(params[:avatar])
user.avatar.attach(io: File.open("/path/to/file"), filename: "avatar.jpg", content_type: "image/jpeg")

# Checking attachment
user.avatar.attached?

# URL generation
url_for(user.avatar)
rails_blob_path(user.avatar, disposition: "attachment")

# Variants (image processing)
user.avatar.variant(resize_to_limit: [200, 200])
user.avatar.variant(:thumb)  # Named variant

# Eager loading (prevent N+1)
User.with_attached_avatar
User.with_attached_documents

# Direct uploads (from browser)
<%= form.file_field :avatar, direct_upload: true %>
```

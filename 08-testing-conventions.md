# 08 — Testing Conventions

---

## 8.1 Test Directory Structure

### Minitest (Rails Default)

```
test/
├── channels/                  # Action Cable channel tests
├── controllers/               # Controller tests (functional)
├── fixtures/                  # YAML test data
│   ├── users.yml
│   └── posts.yml
├── helpers/                   # Helper tests
├── integration/               # Integration/request tests
├── jobs/                      # Active Job tests
├── mailers/                   # Mailer tests
│   └── previews/              # Mailer previews
├── models/                    # Model/unit tests
├── system/                    # Browser/system tests
├── application_system_test_case.rb
└── test_helper.rb
```

### RSpec (Popular Alternative)

```
spec/
├── channels/
├── factories/                 # FactoryBot factories
│   ├── users.rb
│   └── posts.rb
├── features/ or system/       # Feature/system specs
├── helpers/
├── jobs/
├── mailers/
├── models/
├── requests/                  # Request specs (preferred over controller specs)
├── routing/
├── services/                  # Service object specs
├── support/                   # Shared helpers and configs
│   ├── factory_bot.rb
│   ├── database_cleaner.rb
│   └── shared_examples.rb
├── rails_helper.rb
└── spec_helper.rb
```

---

## 8.2 Minitest Conventions

```ruby
# test/models/user_test.rb
require "test_helper"

class UserTest < ActiveSupport::TestCase
  # Setup and teardown
  setup do
    @user = users(:john)  # From fixtures
  end

  # Convention: Test names start with "test_" or use "test" block
  test "should be valid with all required attributes" do
    assert @user.valid?
  end

  test "should require email" do
    @user.email = nil
    assert_not @user.valid?
    assert_includes @user.errors[:email], "can't be blank"
  end

  test "should require unique email" do
    duplicate = @user.dup
    duplicate.email = @user.email
    assert_not duplicate.valid?
  end

  test "should normalize email before saving" do
    @user.update!(email: "  TEST@Example.COM  ")
    assert_equal "test@example.com", @user.reload.email
  end

  # Alternative method-style naming
  def test_full_name_concatenation
    @user.first_name = "John"
    @user.last_name = "Doe"
    assert_equal "John Doe", @user.full_name
  end
end
```

### Minitest Assertions

```ruby
assert value                          # Truthy
assert_not value                      # Falsy
assert_equal expected, actual         # Equality
assert_not_equal expected, actual
assert_nil value
assert_not_nil value
assert_includes collection, item
assert_match /regex/, string
assert_raises(ErrorClass) { block }
assert_no_difference("User.count") { block }
assert_difference("User.count", 1) { block }
assert_changes("@user.name") { block }
assert_no_changes("@user.email") { block }
assert_enqueued_jobs 1 { block }
assert_enqueued_emails 1 { block }
assert_performed_jobs 1 { block }
```

---

## 8.3 RSpec Conventions

```ruby
# spec/models/user_spec.rb
require "rails_helper"

RSpec.describe User, type: :model do
  # Subject
  subject(:user) { build(:user) }

  # Associations
  describe "associations" do
    it { is_expected.to belong_to(:organization) }
    it { is_expected.to have_many(:posts).dependent(:destroy) }
    it { is_expected.to have_one(:profile) }
  end

  # Validations
  describe "validations" do
    it { is_expected.to validate_presence_of(:email) }
    it { is_expected.to validate_uniqueness_of(:email).case_insensitive }
    it { is_expected.to validate_length_of(:name).is_at_most(100) }
  end

  # Scopes
  describe ".active" do
    it "returns only active users" do
      active_user = create(:user, active: true)
      inactive_user = create(:user, active: false)

      expect(User.active).to include(active_user)
      expect(User.active).not_to include(inactive_user)
    end
  end

  # Instance methods
  describe "#full_name" do
    it "concatenates first and last name" do
      user = build(:user, first_name: "John", last_name: "Doe")
      expect(user.full_name).to eq("John Doe")
    end

    it "handles missing last name" do
      user = build(:user, first_name: "John", last_name: nil)
      expect(user.full_name).to eq("John")
    end
  end

  # Callbacks
  describe "callbacks" do
    it "normalizes email before save" do
      user = create(:user, email: "TEST@Example.COM")
      expect(user.email).to eq("test@example.com")
    end
  end

  # Context blocks for different scenarios
  context "when admin" do
    let(:user) { create(:user, role: :admin) }

    it "returns true for admin?" do
      expect(user).to be_admin
    end
  end
end
```

### RSpec Best Practices

```ruby
# ✅ Use let (lazy) and let! (eager)
let(:user) { create(:user) }        # Created only when first called
let!(:user) { create(:user) }       # Created before each test

# ✅ Use build (not create) when you don't need persistence
let(:user) { build(:user) }

# ✅ Use described_class
RSpec.describe User do
  it "creates a new instance" do
    expect(described_class.new).to be_a(User)
  end
end

# ✅ One expectation per test (generally)
# ✅ Use contexts to group related tests
# ✅ Use shared_examples for repeated behavior
shared_examples "a timestamped record" do
  it { is_expected.to respond_to(:created_at) }
  it { is_expected.to respond_to(:updated_at) }
end

RSpec.describe User do
  it_behaves_like "a timestamped record"
end
```

---

## 8.4 Fixtures (Minitest Default)

```yaml
# test/fixtures/users.yml
# Convention: Use meaningful, descriptive names

john:
  name: John Doe
  email: john@example.com
  password_digest: <%= BCrypt::Password.create("password") %>
  role: member
  active: true

admin:
  name: Admin User
  email: admin@example.com
  password_digest: <%= BCrypt::Password.create("password") %>
  role: admin
  active: true

inactive_user:
  name: Inactive User
  email: inactive@example.com
  password_digest: <%= BCrypt::Password.create("password") %>
  role: member
  active: false

# With associations
# test/fixtures/posts.yml
johns_first_post:
  title: First Post
  body: Hello world
  user: john           # References the fixture name, not ID
  published: true
```

---

## 8.5 FactoryBot (RSpec Standard)

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name { Faker::Name.name }
    email { Faker::Internet.unique.email }
    password { "password123" }
    password_confirmation { "password123" }
    role { :member }
    active { true }

    # Traits for variations
    trait :admin do
      role { :admin }
    end

    trait :inactive do
      active { false }
    end

    trait :with_posts do
      after(:create) do |user|
        create_list(:post, 3, user: user)
      end
    end

    trait :with_profile do
      after(:create) do |user|
        create(:profile, user: user)
      end
    end

    # Named factories
    factory :admin_user, traits: [:admin]
  end
end

# Usage:
build(:user)                    # In-memory, not persisted
create(:user)                   # Persisted to database
create(:user, :admin)           # With trait
create(:user, name: "Custom")   # With overrides
create_list(:user, 5)           # Create 5 users
build_stubbed(:user)            # Stubbed (fastest, no DB)
```

---

## 8.6 Request / Integration Tests

```ruby
# test/integration/user_flows_test.rb (Minitest)
require "test_helper"

class UserFlowsTest < ActionDispatch::IntegrationTest
  test "user can sign up and view profile" do
    get new_user_path
    assert_response :success

    post users_path, params: {
      user: { name: "New User", email: "new@example.com", password: "password" }
    }
    assert_redirected_to user_path(User.last)

    follow_redirect!
    assert_response :success
    assert_select "h1", "New User"
  end
end

# spec/requests/users_spec.rb (RSpec)
require "rails_helper"

RSpec.describe "Users", type: :request do
  describe "GET /users" do
    it "returns a successful response" do
      get users_path
      expect(response).to have_http_status(:success)
    end

    it "returns all users as JSON" do
      create_list(:user, 3)
      get users_path, headers: { "Accept" => "application/json" }

      expect(response).to have_http_status(:success)
      expect(JSON.parse(response.body).size).to eq(3)
    end
  end

  describe "POST /users" do
    let(:valid_params) { { user: attributes_for(:user) } }

    it "creates a new user" do
      expect {
        post users_path, params: valid_params
      }.to change(User, :count).by(1)
    end

    it "returns errors for invalid params" do
      post users_path, params: { user: { email: "" } }
      expect(response).to have_http_status(:unprocessable_entity)
    end
  end
end
```

---

## 8.7 System Tests (Browser Tests)

```ruby
# test/system/users_test.rb (Minitest)
require "application_system_test_case"

class UsersTest < ApplicationSystemTestCase
  test "creating a user" do
    visit new_user_path

    fill_in "Name", with: "John Doe"
    fill_in "Email", with: "john@example.com"
    fill_in "Password", with: "password123"
    click_on "Create User"

    assert_text "User was successfully created"
    assert_text "John Doe"
  end
end

# spec/system/users_spec.rb (RSpec)
require "rails_helper"

RSpec.describe "Users", type: :system do
  before do
    driven_by(:selenium_chrome_headless)
  end

  it "allows creating a user" do
    visit new_user_path

    fill_in "Name", with: "John Doe"
    fill_in "Email", with: "john@example.com"
    click_on "Create User"

    expect(page).to have_text("User was successfully created")
  end
end
```

---

## 8.8 Test Naming Conventions

```ruby
# Minitest: test names describe the assertion
test "should not save user without email" do
test "should calculate total correctly with discount" do
test "should send welcome email after creation" do

# RSpec: describe the behavior
describe "#full_name" do
  it "concatenates first and last name"
  it "returns first name when last name is nil"
  it "strips leading and trailing whitespace"
end

context "when user is admin" do
  it "allows editing other users"
  it "shows admin dashboard"
end

# Convention: Focus on BEHAVIOR, not implementation
# ✅ "should calculate correct total"
# ❌ "should call calculate_subtotal and add_tax methods"
```

---

## 8.9 Testing Jobs, Mailers, Channels

```ruby
# Job test
test "enqueues welcome email job" do
  assert_enqueued_with(job: SendWelcomeEmailJob) do
    User.create!(name: "Test", email: "test@example.com", password: "pass")
  end
end

# Mailer test
test "welcome email has correct subject" do
  user = users(:john)
  email = UserMailer.welcome_email(user)
  assert_equal "Welcome!", email.subject
  assert_equal [user.email], email.to
  assert_match "Hello John", email.body.encoded
end

# Mailer preview (for visual testing)
# test/mailers/previews/user_mailer_preview.rb
class UserMailerPreview < ActionMailer::Preview
  def welcome_email
    UserMailer.welcome_email(User.first)
  end
end
# Visit: http://localhost:3000/rails/mailers/user_mailer/welcome_email
```

---

## 8.10 Test Configuration

```ruby
# test/test_helper.rb
ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"
require "rails/test_help"

class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
  fixtures :all  # Load all fixtures

  # Helper methods available in all tests
  def sign_in(user)
    post login_path, params: { email: user.email, password: "password" }
  end
end

# Convention: Tests should be:
# - Independent (no test depends on another)
# - Repeatable (same result every time)
# - Fast (use build over create when possible)
# - Focused (test one thing per test)
```

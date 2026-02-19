# Ruby on Rails — 10: Testing

## Table of Contents
- [Testing Philosophy](#testing-philosophy)
- [RSpec Setup](#rspec-setup)
- [Model Specs](#model-specs)
- [Controller / Request Specs](#controller--request-specs)
- [Factory Bot](#factory-bot)
- [Shared Examples & Contexts](#shared-examples--contexts)
- [Mocking & Stubbing](#mocking--stubbing)
- [System Tests (Capybara)](#system-tests-capybara)

---

## Testing Philosophy

- **Unit tests** — Test a single class/method in isolation
- **Integration tests** — Test multiple components working together
- **System/E2E tests** — Test full browser flows
- **TDD (Test-Driven Development)** — Write test first, then code
- Aim for fast, isolated, reliable tests
- Test behavior, not implementation

---

## RSpec Setup

```ruby
# Gemfile
group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
  gem "shoulda-matchers"
end

group :test do
  gem "database_cleaner-active_record"
  gem "capybara"
  gem "selenium-webdriver"
end
```

```bash
rails generate rspec:install
# Creates: .rspec, spec/, spec/spec_helper.rb, spec/rails_helper.rb
```

```ruby
# spec/rails_helper.rb
require "spec_helper"
require "rails_helper"

RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
  config.include Devise::Test::IntegrationHelpers, type: :request

  config.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation)
  end

  config.around(:each) do |example|
    DatabaseCleaner.cleaning { example.run }
  end
end

# shoulda-matchers config
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

---

## Model Specs

```ruby
# spec/models/user_spec.rb
RSpec.describe User, type: :model do

  # Shoulda-matchers for quick validation/association tests
  describe "associations" do
    it { is_expected.to have_many(:posts).dependent(:destroy) }
    it { is_expected.to have_one(:profile) }
    it { is_expected.to belong_to(:organization).optional }
  end

  describe "validations" do
    it { is_expected.to validate_presence_of(:name) }
    it { is_expected.to validate_presence_of(:email) }
    it { is_expected.to validate_uniqueness_of(:email).case_insensitive }
    it { is_expected.to validate_length_of(:name).is_at_most(50) }
  end

  # Custom validation
  describe "#email_domain_allowed" do
    it "is invalid with a banned domain" do
      user = build(:user, email: "test@spam.com")
      expect(user).not_to be_valid
      expect(user.errors[:email]).to include("domain is not allowed")
    end

    it "is valid with an allowed domain" do
      user = build(:user, email: "test@gmail.com")
      expect(user).to be_valid
    end
  end

  # Instance methods
  describe "#full_name" do
    it "combines first and last name" do
      user = build(:user, first_name: "Alice", last_name: "Smith")
      expect(user.full_name).to eq("Alice Smith")
    end
  end

  describe "#admin?" do
    it "returns true for admin role" do
      user = build(:user, role: :admin)
      expect(user).to be_admin
    end

    it "returns false for regular users" do
      user = build(:user, role: :user)
      expect(user).not_to be_admin
    end
  end

  # Class methods / scopes
  describe ".active" do
    it "returns only active users" do
      active_user   = create(:user, active: true)
      inactive_user = create(:user, active: false)

      expect(User.active).to include(active_user)
      expect(User.active).not_to include(inactive_user)
    end
  end

  describe ".search" do
    let!(:alice) { create(:user, name: "Alice") }
    let!(:bob)   { create(:user, name: "Bob") }

    it "returns matching users" do
      expect(User.search("ali")).to contain_exactly(alice)
    end

    it "returns all users when query is empty" do
      expect(User.search("")).to include(alice, bob)
    end
  end

  # Callbacks
  describe "callbacks" do
    it "normalizes email before save" do
      user = create(:user, email: "  ALICE@EXAMPLE.COM  ")
      expect(user.reload.email).to eq("alice@example.com")
    end

    it "sends welcome email after creation" do
      expect {
        create(:user)
      }.to have_enqueued_mail(WelcomeMailer, :welcome_email)
    end
  end
end
```

---

## Controller / Request Specs

```ruby
# spec/requests/users_spec.rb (preferred in Rails 5+)
RSpec.describe "Users", type: :request do

  describe "GET /users" do
    let!(:users) { create_list(:user, 3) }

    it "returns 200 and list of users" do
      get users_path
      expect(response).to have_http_status(:ok)
      expect(response.body).to include(users.first.name)
    end

    context "when authenticated as admin" do
      let(:admin) { create(:user, :admin) }

      before { sign_in admin }

      it "includes all users including inactive" do
        inactive = create(:user, active: false)
        get users_path
        expect(response.body).to include(inactive.name)
      end
    end
  end

  describe "POST /users" do
    let(:valid_params) do
      { user: { name: "Alice", email: "alice@example.com", password: "Password1" } }
    end

    context "with valid params" do
      it "creates a user and redirects" do
        expect {
          post users_path, params: valid_params
        }.to change(User, :count).by(1)

        expect(response).to redirect_to(user_path(User.last))
      end
    end

    context "with invalid params" do
      it "does not create user and renders errors" do
        expect {
          post users_path, params: { user: { name: "" } }
        }.not_to change(User, :count)

        expect(response).to have_http_status(:unprocessable_entity)
      end
    end
  end

  # JSON API specs
  describe "GET /api/v1/users" do
    before { create_list(:user, 3) }

    it "returns JSON list of users" do
      get api_v1_users_path, headers: { "Accept" => "application/json" }
      body = JSON.parse(response.body)

      expect(response).to have_http_status(:ok)
      expect(body["users"].length).to eq(3)
      expect(body["users"].first).to include("name", "email")
    end
  end
end
```

---

## Factory Bot

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name  { Faker::Name.full_name }
    email { Faker::Internet.unique.email }
    password { "Password1" }
    active { true }
    role   { :user }

    # Traits — reusable modifications
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

    # Nested factory
    factory :admin_user do
      role { :admin }
    end
  end
end

# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    title   { Faker::Lorem.sentence }
    body    { Faker::Lorem.paragraphs(number: 3).join("\n") }
    published { true }
    association :user   # creates a user automatically

    trait :draft do
      published { false }
    end

    trait :with_comments do
      after(:create) do |post|
        create_list(:comment, 5, post: post)
      end
    end
  end
end

# Usage
build(:user)                         # not persisted
create(:user)                        # saved to DB
build_stubbed(:user)                 # fake-persisted (faster)
attributes_for(:user)                # just the attributes hash

create(:user, :admin)                # with admin trait
create(:user, :admin, :with_posts)   # multiple traits
create(:user, name: "Alice")         # override attributes
create_list(:user, 5)                # create 5 users
create_list(:user, 3, :admin)        # 3 admin users
```

---

## Shared Examples & Contexts

```ruby
# spec/support/shared_examples/api_authentication.rb
RSpec.shared_examples "requires authentication" do
  it "returns 401 when not authenticated" do
    send(http_method, path)
    expect(response).to have_http_status(:unauthorized)
  end
end

RSpec.shared_context "authenticated user" do
  let(:user) { create(:user) }
  before { sign_in user }
end

# Usage in specs
RSpec.describe "Posts API", type: :request do
  include_context "authenticated user"

  describe "GET /api/v1/posts" do
    it_behaves_like "requires authentication" do
      let(:http_method) { :get }
      let(:path) { api_v1_posts_path }
    end
  end
end
```

---

## Mocking & Stubbing

```ruby
RSpec.describe PaymentService, type: :service do
  let(:user) { create(:user) }

  describe "#charge" do
    context "when payment succeeds" do
      before do
        allow(Stripe::Charge).to receive(:create).and_return(
          double("charge", id: "ch_123", status: "succeeded")
        )
      end

      it "returns success" do
        result = described_class.new(user).charge(100)
        expect(result).to be_success
      end
    end

    context "when payment fails" do
      before do
        allow(Stripe::Charge).to receive(:create)
          .and_raise(Stripe::CardError.new("Declined", nil))
      end

      it "handles the error gracefully" do
        result = described_class.new(user).charge(100)
        expect(result).to be_failure
        expect(result.error).to eq("Declined")
      end
    end
  end

  # Spy / Message expectations
  it "calls the mailer after successful payment" do
    mailer_double = double("mailer")
    allow(PaymentMailer).to receive(:with).and_return(mailer_double)
    allow(mailer_double).to receive(:receipt).and_return(mailer_double)
    allow(mailer_double).to receive(:deliver_later)

    described_class.new(user).charge(100)

    expect(PaymentMailer).to have_received(:with).with(user: user, amount: 100)
  end
end

# instance_double — verifies the interface exists
user_double = instance_double(User, name: "Alice", email: "alice@example.com")
```

---

## System Tests (Capybara)

```ruby
# spec/system/user_registration_spec.rb
RSpec.describe "User Registration", type: :system do
  before do
    driven_by(:rack_test)  # or :selenium_chrome for JS
  end

  it "allows a new user to register" do
    visit new_user_registration_path

    fill_in "Name",     with: "Alice Smith"
    fill_in "Email",    with: "alice@example.com"
    fill_in "Password", with: "Password1"
    fill_in "Password confirmation", with: "Password1"

    click_button "Sign Up"

    expect(page).to have_current_path(root_path)
    expect(page).to have_text("Welcome, Alice!")
    expect(User.last.email).to eq("alice@example.com")
  end

  it "shows errors for invalid data" do
    visit new_user_registration_path
    click_button "Sign Up"

    expect(page).to have_text("Name can't be blank")
    expect(page).to have_text("Email can't be blank")
  end
end

# Capybara helpers
visit "/"
click_link "About"
click_button "Submit"
fill_in "Email", with: "test@example.com"
select "Admin", from: "Role"
check "Accept terms"
uncheck "Subscribe"
attach_file "Photo", Rails.root.join("spec/fixtures/photo.jpg")
find("#submit-btn").click
within("#nav") { click_link "Login" }
page.has_text?("Welcome")
page.has_css?(".error")
expect(page).to have_selector("table tr", count: 5)
```

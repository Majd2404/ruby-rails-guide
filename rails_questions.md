# Interview Prep — Rails Questions

> Most common Rails interview questions with complete answers.

---

## Architecture & Design

### Q: Explain the MVC pattern in Rails

```
M (Model)      — Business logic, data validation, database interaction (ActiveRecord)
V (View)       — Presentation layer: ERB templates, JSON serializers (ActionView)
C (Controller) — Request handling, coordinates M and V (ActionController)

Request flow:
Browser → Router → Controller → Model (+ DB) → View → Response
```

---

### Q: What is ActiveRecord? What pattern does it implement?

> ActiveRecord implements the **Active Record design pattern** (from Martin Fowler's Patterns of Enterprise Application Architecture). Each model class maps to a database table, and each instance represents a row.

```ruby
# Rails automatically infers:
# - Table name from class name (User → users, BlogPost → blog_posts)
# - Primary key (id)
# - Foreign keys (user_id for belongs_to :user)

class User < ApplicationRecord
  # Convention:
  # Table: users
  # Columns accessible as attributes: user.name, user.email
  # Timestamps: created_at, updated_at (auto-managed)
end
```

---

### Q: What is the difference between `render` and `redirect_to`?

```ruby
# render — renders a view template, DOES NOT make new HTTP request
def create
  @post = Post.new(post_params)
  if @post.save
    redirect_to @post  # new request: GET /posts/:id
  else
    render :new, status: :unprocessable_entity
    # renders new.html.erb with current @post (with errors)
    # NO new request — same URL, same controller action's instance variables
  end
end

# Key differences:
# redirect_to — sends 302 response, browser makes NEW request
#   - use after successful create/update/destroy
#   - prevents double-submit on refresh
#   - instance variables are LOST

# render — sends HTML directly in current response
#   - use when showing errors (keep @post with errors)
#   - instance variables are PRESERVED
#   - browser URL doesn't change
```

---

### Q: What is the N+1 query problem and how do you fix it?

```ruby
# THE PROBLEM
posts = Post.all                   # Query 1: SELECT * FROM posts
posts.each do |post|
  puts post.user.name              # Query per post! (N queries)
end
# 100 posts = 101 queries!

# FIX 1: includes (most common)
posts = Post.includes(:user)       # 2 queries total:
# SELECT * FROM posts
# SELECT * FROM users WHERE id IN (1, 2, 3, ...)
posts.each do |post|
  puts post.user.name              # No extra query
end

# FIX 2: eager_load (one LEFT JOIN query)
posts = Post.eager_load(:user)     # 1 query with LEFT JOIN
# Useful when you need to filter by association columns

# FIX 3: preload (separate queries, like includes default)
posts = Post.preload(:user)

# Nested associations
Post.includes(:user, comments: [:author, :likes])

# DETECT N+1: use the Bullet gem
# gem 'bullet'
# Alerts you in development when N+1 occurs

# VERIFY: check query count in tests
expect {
  get :index
}.to make_database_queries(count: 2)
```

---

### Q: Explain the difference between `includes`, `joins`, `eager_load`, and `preload`

```ruby
# joins — INNER JOIN — for filtering/sorting, no data loading
Post.joins(:user).where(users: { active: true })
# Only loads Post data! Accessing post.user still triggers query.

# includes — smart eager loading (picks preload or eager_load)
Post.includes(:user)
# If no WHERE on joined table → 2 separate queries (preload behavior)
# If WHERE on joined table → 1 LEFT JOIN query (eager_load behavior)

# preload — always uses separate queries (like includes default)
Post.preload(:user)
# SELECT * FROM posts
# SELECT * FROM users WHERE id IN (...)

# eager_load — always uses LEFT JOIN
Post.eager_load(:user)
# SELECT posts.*, users.* FROM posts LEFT OUTER JOIN users ON...

# When to use what:
# joins    → filter/sort by association, don't need association data
# includes → most N+1 fixes (let Rails decide)
# eager_load → filter by association AND need association data
# preload  → when eager_load causes issues with large datasets
```

---

### Q: What are Rails callbacks? What are the risks?

```ruby
class User < ApplicationRecord
  before_validation :normalize_email
  before_save       :encrypt_password
  after_create      :send_welcome_email
  after_commit      :clear_cache
end

# RISKS of callbacks:
# 1. Hidden business logic — hard to trace
# 2. Makes testing hard (can't save without triggering callbacks)
# 3. Slows down tests (emails, jobs, etc.)
# 4. Order dependency — changing order can break things
# 5. Can cause unexpected behavior in admin/rake tasks

# Example: accidentally sending emails in seeds
# db/seeds.rb
User.create!(name: "Admin")  # triggers welcome email!

# Better alternatives for complex logic: Service Objects
class UserRegistration
  def initialize(user)
    @user = user
  end

  def call
    User.transaction do
      @user.save!
      WelcomeMailer.with(user: @user).welcome.deliver_later
      Analytics.track("user_registered", user: @user)
    end
  end
end

# Use callbacks ONLY for:
# - Data normalization (before_save to normalize email)
# - Setting derived values (before_create to generate slug)
# - Maintaining consistency (after_destroy to delete files)
```

---

### Q: What is Strong Parameters and why is it important?

```ruby
# SECURITY: prevents mass assignment vulnerability
# Without it:
User.update_all(params[:user])  # attacker could send role: "admin"!

# Strong Parameters — whitelist what's allowed:
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    @user.save
  end

  private

  def user_params
    params.require(:user)         # must have :user key
          .permit(:name, :email, :password)  # only these fields
          # role is NOT listed — can't be mass-assigned
  end
end

# Nested params
def post_params
  params.require(:post).permit(
    :title, :body,
    tags: [],                          # array
    address: [:street, :city, :zip],   # nested hash
    metadata: {}                        # any hash keys
  )
end

# If attacker sends: { user: { name: "Bob", role: "admin" } }
# user_params will strip :role — it's not permitted!
```

---

### Q: What is the asset pipeline and how does it work?

```ruby
# Asset Pipeline (Sprockets / Propshaft):
# 1. Concatenate multiple CSS/JS files into one
# 2. Minify (remove whitespace, shorten variables)
# 3. Fingerprint filenames (application-abc123.css) for cache busting
# 4. Compile assets (SCSS → CSS, TypeScript → JS)

# app/assets/stylesheets/application.css
# *= require_self
# *= require_tree .       (include all files in directory)

# In production:
rails assets:precompile   # compile all assets to public/assets/

# Accessing in views:
<%= stylesheet_link_tag "application" %>
<%= javascript_include_tag "application" %>
<%= image_tag "logo.png" %>
image_path("logo.png")  # => "/assets/logo-abc123.png"

# Rails 7+ options:
# importmaps — no bundler needed for simple apps
# jsbundling-rails — for complex JS (webpack/esbuild/rollup)
# propshaft — simpler replacement for Sprockets
```

---

### Q: Explain Rails caching strategies

```ruby
# 1. Page caching (removed in Rails 4, actionpack-page_caching gem)

# 2. Action caching (removed, actionpack-action_caching gem)

# 3. Fragment caching — most common
<% cache @post do %>
  <%= render @post %>
<% end %>

<% cache ["v1", @post] do %>
  <%= @post.title %>
<% end %>

# 4. Russian Doll caching — nested caches
<% cache ["v2", @post] do %>
  <%= render @post.comments %>  # each comment is also cached
<% end %>

# 5. Low-level caching
Rails.cache.fetch("user_count") do
  User.count   # only runs if cache is empty/expired
end

Rails.cache.fetch("user/#{user.id}/profile", expires_in: 1.hour) do
  expensive_profile_calculation(user)
end

Rails.cache.write("key", value, expires_in: 30.minutes)
Rails.cache.read("key")
Rails.cache.delete("key")
Rails.cache.exist?("key")

# Cache stores:
# :memory_store    — in-process, lost on restart
# :file_store      — on disk
# :redis_cache_store — Redis (recommended for production)
# :mem_cache_store   — Memcached
# :solid_cache_store — Database (Rails 7.1+ via Solid Cache gem)

# Conditional caching
skip_before_action :authenticate, if: -> { action_name == "index" }
```

---

### Q: What is a service object pattern?

```ruby
# Service objects extract complex business logic from controllers/models
# "Fat Model, Skinny Controller" → "Skinny Everything, Fat Service"

# app/services/user_registration_service.rb
class UserRegistrationService
  Result = Struct.new(:success, :user, :errors, keyword_init: true)

  def initialize(params)
    @params = params
  end

  def call
    user = User.new(@params)

    if user.save
      send_welcome_email(user)
      create_default_settings(user)
      notify_admins(user)
      Result.new(success: true, user: user, errors: [])
    else
      Result.new(success: false, user: user, errors: user.errors.full_messages)
    end
  end

  private

  def send_welcome_email(user)
    WelcomeMailer.with(user: user).welcome.deliver_later
  end

  def create_default_settings(user)
    UserSettings.create!(user: user, theme: :light, notifications: true)
  end

  def notify_admins(user)
    AdminNotificationJob.perform_later(user_id: user.id)
  end
end

# In controller (now very thin):
class UsersController < ApplicationController
  def create
    result = UserRegistrationService.new(user_params).call
    if result.success
      redirect_to result.user
    else
      @errors = result.errors
      render :new
    end
  end
end
```

---

### Q: Explain the differences between `has_many :through` and `has_and_belongs_to_many`

```ruby
# has_and_belongs_to_many (HABTM) — simple many-to-many, no join model
class Post < ApplicationRecord
  has_and_belongs_to_many :tags
end
# Requires posts_tags table with just post_id and tag_id
# No extra data on the join, no join model class

# has_many :through — many-to-many WITH a join model
class Post < ApplicationRecord
  has_many :taggings
  has_many :tags, through: :taggings
end

class Tagging < ApplicationRecord
  belongs_to :post
  belongs_to :tag
  # Can have extra columns: created_by, weight, etc.
end

# USE has_many :through when:
# - You need extra data on the relationship (created_at, notes)
# - You need to query/manipulate the join records directly
# - When in doubt (it's more flexible)

# USE HABTM when:
# - Pure many-to-many with no extra data
# - You'll never query the join table directly
# - Legacy code or very simple relationships
```

---

### Q: What is CSRF protection and how does Rails handle it?

```ruby
# CSRF (Cross-Site Request Forgery) — attacker tricks user into
# submitting a form to your app from a malicious site

# Rails protection:
# 1. ApplicationController includes protect_from_forgery
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception  # raises on invalid token
  # or:
  protect_from_forgery with: :null_session  # clears session instead
end

# 2. Forms automatically include authenticity token
<%= form_with model: @user do |f| %>
  <!-- Hidden field automatically added: -->
  <!-- <input name="authenticity_token" value="abc123..." type="hidden"> -->
  ...
<% end %>

# 3. Token verification on non-GET requests
# Rails checks that the submitted token matches the session token

# For API endpoints (stateless):
class ApiController < ActionController::API
  # ActionController::API skips CSRF by default
  # Use token-based auth instead (JWT, API keys)
end

# For Ajax requests:
# Rails includes meta tag in layout:
<%= csrf_meta_tags %>
# JavaScript reads these and includes in request headers
```

---

### Q: How does Rails handle database migrations?

```ruby
# Migrations are versioned changes to the database schema
# Each migration has a unique timestamp: 20240101000000_create_users.rb

# Run migrations:
rails db:migrate            # run pending migrations
rails db:rollback           # undo last migration
rails db:rollback STEP=3    # undo last 3
rails db:migrate VERSION=20240101000000  # go to specific version
rails db:migrate:status     # see what's run and what hasn't

# schema.rb vs structure.sql:
# schema.rb — default, Ruby DSL, db-agnostic
# structure.sql — raw SQL, needed for DB-specific features
# config/application.rb:
config.active_record.schema_format = :sql  # use structure.sql

# Safe migrations (avoid downtime):
# DON'T: add NOT NULL column without default to existing table
# DO: add nullable first, backfill data, then add NOT NULL constraint

# Wrong (locks table on large dataset):
add_column :users, :status, :string, null: false, default: "active"

# Right:
add_column :users, :status, :string              # Step 1: nullable
User.update_all(status: "active")                 # Step 2: backfill
change_column_null :users, :status, false         # Step 3: add constraint

# strong_migrations gem helps catch unsafe migrations
gem 'strong_migrations'
```

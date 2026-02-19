# Interview Prep — System Design & Advanced Topics

---

## System Design Questions

### Q: How would you design a background job system in Rails?

```ruby
# Active Job (Rails abstraction over background job adapters)
# Supported backends: Sidekiq, Resque, Delayed Job, Solid Queue, etc.

# Define a job
class SendEmailJob < ApplicationJob
  queue_as :default
  retry_on Net::SMTPError, wait: 5.minutes, attempts: 3
  discard_on ActiveJob::DeserializationError

  def perform(user_id, template)
    user = User.find(user_id)
    UserMailer.with(user: user).send(template).deliver_now
  end
end

# Enqueue a job
SendEmailJob.perform_later(user.id, :welcome)
SendEmailJob.set(wait: 10.minutes).perform_later(user.id, :reminder)
SendEmailJob.set(wait_until: 1.day.from_now).perform_later(user.id, :followup)
SendEmailJob.set(queue: :critical).perform_later(user.id, :password_reset)

# Sidekiq setup (most popular in production)
# Gemfile: gem 'sidekiq'
# config/application.rb:
config.active_job.queue_adapter = :sidekiq

# Sidekiq-specific options
class HeavyProcessingJob < ApplicationJob
  sidekiq_options retry: 5, dead: false
  queue_as :heavy
end

# Architecture recommendations:
# - Use separate queues for priority levels (critical, default, low)
# - Set appropriate timeouts and retry limits
# - Monitor with Sidekiq Web UI
# - Use Solid Queue (Rails 7.1+) for smaller apps (no Redis needed)
```

---

### Q: How would you implement API versioning?

```ruby
# Strategy 1: URL versioning (most common)
# /api/v1/users, /api/v2/users

# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :users
  end
  namespace :v2 do
    resources :users
  end
end

# Strategy 2: Header versioning
# Accept: application/vnd.myapp.v2+json

class Api::BaseController < ActionController::API
  before_action :set_api_version

  private

  def set_api_version
    @api_version = request.headers["Accept"]
                           &.match(/vnd\.myapp\.v(\d+)/)
                           &.captures&.first&.to_i || 1
  end
end

# Strategy 3: Subdomain (api.v1.example.com)
# Strategy 4: Query param (?version=2)

# Best practices:
# - Never break existing API contracts
# - Deprecate before removing
# - Document all versions
# - Use Semantic Versioning
```

---

### Q: How would you design a multi-tenant Rails app?

```ruby
# Strategy 1: Separate databases
class Tenant < ApplicationRecord
  def database_config
    { adapter: "postgresql", database: "tenant_#{slug}_production" }
  end
end

# Switch connection per request
class ApplicationController < ActionController::Base
  around_action :switch_database

  def switch_database
    tenant = Tenant.find_by(subdomain: request.subdomain)
    ActiveRecord::Base.establish_connection(tenant.database_config)
    yield
  ensure
    ActiveRecord::Base.establish_connection(:default)
  end
end

# Strategy 2: Schema-based (PostgreSQL)
# Each tenant has their own schema (namespace)
# Apartment gem: gem 'apartment'

# Strategy 3: Row-based (most common for SaaS)
# All tenants share tables, filter by tenant_id

class ApplicationRecord < ActiveRecord::Base
  def self.inherited(subclass)
    super
    subclass.belongs_to(:organization) if subclass.column_names.include?("organization_id")
  end
end

# Global scope for tenant isolation
module TenantScoped
  extend ActiveSupport::Concern

  included do
    belongs_to :organization
    default_scope { where(organization: Current.organization) }
  end
end

class Post < ApplicationRecord
  include TenantScoped
end

# Set current organization per request
class ApplicationController < ActionController::Base
  before_action :set_current_organization

  private

  def set_current_organization
    Current.organization = current_user.organization
  end
end

# Using Current attributes (thread-safe)
class Current < ActiveSupport::CurrentAttributes
  attribute :organization, :user
end
```

---

### Q: How would you handle file uploads at scale?

```ruby
# Active Storage (Rails built-in)
class User < ApplicationRecord
  has_one_attached :avatar
  has_many_attached :documents

  validates :avatar, content_type: ["image/jpeg", "image/png"],
                     size: { less_than: 5.megabytes }
end

# Direct uploads (to S3 without going through Rails server)
# config/storage.yml
amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: us-east-1
  bucket: myapp-production

# config/environments/production.rb
config.active_storage.service = :amazon

# Image processing with Active Storage variants
class User < ApplicationRecord
  has_one_attached :avatar do |attachable|
    attachable.variant :thumb, resize_to_limit: [100, 100]
    attachable.variant :medium, resize_to_limit: [400, 400]
  end
end

# In views:
<%= image_tag user.avatar.variant(:thumb) %>

# Background processing
# app/jobs/process_upload_job.rb
class ProcessUploadJob < ApplicationJob
  def perform(attachment_id)
    attachment = ActiveStorage::Attachment.find(attachment_id)
    # generate variants, run virus scan, etc.
    attachment.blob.variant(:thumb).processed
  end
end
```

---

### Q: How would you implement search in Rails?

```ruby
# Level 1: Basic database search
scope :search, ->(query) {
  where("name LIKE :q OR email LIKE :q", q: "%#{sanitize_sql_like(query)}%")
}

# Level 2: Full-text search with PostgreSQL
# Migration:
add_column :posts, :search_vector, :tsvector
execute <<-SQL
  CREATE INDEX posts_search_idx ON posts USING gin(search_vector);
SQL

# Trigger to update search vector:
execute <<-SQL
  CREATE OR REPLACE FUNCTION posts_search_trigger() RETURNS trigger AS $$
  begin
    new.search_vector :=
      setweight(to_tsvector('english', coalesce(new.title, '')), 'A') ||
      setweight(to_tsvector('english', coalesce(new.body, '')), 'B');
    return new;
  end
  $$ LANGUAGE plpgsql;

  CREATE TRIGGER posts_search_update
    BEFORE INSERT OR UPDATE ON posts
    FOR EACH ROW EXECUTE PROCEDURE posts_search_trigger();
SQL

# Query:
Post.where("search_vector @@ plainto_tsquery('english', ?)", query)
    .order(Arel.sql("ts_rank(search_vector, plainto_tsquery('english', #{connection.quote(query)})) DESC"))

# Level 3: Dedicated search engine
# Elasticsearch via Searchkick gem
class Post < ApplicationRecord
  searchkick word_start: [:title]  # fuzzy matching

  def search_data
    { title: title, body: body, tags: tags.map(&:name), published: published }
  end
end

Post.search("ruby rails", where: { published: true }, order: { _score: :desc })
Post.search("*")  # all records
Post.search("rails", boost_by: :views_count)
```

---

## Performance Optimization

### Q: How do you identify and fix performance issues?

```ruby
# 1. Query optimization
# Use EXPLAIN ANALYZE
ActiveRecord::Base.connection.execute("EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@x.com'")

# Or in rails console:
User.where(email: "alice@x.com").explain

# 2. Add database indexes
add_index :users, :email, unique: true
add_index :posts, [:user_id, :created_at]   # composite index
add_index :posts, :title                      # for search

# 3. Pagination (never load all records)
# kaminari or pagy gem
@posts = Post.page(params[:page]).per(25)
# or
@posts = Post.order(:created_at).page(1).per(25)

# 4. Background jobs for slow operations
# Don't do this in a request:
def create
  @user = User.create!(user_params)
  generate_pdf_report(@user)        # takes 5 seconds!
  send_50_emails(@user)             # takes 10 seconds!
end

# Do this instead:
def create
  @user = User.create!(user_params)
  GeneratePdfJob.perform_later(@user.id)
  BulkEmailJob.perform_later(@user.id)
  redirect_to @user
end

# 5. Caching
def expensive_calculation
  Rails.cache.fetch("calc/#{cache_key_with_version}", expires_in: 1.hour) do
    # heavy computation
  end
end

# 6. Database connection pooling
# config/database.yml
production:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

# 7. Bullet gem for N+1 detection
# Rack Mini Profiler for request profiling
# New Relic / Datadog / Scout for production APM
```

---

## Security Best Practices

```ruby
# 1. SQL Injection prevention — always use parameterized queries
# BAD:
User.where("name = '#{params[:name]}'")  # SQL injection!

# GOOD:
User.where(name: params[:name])
User.where("name = ?", params[:name])

# 2. XSS prevention — Rails auto-escapes in ERB
<%= user.name %>       # auto-escaped ✓
<%= raw user.name %>   # bypasses escaping — DANGEROUS
<%= user.name.html_safe %>  # bypasses escaping — DANGEROUS

# 3. Mass assignment — use Strong Parameters
def user_params
  params.require(:user).permit(:name, :email)
  # role is not permitted!
end

# 4. Authorization — Pundit / CanCanCan
class PostPolicy < ApplicationPolicy
  def update?
    user.admin? || record.user == user
  end
end

# 5. Secure cookies
Rails.application.config.session_store :cookie_store,
  key: "_app_session",
  secure: Rails.env.production?,   # HTTPS only
  httponly: true,                   # No JS access
  same_site: :lax                   # CSRF protection

# 6. Environment variables for secrets
# NEVER commit secrets to git
# Use Rails credentials:
Rails.application.credentials.dig(:stripe, :api_key)

# Or environment variables:
ENV['STRIPE_API_KEY']

# 7. Brakeman — static security scanner
# gem 'brakeman', require: false
# bundle exec brakeman
```

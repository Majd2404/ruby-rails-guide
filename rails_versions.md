# Version Differences — Rails 5 to 8

## Quick Summary

| Version | Key Highlights |
|---|---|
| Rails 5.0 | API mode, ActionCable, ApplicationRecord base |
| Rails 5.1 | Webpack, yarn, encrypted secrets, form_with |
| Rails 5.2 | Active Storage, credentials, Redis cache |
| Rails 6.0 | Action Mailbox, Action Text, multiple DBs, Webpacker default |
| Rails 6.1 | Delegated types, destroy_association_async, where.associated |
| Rails 7.0 | Hotwire default, importmaps, encrypted attributes, async queries |
| Rails 7.1 | Dockerfiles, Solid Queue/Cache, composite primary keys, generates_token_for |
| Rails 7.2 | Dev containers, default Progressive Web App support |
| Rails 8.0 | Solid Queue/Cable/Cache default, Kamal deployment, auth generator |

---

## Rails 5.0

```ruby
# ApplicationRecord base class (instead of directly < ActiveRecord::Base)
class User < ApplicationRecord
  # All models now inherit from this, which inherits from ActiveRecord::Base
  # Makes it easy to add shared model behavior
end

# ActionCable — WebSocket framework
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    stream_from "chat_#{params[:room]}"
  end

  def receive(data)
    ActionCable.server.broadcast("chat_#{params[:room]}", data)
  end
end

# API mode — lightweight Rails without views
# rails new myapi --api
class ApplicationController < ActionController::API
  # No view rendering, cookies, etc.
end

# has_secure_token
class User < ApplicationRecord
  has_secure_token :auth_token
  has_secure_token :reset_token
end
user = User.create(name: "Alice")
user.auth_token  # => "abc123xyz..." (auto-generated)

# Attributes API
class Product < ApplicationRecord
  attribute :price, :decimal, precision: 10, scale: 2
  attribute :status, :string, default: "pending"
  attribute :metadata, :json
end
```

---

## Rails 5.2

```ruby
# Active Storage — file uploads built-in
class User < ApplicationRecord
  has_one_attached :avatar
  has_many_attached :photos
end

user.avatar.attach(io: File.open("/path/to/image.jpg"), filename: "avatar.jpg")
user.avatar.attached?     # => true
user.avatar.url           # => "https://storage.example.com/..."

# In views
<%= image_tag user.avatar %>
<%= link_to "Download", rails_blob_path(user.avatar, disposition: "attachment") %>

# Credentials (replaces secrets.yml)
# rails credentials:edit
# config/credentials.yml.enc (encrypted)
Rails.application.credentials.dig(:aws, :access_key_id)
Rails.application.credentials.stripe[:api_key]

# Redis Cache Store
# config/environments/production.rb
config.cache_store = :redis_cache_store, { url: ENV["REDIS_URL"] }

# HTTP cache improvements
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    fresh_when(@post)  # ETags and Last-Modified automatically
  end
end
```

---

## Rails 6.0

```ruby
# Action Mailbox — incoming email routing
class SupportMailbox < ApplicationMailbox
  routing /support@/i => :support

  def process
    ticket = Ticket.create!(
      subject: mail.subject,
      body: mail.decoded,
      from: mail.from.first
    )
  end
end

# Action Text — rich text content
class Article < ApplicationRecord
  has_rich_text :body
end

# In migration:
add_column :articles, :body, :text
# Or create trix_rich_texts table via:
# rails action_text:install

# In view:
<%= form.rich_text_area :body %>
<%= @article.body %>

# Multiple databases
# config/database.yml
production:
  primary:
    <<: *default
    database: myapp_primary
  animals:
    <<: *default
    database: myapp_animals

class AnimalsRecord < ApplicationRecord
  self.abstract_class = true
  connects_to database: { writing: :animals, reading: :animals_replica }
end

class Dog < AnimalsRecord
end

# Parallel testing
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
end

# Action Mailer previews improved
# Webpacker is now default (webpack-based)
```

---

## Rails 6.1

```ruby
# Delegated types — elegant polymorphism alternative
# Instead of STI or full polymorphism:
class Entry < ApplicationRecord
  delegated_type :entryable, types: %w[Message Comment]
end

class Message < ApplicationRecord
  include Entry::Entryable
  # has entry-specific methods
end

class Comment < ApplicationRecord
  include Entry::Entryable
end

entry = Entry.create!(entryable: Message.new(subject: "Hello"))
entry.entryable_type   # => "Message"
entry.message          # => the Message record
entry.message?         # => true
entry.comment?         # => false

# where.associated — filter records with associations
Post.where.associated(:comments)    # posts that have at least one comment
User.where.missing(:profile)        # users without a profile

# destroy_association_async — destroy in background job
class User < ApplicationRecord
  has_many :posts, dependent: :destroy_async
end

# Strict loading — detect N+1 in development
class ApplicationRecord < ActiveRecord::Base
  self.strict_loading_by_default = true
end
# Or per-query:
Post.strict_loading.includes(:comments).each do |post|
  post.comments  # ✓ fine
  post.user      # ❌ StrictLoadingViolationError!
end

# Connection switching per-request
class ApplicationController < ActionController::Base
  around_action :switch_tenant

  def switch_tenant
    Tenant.with_tenant(current_tenant) { yield }
  end
end
```

---

## Rails 7.0 ⭐ Major Release

```ruby
# Hotwire = Turbo + Stimulus (default frontend)
# replaces Webpacker with importmaps by default

# Turbo Streams — real-time UI updates
# In controller:
def create
  @post = Post.create!(post_params)
  respond_to do |format|
    format.turbo_stream
    format.html { redirect_to @post }
  end
end

# app/views/posts/create.turbo_stream.erb
<%= turbo_stream.prepend "posts", @post %>
<%= turbo_stream.update "posts-count", Post.count %>

# Turbo Frames — partial page updates without full reload
<%= turbo_frame_tag "post_#{@post.id}" do %>
  <h1><%= @post.title %></h1>
  <%= link_to "Edit", edit_post_path(@post) %>
<% end %>

# Encrypted attributes
class User < ApplicationRecord
  encrypts :ssn, :phone_number
  encrypts :email, deterministic: true  # allows where() queries
end

user.ssn          # => "123-45-6789" (decrypted automatically)
User.where(email: "alice@example.com")  # works with deterministic

# Async query loading
def index
  @posts  = Post.all.load_async
  @users  = User.active.load_async
  # Both queries run in parallel!
  render  # @posts and @users are both ready here
end

# Query by attribute with scoping
Post.where(author: User.where(role: :admin))   # subquery

# Zeitwerk autoloading is now the only option (classic removed)

# importmap-rails (no Node.js needed for simple apps)
# config/importmap.rb
pin "application"
pin "@hotwired/turbo-rails", to: "turbo.min.js"
pin "@hotwired/stimulus", to: "stimulus.min.js"

# with_options
class User < ApplicationRecord
  with_options if: :admin? do
    validates :admin_notes, presence: true
  end
end
```

---

## Rails 7.1

```ruby
# Dockerfiles generated automatically
# rails new myapp --database=postgresql
# → generates Dockerfile, .dockerignore, docker-compose.yml

# Solid Queue — DB-backed job queue (no Redis needed)
# Gemfile:
gem "solid_queue"
# config/queue.yml automatically generated

# Solid Cache — DB-backed cache (no Redis needed)
gem "solid_cache"

# generates_token_for — better token generation
class User < ApplicationRecord
  generates_token_for :email_verification, expires_in: 2.days do
    email  # include email in token so it invalidates on change
  end

  generates_token_for :password_reset, expires_in: 15.minutes do
    password_salt.last(10)  # invalidates when password changes
  end
end

token = user.generate_token_for(:email_verification)
User.find_by_token_for(:email_verification, token)  # => user or nil

# Composite primary keys
class Order < ApplicationRecord
  self.primary_key = [:shop_id, :id]
end

# Normalizes — clean up data before saving
class User < ApplicationRecord
  normalizes :email, with: ->(email) { email.strip.downcase }
  normalizes :phone, with: ->(phone) { phone.gsub(/\D/, "") }
end

user = User.new(email: "  ALICE@EXAMPLE.COM  ")
user.email  # => "alice@example.com"

# ActiveRecord::Base#in_order_of
Post.in_order_of(:status, %w[featured published draft])

# Better error pages in development
# More informative with better stack traces
```

---

## Rails 8.0

```ruby
# Authentication generator (built-in!)
rails generate authentication
# Generates: User, Session models, SessionsController
# Complete, production-ready auth scaffold

# Solid trio as default (no Redis/external dependencies!)
# Solid Queue  — background jobs
# Solid Cache  — caching
# Solid Cable  — WebSocket adapter

# Kamal deployment built-in
# config/deploy.yml automatically generated
# Deploys to any server via Docker + SSH

# Propshaft asset pipeline (replaces Sprockets as default)

# Brakeman security scanner built-in
# rails brakeman

# Rubocop integration
# rails rubocop

# No more bin/importmap (importmaps simplified)

# Improved ActiveRecord
class User < ApplicationRecord
  # password_challenge — validate current password on update
  has_secure_password

  attr_accessor :password_challenge
  validates :password, confirmation: true
end

# allow_browser for version requirements
class ApplicationController < ActionController::Base
  allow_browser versions: :modern
end
```

---

## Upgrade Tips

### Key Changes Per Version

```
Rails 5 → 6:
- gems: update all gems, especially devise, pundit
- Webpacker replaces asset pipeline for JS
- Action Text / Mailbox are opt-in

Rails 6 → 7:
- Webpacker → importmaps or jsbundling-rails
- jQuery likely not needed anymore
- Turbo replaces UJS (rails-ujs)
- `form_with` defaults to non-remote (was remote in 6)

Rails 7 → 8:
- sprockets → propshaft (optional)
- authentication generator available
- Solid * gems replace Redis for most use cases
```

```bash
# Check your app for upgrade issues
bundle exec rails app:update

# Useful gems for upgrading
gem 'rails_upgrade'       # automated upgrade help
gem 'strong_migrations'   # safe migration checks
```

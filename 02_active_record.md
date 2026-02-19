# Ruby on Rails — 02: ActiveRecord Deep Dive

> ActiveRecord is Rails' ORM (Object-Relational Mapper). It bridges Ruby objects and database tables.

## Table of Contents
- [Migrations](#migrations)
- [Validations](#validations)
- [Associations](#associations)
- [Querying](#querying)
- [Scopes](#scopes)
- [Callbacks](#callbacks)
- [N+1 Problem & Eager Loading](#n1-problem--eager-loading)
- [Transactions](#transactions)
- [Counter Cache & Touch](#counter-cache--touch)

---

## Migrations

```ruby
# db/migrate/20240101000000_create_users.rb
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string  :name,    null: false
      t.string  :email,   null: false
      t.integer :age
      t.string  :role,    default: "user"
      t.boolean :active,  default: true
      t.text    :bio
      t.date    :born_on

      t.timestamps         # adds created_at and updated_at
    end

    add_index :users, :email, unique: true
    add_index :users, :role
  end
end

# Adding columns
class AddPhoneToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :phone, :string
    add_column :users, :confirmed_at, :datetime
  end
end

# Removing columns
class RemoveAgeFromUsers < ActiveRecord::Migration[7.0]
  def change
    remove_column :users, :age, :integer  # type needed for reversibility
  end
end

# Complex migration with up/down
class RenameUserFullName < ActiveRecord::Migration[7.0]
  def up
    rename_column :users, :name, :full_name
  end

  def down
    rename_column :users, :full_name, :name
  end
end

# Column types reference:
# :string      (VARCHAR 255)
# :text        (TEXT — long content)
# :integer     (INT)
# :bigint      (BIGINT)
# :float       (FLOAT)
# :decimal     (DECIMAL — use for money!)
# :boolean     (BOOLEAN)
# :date        (DATE)
# :datetime    (DATETIME)
# :timestamps  (created_at + updated_at)
# :references  (foreign key — same as :bigint + index)
# :json        (JSON — PostgreSQL)
# :jsonb       (JSONB — PostgreSQL, faster, indexable)

# Money example — never use float!
add_column :orders, :amount_cents, :integer, default: 0, null: false
# Use the 'money-rails' gem for this pattern
```

---

## Validations

```ruby
class User < ApplicationRecord
  # Presence
  validates :name, presence: true
  validates :email, presence: true

  # Length
  validates :name,  length: { minimum: 2, maximum: 50 }
  validates :bio,   length: { maximum: 500 }, allow_blank: true
  validates :zip,   length: { is: 5 }

  # Format
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :phone, format: { with: /\A\+?[\d\s\-]{7,15}\z/ }, allow_blank: true

  # Uniqueness
  validates :email, uniqueness: true
  validates :username, uniqueness: { case_sensitive: false }
  validates :slug, uniqueness: { scope: :blog_id }  # unique within scope

  # Inclusion / Exclusion
  validates :role, inclusion: { in: %w[admin user moderator] }
  validates :username, exclusion: { in: %w[admin root superuser] }

  # Numericality
  validates :age, numericality: { greater_than: 0, less_than: 150 }
  validates :score, numericality: { only_integer: true }

  # Acceptance (checkboxes)
  validates :terms_of_service, acceptance: true

  # Confirmation (password re-entry)
  validates :password, confirmation: true
  validates :password_confirmation, presence: true, if: :password_changed?

  # Multiple attributes
  validates :name, :email, presence: true
  validates :name, :email, length: { maximum: 255 }

  # Conditional validations
  validates :company_name, presence: true, if: :business_account?
  validates :discount_code, presence: true, unless: -> { role == "admin" }

  # Custom validation method
  validate :password_complexity
  validate :email_domain_allowed

  private

  def password_complexity
    return unless password.present?
    unless password.match?(/[A-Z]/) && password.match?(/\d/)
      errors.add(:password, "must include at least one uppercase letter and one digit")
    end
  end

  def email_domain_allowed
    banned_domains = ["spam.com", "throwaway.net"]
    domain = email.to_s.split("@").last
    if banned_domains.include?(domain)
      errors.add(:email, "domain is not allowed")
    end
  end
end

# Using validations
user = User.new(name: "", email: "invalid")
user.valid?           # => false
user.invalid?         # => true
user.errors.full_messages  # => ["Name can't be blank", "Email is invalid"]
user.errors[:email]   # => ["is invalid"]

# save vs save!
user.save             # returns false if invalid
user.save!            # raises ActiveRecord::RecordInvalid if invalid
User.create!          # raises on invalid

# Skip validations (use with care)
user.save(validate: false)
user.update_columns(name: "Alice")  # skips validations & callbacks
```

---

## Associations

```ruby
# ── belongs_to ─────────────────────────────────────────────
class Post < ApplicationRecord
  belongs_to :user                   # requires user_id column
  belongs_to :category, optional: true  # allows nil
end

# ── has_many ────────────────────────────────────────────────
class User < ApplicationRecord
  has_many :posts                    # user.posts
  has_many :posts, dependent: :destroy   # delete posts when user is deleted
  has_many :posts, dependent: :nullify   # set user_id to nil
  has_many :active_posts, -> { where(active: true) }, class_name: "Post"
  has_many :published_posts, -> { where(published: true).order(:created_at) },
           class_name: "Post"
end

# ── has_one ─────────────────────────────────────────────────
class User < ApplicationRecord
  has_one :profile
  has_one :profile, dependent: :destroy
end

class Profile < ApplicationRecord
  belongs_to :user
end

# ── has_many :through (join model) ──────────────────────────
class Doctor < ApplicationRecord
  has_many :appointments
  has_many :patients, through: :appointments
end

class Patient < ApplicationRecord
  has_many :appointments
  has_many :doctors, through: :appointments
end

class Appointment < ApplicationRecord
  belongs_to :doctor
  belongs_to :patient
  # can have extra columns: date, notes, etc.
end

# Usage:
doctor.patients              # all patients via appointments
patient.doctors              # all doctors via appointments
doctor.appointments          # the join records with extra data

# ── has_and_belongs_to_many (no join model) ─────────────────
class Post < ApplicationRecord
  has_and_belongs_to_many :tags
end

class Tag < ApplicationRecord
  has_and_belongs_to_many :posts
end
# Requires posts_tags join table with no primary key

# ── Polymorphic associations ─────────────────────────────────
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
  # needs commentable_id and commentable_type columns
end

class Post < ApplicationRecord
  has_many :comments, as: :commentable
end

class Video < ApplicationRecord
  has_many :comments, as: :commentable
end

post.comments    # all comments on this post
video.comments   # all comments on this video

# ── Self-referential association ──────────────────────────────
class Employee < ApplicationRecord
  belongs_to :manager, class_name: "Employee", optional: true
  has_many   :subordinates, class_name: "Employee", foreign_key: :manager_id
end

ceo = Employee.create(name: "Alice")
dev = Employee.create(name: "Bob", manager: ceo)
ceo.subordinates  # => [Bob]
dev.manager       # => Alice
```

---

## Querying

```ruby
# Basic finders
User.all                          # all users
User.first                        # first user (by primary key)
User.last                         # last user
User.find(1)                      # by primary key, raises if not found
User.find([1, 2, 3])              # multiple by PK
User.find_by(email: "alice@x.com") # first match, nil if not found
User.find_by!(email: "alice@x.com") # raises if not found

# where
User.where(active: true)
User.where(role: "admin")
User.where(role: ["admin", "moderator"])  # IN clause
User.where("age > ?", 18)                 # placeholder (safe)
User.where("age BETWEEN ? AND ?", 18, 65)
User.where.not(active: false)
User.where("created_at > ?", 1.week.ago)
User.where(users: { active: true })       # qualified table name

# Order, Limit, Offset
User.order(:name)                     # ASC
User.order(name: :asc)
User.order(name: :desc, created_at: :asc)
User.limit(10)
User.offset(20)
User.limit(10).offset(20)              # page 3 of 10 per page

# Select (only load specific columns)
User.select(:id, :name, :email)
User.select("name, LOWER(email) as email_lower")

# Distinct
User.distinct
User.select(:role).distinct

# Count, sum, average, min, max
User.count
User.where(active: true).count
User.sum(:age)
User.average(:age)
User.minimum(:age)
User.maximum(:age)

# Pluck — get array of values (no Ruby objects, fast!)
User.pluck(:email)              # => ["alice@...", "bob@..."]
User.pluck(:id, :name)          # => [[1,"Alice"], [2,"Bob"]]
User.where(active: true).pluck(:id)

# ids shortcut
User.ids                        # => [1, 2, 3, ...]

# Exists?
User.exists?(email: "alice@x.com")  # => true/false

# Joining
Post.joins(:user)                    # INNER JOIN
Post.joins(:user, :comments)         # multiple joins
Post.joins(user: :profile)           # nested join
Post.left_outer_joins(:comments)     # LEFT JOIN

# Including (for eager loading — see N+1 section)
Post.includes(:user, :comments)
Post.eager_load(:user)
Post.preload(:comments)

# Group by
Order.group(:status).count           # => {"pending"=>5, "shipped"=>12}
Order.group(:status).sum(:amount)    # => {"pending"=>500, "shipped"=>1200}
Order.group("DATE(created_at)").count # group by date

# Having (filter on aggregates)
Order.group(:user_id).having("COUNT(*) > 5").count

# Calculation on associations
User.joins(:posts).group(:id).having("COUNT(posts.id) > 3")

# Subqueries
User.where(id: Post.select(:user_id).where(published: true))

# Raw SQL (escape carefully!)
User.find_by_sql("SELECT * FROM users WHERE...")
ActiveRecord::Base.connection.execute("...")
```

---

## Scopes

```ruby
class Post < ApplicationRecord
  # Named scopes
  scope :published,   -> { where(published: true) }
  scope :drafts,      -> { where(published: false) }
  scope :recent,      -> { order(created_at: :desc) }
  scope :featured,    -> { where(featured: true) }
  scope :by_user,     ->(user) { where(user: user) }
  scope :created_after, ->(date) { where("created_at > ?", date) }

  # Default scope (use with care — can cause confusion)
  default_scope { order(created_at: :desc) }

  # Combine scopes
  scope :published_and_recent, -> { published.recent }
end

# Usage
Post.published
Post.published.recent
Post.published.by_user(current_user).limit(5)
Post.created_after(1.month.ago)

# Unscoping
Post.unscoped.all                    # remove default scope
Post.published.unscope(:order)       # remove specific scope part

# Class methods work like scopes too (better for complex logic)
class Post < ApplicationRecord
  def self.trending
    joins(:views).group(:id).order("COUNT(views.id) DESC").limit(10)
  end
end
```

---

## Callbacks

```ruby
class User < ApplicationRecord
  # Lifecycle callbacks (in order)
  before_validation :normalize_email
  after_validation  :log_errors, if: :errors?

  before_save       :set_slug
  around_save       :measure_save_time  # wraps the save

  before_create     :set_confirmation_token
  after_create      :send_welcome_email

  before_update     :track_changes
  after_update      :notify_if_email_changed

  before_destroy    :check_dependencies
  after_destroy     :cleanup_files

  after_commit      :clear_cache         # runs after DB transaction commits
  after_rollback    :handle_rollback

  private

  def normalize_email
    self.email = email.to_s.strip.downcase
  end

  def set_slug
    self.slug = name.parameterize if name_changed?
  end

  def send_welcome_email
    WelcomeMailer.with(user: self).welcome_email.deliver_later
  end

  def check_dependencies
    if posts.exists?
      errors.add(:base, "Cannot delete user with posts")
      throw :abort   # MUST use throw :abort to cancel the callback
    end
  end
end

# Callback order for save:
# before_validation → (validate) → after_validation →
# before_save → before_create/update → (save to DB) →
# after_create/update → after_save → after_commit
```

---

## N+1 Problem & Eager Loading

```ruby
# ❌ N+1 Problem
posts = Post.all          # 1 query
posts.each do |post|
  puts post.user.name     # 1 query per post! = N queries
end
# If 100 posts → 101 queries!

# ✅ Solution: includes (eager loading)
posts = Post.includes(:user)  # 2 queries total
posts.each do |post|
  puts post.user.name         # no extra query
end

# Nested eager loading
posts = Post.includes(:user, comments: :author)
# Loads posts, users, comments, and comment authors in minimal queries

# includes vs joins
# includes: for accessing association data (prevents N+1)
# joins: for filtering/ordering on association columns

Post.joins(:user).where(users: { active: true })     # filter on user
Post.includes(:user).where(users: { active: true })  # WARNING — triggers LEFT JOIN + filter

# eager_load: forces LEFT OUTER JOIN (includes filter-capable)
Post.eager_load(:user).where(users: { active: true }) # ✓ correct
Post.joins(:user).includes(:user).where(users: { active: true }) # older pattern

# Detect N+1 with Bullet gem in development:
# gem 'bullet', group: :development
# config.after_initialize do
#   Bullet.enable = true
#   Bullet.alert = true
# end
```

---

## Transactions

```ruby
# Basic transaction
ActiveRecord::Base.transaction do
  user.update!(balance: user.balance - 100)
  merchant.update!(balance: merchant.balance + 100)
  # If any raises, BOTH updates are rolled back
end

# Nested transactions (savepoints in PostgreSQL)
User.transaction do
  user.save!
  begin
    Order.transaction(requires_new: true) do
      order.save!
      raise "Oops" if order.amount > user.balance
    end
  rescue ActiveRecord::RecordInvalid => e
    # order rolled back, but user is still saved
  end
end

# Manual rollback
ActiveRecord::Base.transaction do
  user.update!(credits: -1)
  raise ActiveRecord::Rollback  # silently rolls back without re-raising
end

# Handling exceptions
begin
  ActiveRecord::Base.transaction do
    transfer.execute!
  end
rescue ActiveRecord::RecordInvalid => e
  puts "Validation failed: #{e.message}"
rescue ActiveRecord::RecordNotFound => e
  puts "Record not found: #{e.message}"
end
```

---

## Counter Cache & Touch

```ruby
# Counter Cache — avoid COUNT queries on associations
class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
  # Adds automatic user.posts_count tracking
end

# Migration needed:
add_column :users, :posts_count, :integer, default: 0

# Now:
user.posts_count   # reads cached integer, NO SQL COUNT query

# Touch — update parent's updated_at when child changes
class Comment < ApplicationRecord
  belongs_to :post, touch: true
  # When a comment saves/updates, post.updated_at is updated
end
```

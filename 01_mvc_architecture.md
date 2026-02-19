# Ruby on Rails — 01: MVC Architecture

## Table of Contents
- [MVC Overview](#mvc-overview)
- [Rails App Structure](#rails-app-structure)
- [Request/Response Lifecycle](#requestresponse-lifecycle)
- [Convention Over Configuration](#convention-over-configuration)
- [Rails CLI Commands](#rails-cli-commands)

---

## MVC Overview

```
User Request
     │
     ▼
┌─────────────┐
│   Router    │  config/routes.rb
└──────┬──────┘
       │ maps URL to controller#action
       ▼
┌─────────────┐
│ Controller  │  app/controllers/
└──────┬──────┘
       │ fetches/manipulates data
       ▼
┌─────────────┐
│    Model    │  app/models/
│  (+ DB)     │  + db/schema.rb
└──────┬──────┘
       │ returns data
       ▼
┌─────────────┐
│    View     │  app/views/
└──────┬──────┘
       │
       ▼
   HTML Response
```

- **Model** — Business logic, data persistence (ActiveRecord)
- **View** — Presentation layer (ERB templates, JSON responses)
- **Controller** — Coordinator between M and V (ActionController)

---

## Rails App Structure

```
my_app/
├── app/
│   ├── controllers/
│   │   ├── application_controller.rb   # base controller
│   │   └── users_controller.rb
│   ├── models/
│   │   ├── application_record.rb       # base model
│   │   └── user.rb
│   ├── views/
│   │   ├── layouts/
│   │   │   └── application.html.erb    # master layout
│   │   └── users/
│   │       ├── index.html.erb
│   │       ├── show.html.erb
│   │       └── _user.html.erb          # partial (starts with _)
│   ├── helpers/                        # view helper methods
│   ├── jobs/                           # background jobs
│   ├── mailers/                        # emails
│   ├── channels/                       # WebSockets
│   └── assets/                         # CSS, JS, images
├── config/
│   ├── routes.rb                       # URL routing
│   ├── database.yml                    # DB config
│   ├── application.rb                  # app config
│   └── environments/
│       ├── development.rb
│       ├── test.rb
│       └── production.rb
├── db/
│   ├── migrate/                        # database migrations
│   └── schema.rb                       # current DB structure
├── spec/  (or test/)                   # tests
├── Gemfile                             # dependencies
├── Gemfile.lock
└── README.md
```

---

## Request/Response Lifecycle

### Example: `GET /users/5`

**1. Router** (`config/routes.rb`)
```ruby
Rails.application.routes.draw do
  resources :users     # generates all RESTful routes
end
```

**2. Controller** (`app/controllers/users_controller.rb`)
```ruby
class UsersController < ApplicationController
  before_action :set_user, only: [:show, :edit, :update, :destroy]
  before_action :authenticate_user!, except: [:index, :show]

  # GET /users
  def index
    @users = User.all
    # renders app/views/users/index.html.erb by default
  end

  # GET /users/:id
  def show
    # @user is set by before_action
    # renders app/views/users/show.html.erb
  end

  # GET /users/new
  def new
    @user = User.new
  end

  # POST /users
  def create
    @user = User.new(user_params)
    if @user.save
      redirect_to @user, notice: "User created!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  # GET /users/:id/edit
  def edit; end

  # PATCH/PUT /users/:id
  def update
    if @user.update(user_params)
      redirect_to @user, notice: "User updated!"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  # DELETE /users/:id
  def destroy
    @user.destroy
    redirect_to users_url, notice: "User deleted!"
  end

  private

  def set_user
    @user = User.find(params[:id])
  end

  def user_params
    params.require(:user).permit(:name, :email, :age)
  end
end
```

**3. Model** (`app/models/user.rb`)
```ruby
class User < ApplicationRecord
  validates :name, presence: true
  validates :email, presence: true, uniqueness: true
  has_many :posts
end
```

**4. View** (`app/views/users/show.html.erb`)
```erb
<h1><%= @user.name %></h1>
<p>Email: <%= @user.email %></p>
<%= link_to "Edit", edit_user_path(@user) %>
<%= link_to "Delete", @user, method: :delete, data: { confirm: "Sure?" } %>
```

---

## Convention Over Configuration

Rails uses naming conventions so you don't need to configure everything:

| Convention | Example |
|---|---|
| Model name (singular, CamelCase) | `User`, `BlogPost` |
| Table name (plural, snake_case) | `users`, `blog_posts` |
| Controller (plural, CamelCase + "Controller") | `UsersController` |
| Views folder (plural, snake_case) | `app/views/users/` |
| Helper (plural + "Helper") | `UsersHelper` |
| Primary key | `id` |
| Foreign key | `user_id` (for belongs_to :user) |
| Join table | `posts_tags` (alphabetical, plural) |

```ruby
# Rails can infer the table name:
class User < ApplicationRecord
  # automatically uses "users" table
  # automatically uses "id" as primary key
end

class BlogPost < ApplicationRecord
  # automatically uses "blog_posts" table
end

# Override if needed:
class LegacyUser < ApplicationRecord
  self.table_name = "tbl_usuarios"
  self.primary_key = "usuario_id"
end
```

---

## Rails CLI Commands

```bash
# Generate scaffolding (model + controller + views + migration)
rails generate scaffold User name:string email:string age:integer

# Generate just a model
rails generate model Post title:string body:text user:references

# Generate controller
rails generate controller Articles index show

# Generate migration
rails generate migration AddPhoneToUsers phone:string
rails generate migration CreateComments body:text user:references post:references

# Run migrations
rails db:migrate
rails db:rollback              # undo last migration
rails db:rollback STEP=3       # undo last 3 migrations
rails db:migrate:status        # see migration status

# Routes
rails routes                   # list all routes
rails routes -g users          # grep for "users"

# Console
rails console                  # interactive Rails console
rails console --sandbox        # rollback all changes on exit

# Server
rails server                   # start dev server on :3000
rails server -p 4000           # custom port

# Tests
rails test
rails spec                     # if using RSpec

# Check app for issues
rails app:update

# Destroy (opposite of generate)
rails destroy model Post
rails destroy scaffold User
```

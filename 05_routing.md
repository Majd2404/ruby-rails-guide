# Ruby on Rails — 05: Routing

## Table of Contents
- [RESTful Resources](#restful-resources)
- [Route Helpers](#route-helpers)
- [Nested Routes](#nested-routes)
- [Custom Routes](#custom-routes)
- [Constraints & Namespaces](#constraints--namespaces)
- [API Routing](#api-routing)

---

## RESTful Resources

```ruby
# config/routes.rb
Rails.application.routes.draw do

  # resources generates 7 standard routes:
  resources :posts
end
```

| Method | Path | Action | Purpose |
|---|---|---|---|
| GET | /posts | index | List all |
| GET | /posts/new | new | Show form |
| POST | /posts | create | Create |
| GET | /posts/:id | show | Show one |
| GET | /posts/:id/edit | edit | Show edit form |
| PATCH/PUT | /posts/:id | update | Update |
| DELETE | /posts/:id | destroy | Delete |

```ruby
# Limit which actions are generated
resources :photos, only: [:index, :show]
resources :tags,   except: [:destroy]

# Singular resource (no :id — current user's profile)
resource :profile           # generates: new, create, show, edit, update, destroy
resource :session, only: [:new, :create, :destroy]
```

---

## Route Helpers

```ruby
# For resources :posts, Rails generates path helpers:
posts_path              # => "/posts"
posts_url               # => "http://example.com/posts"
new_post_path           # => "/posts/new"
post_path(@post)        # => "/posts/42"
edit_post_path(@post)   # => "/posts/42/edit"

# Using in views
link_to "All Posts", posts_path
link_to "Show", post_path(@post)
link_to "Edit", edit_post_path(@post)
link_to "Delete", @post, method: :delete, data: { confirm: "Sure?" }

# Using in controllers
redirect_to posts_path
redirect_to @post              # Rails infers post_path(@post)
redirect_to [:edit, @post]     # edit_post_path(@post)

# Check all routes:
# rails routes
# rails routes -g posts   (grep)
```

---

## Nested Routes

```ruby
resources :posts do
  resources :comments    # /posts/:post_id/comments/:id
end

# Generates:
# GET    /posts/:post_id/comments          -> comments#index
# GET    /posts/:post_id/comments/new      -> comments#new
# POST   /posts/:post_id/comments          -> comments#create
# GET    /posts/:post_id/comments/:id      -> comments#show
# etc.

# In CommentsController:
class CommentsController < ApplicationController
  def index
    @post    = Post.find(params[:post_id])
    @comments = @post.comments
  end

  def create
    @post    = Post.find(params[:post_id])
    @comment = @post.comments.build(comment_params)
    if @comment.save
      redirect_to post_comments_path(@post)
    else
      render :new
    end
  end
end

# Route helper for nested:
post_comments_path(@post)           # /posts/1/comments
new_post_comment_path(@post)        # /posts/1/comments/new
post_comment_path(@post, @comment)  # /posts/1/comments/5

# Shallow nesting — avoid deep nesting (max 2 levels recommended)
resources :posts do
  resources :comments, shallow: true
end
# /posts/:post_id/comments   -> collection actions
# /comments/:id              -> member actions (show, edit, update, destroy)
```

---

## Custom Routes

```ruby
resources :posts do
  # Member route (operates on single resource)
  member do
    post   :publish      # POST /posts/:id/publish
    delete :unpublish    # DELETE /posts/:id/unpublish
    get    :preview      # GET /posts/:id/preview
  end

  # Collection route (operates on the whole collection)
  collection do
    get :search          # GET /posts/search
    get :drafts          # GET /posts/drafts
    post :bulk_delete    # POST /posts/bulk_delete
  end
end

# Shorthand
resources :posts do
  post :publish, on: :member
  get  :search,  on: :collection
end

# Custom routes
get  "/about",           to: "pages#about",    as: :about
get  "/contact",         to: "pages#contact"
root "home#index"                               # root path

# Route with dynamic segment
get "/users/:username",  to: "users#show",     as: :user_profile

# Redirect
get "/old-path", to: redirect("/new-path")
get "/blog/*path", to: redirect("/articles/%{path}")
```

---

## Constraints & Namespaces

```ruby
# Namespace — groups routes + controllers under a prefix
namespace :admin do
  resources :users     # /admin/users -> Admin::UsersController
  resources :posts
  root "dashboard#index"
end

# Scope (URL prefix only, no module)
scope path: "/api" do
  resources :users     # /api/users -> UsersController (no Admin:: module)
end

# Module (module prefix only, no URL change)
scope module: :admin do
  resources :users     # /users -> Admin::UsersController
end

# Constraints
constraints(id: /\d+/) do
  resources :posts
end

get "/users/:username", to: "users#show",
    constraints: { username: /[A-Za-z0-9_]+/ }

# Subdomain constraint
constraints subdomain: "api" do
  namespace :api do
    resources :users
  end
end

# Custom constraint class
class AuthenticatedUser
  def matches?(request)
    request.session[:user_id].present?
  end
end

constraints AuthenticatedUser.new do
  resources :dashboard
end
```

---

## API Routing

```ruby
# API versioning pattern
namespace :api do
  namespace :v1 do
    resources :users, only: [:index, :show, :create, :update, :destroy]
    resources :posts
    resources :sessions, only: [:create, :destroy]
  end

  namespace :v2 do
    resources :users
  end
end

# Generates:
# /api/v1/users          -> Api::V1::UsersController
# /api/v2/users          -> Api::V2::UsersController

# In app/controllers/api/v1/users_controller.rb:
module Api
  module V1
    class UsersController < ApplicationController
      def index
        render json: User.all
      end
    end
  end
end

# Defaults for API (respond with JSON)
namespace :api, defaults: { format: :json } do
  resources :users
end
```

# Example — Blog Application Walkthrough

> A complete example tying together all Rails concepts. Follow along to understand how they connect.

---

## Setup

```bash
rails new blog --database=postgresql
cd blog
rails db:create
```

---

## Models & Migrations

```bash
rails generate model User name:string email:string:uniq role:string bio:text
rails generate model Post title:string body:text published:boolean user:references
rails generate model Comment body:text user:references post:references
rails generate model Tag name:string:uniq
rails generate model Tagging post:references tag:references
rails db:migrate
```

### Models

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password

  has_many :posts, dependent: :destroy
  has_many :comments, dependent: :destroy
  has_many :published_posts, -> { published }, class_name: "Post"

  validates :name,  presence: true, length: { 2..50 }
  validates :email, presence: true, uniqueness: { case_sensitive: false },
                    format: { with: URI::MailTo::EMAIL_REGEXP }

  normalizes :email, with: ->(e) { e.strip.downcase }

  enum role: { user: "user", moderator: "moderator", admin: "admin" }

  scope :active,  -> { where(active: true) }
  scope :authors, -> { joins(:posts).distinct }

  def display_name
    name.presence || email.split("@").first
  end
end

# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user
  has_many :comments, dependent: :destroy
  has_many :taggings, dependent: :destroy
  has_many :tags, through: :taggings

  validates :title, presence: true, length: { maximum: 200 }
  validates :body,  presence: true

  scope :published,  -> { where(published: true) }
  scope :drafts,     -> { where(published: false) }
  scope :recent,     -> { order(created_at: :desc) }
  scope :featured,   -> { published.where(featured: true) }
  scope :by_tag,     ->(tag) { joins(:tags).where(tags: { name: tag }) }

  before_save :generate_slug

  def to_param
    slug  # use slug in URLs instead of id
  end

  def reading_time
    words = body.split.count
    (words / 200.0).ceil  # assuming 200 WPM
  end

  private

  def generate_slug
    self.slug = title.parameterize if title_changed?
  end
end

# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :user
  belongs_to :post, counter_cache: true   # post.comments_count

  validates :body, presence: true, length: { minimum: 2, maximum: 1000 }

  scope :recent, -> { order(created_at: :desc) }
end
```

---

## Controllers

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  before_action :authorize_post!, only: [:edit, :update, :destroy]

  # GET /posts
  def index
    @posts = Post.published
                 .includes(:user, :tags)
                 .recent
                 .page(params[:page]).per(10)

    @posts = @posts.by_tag(params[:tag]) if params[:tag].present?
  end

  # GET /posts/:slug
  def show
    @comments = @post.comments.includes(:user).recent
    @comment  = Comment.new
  end

  # GET /posts/new
  def new
    @post = Post.new
  end

  # POST /posts
  def create
    @post = current_user.posts.build(post_params)
    if @post.save
      redirect_to @post, notice: "Post created!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  # GET /posts/:slug/edit
  def edit; end

  # PATCH /posts/:slug
  def update
    if @post.update(post_params)
      redirect_to @post, notice: "Post updated!"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  # DELETE /posts/:slug
  def destroy
    @post.destroy
    redirect_to posts_path, notice: "Post deleted!"
  end

  private

  def set_post
    @post = Post.find_by!(slug: params[:id])
  end

  def authorize_post!
    unless @post.user == current_user || current_user.admin?
      redirect_to @post, alert: "Not authorized"
    end
  end

  def post_params
    params.require(:post).permit(:title, :body, :published, tag_ids: [])
  end
end
```

---

## Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "posts#index"

  resources :posts do
    resources :comments, only: [:create, :destroy], shallow: true
    member do
      post :publish
      post :unpublish
    end
    collection do
      get :drafts
    end
  end

  resources :tags, only: [:index, :show]

  namespace :admin do
    resources :users
    resources :posts
    root "dashboard#index"
  end

  get "/login",  to: "sessions#new",    as: :login
  post "/login", to: "sessions#create"
  delete "/logout", to: "sessions#destroy", as: :logout
end
```

---

## Views

```erb
<!-- app/views/posts/index.html.erb -->
<h1>Blog Posts</h1>

<% if params[:tag] %>
  <p>Showing posts tagged: <strong><%= params[:tag] %></strong></p>
<% end %>

<div class="posts">
  <%= render @posts %>
</div>

<%= paginate @posts %>

<!-- app/views/posts/_post.html.erb -->
<article class="post">
  <h2><%= link_to post.title, post %></h2>
  <p class="meta">
    By <%= post.user.display_name %> ·
    <%= post.reading_time %> min read ·
    <%= time_ago_in_words(post.created_at) %> ago
  </p>
  <p><%= truncate(post.body, length: 200) %></p>
  <div class="tags">
    <% post.tags.each do |tag| %>
      <%= link_to tag.name, posts_path(tag: tag.name), class: "tag" %>
    <% end %>
  </div>
</article>

<!-- app/views/posts/show.html.erb -->
<article>
  <h1><%= @post.title %></h1>
  <p class="meta">By <%= @post.user.display_name %></p>
  <div class="body"><%= @post.body %></div>
</article>

<section class="comments">
  <h2>Comments (<%= @post.comments_count %>)</h2>

  <%= render @comments %>

  <% if user_signed_in? %>
    <%= render "comments/form", post: @post, comment: @comment %>
  <% else %>
    <%= link_to "Sign in to comment", login_path %>
  <% end %>
</section>
```

---

## Testing

```ruby
# spec/models/post_spec.rb
RSpec.describe Post, type: :model do
  let(:user) { create(:user) }
  let(:post) { build(:post, user: user) }

  describe "validations" do
    it { is_expected.to validate_presence_of(:title) }
    it { is_expected.to validate_presence_of(:body) }
    it { is_expected.to belong_to(:user) }
    it { is_expected.to have_many(:comments).dependent(:destroy) }
  end

  describe "scopes" do
    describe ".published" do
      let!(:published) { create(:post, :published, user: user) }
      let!(:draft)     { create(:post, :draft,     user: user) }

      it "returns only published posts" do
        expect(Post.published).to include(published)
        expect(Post.published).not_to include(draft)
      end
    end
  end

  describe "#reading_time" do
    it "calculates based on word count" do
      post = build(:post, body: "word " * 400)  # 400 words
      expect(post.reading_time).to eq(2)         # 2 minutes
    end
  end
end

# spec/requests/posts_spec.rb
RSpec.describe "Posts", type: :request do
  let(:user) { create(:user) }
  let(:post) { create(:post, :published, user: user) }

  describe "GET /posts" do
    it "returns 200" do
      get posts_path
      expect(response).to have_http_status(:ok)
    end
  end

  describe "POST /posts" do
    context "when authenticated" do
      before { sign_in user }

      it "creates post with valid params" do
        expect {
          post posts_path, params: { post: attributes_for(:post) }
        }.to change(Post, :count).by(1)
      end
    end

    context "when not authenticated" do
      it "redirects to login" do
        post posts_path, params: { post: attributes_for(:post) }
        expect(response).to redirect_to(login_path)
      end
    end
  end
end
```

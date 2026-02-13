# 19. Performance & Optimization

## Table of Contents

- [N+1 Queries](#n1-queries)
- [Eager Loading](#eager-loading)
- [Caching Strategies](#caching-strategies)
- [Database Optimization](#database-optimization)
- [Memory Management](#memory-management)
- [Background Processing](#background-processing)
- [Frontend Performance](#frontend-performance)
- [Profiling & Benchmarking](#profiling--benchmarking)
- [Scaling Patterns](#scaling-patterns)

---

## N+1 Queries

The most common Rails performance problem. Occurs when you load a collection and then access an association on each item, triggering a separate query per item.

### The Problem

```ruby
# BAD — N+1: 1 query for posts + N queries for authors
@posts = Post.all
@posts.each do |post|
  puts post.author.name  # Each call triggers a new query
end

# Generated SQL:
# SELECT "posts".* FROM "posts"
# SELECT "authors".* FROM "authors" WHERE "authors"."id" = 1
# SELECT "authors".* FROM "authors" WHERE "authors"."id" = 2
# SELECT "authors".* FROM "authors" WHERE "authors"."id" = 3
# ... one per post
```

### The Fix

```ruby
# GOOD — Eager load with includes
@posts = Post.includes(:author)
@posts.each do |post|
  puts post.author.name  # No additional queries
end

# Generated SQL:
# SELECT "posts".* FROM "posts"
# SELECT "authors".* FROM "authors" WHERE "authors"."id" IN (1, 2, 3, ...)
```

### Detection Tools

```ruby
# Gemfile — use Bullet gem to detect N+1 in development
group :development do
  gem "bullet"
end

# config/environments/development.rb
config.after_initialize do
  Bullet.enable        = true
  Bullet.alert         = true        # JavaScript popup
  Bullet.bullet_logger = true        # log/bullet.log
  Bullet.console       = true        # Browser console
  Bullet.rails_logger  = true        # Rails log
  Bullet.add_footer    = true        # Bottom of page
end

# Strict mode in test — raise on N+1
# config/environments/test.rb
config.after_initialize do
  Bullet.enable        = true
  Bullet.raise         = true  # Fail tests on N+1
end
```

### Use `strict_loading` to Prevent N+1

```ruby
# Rails 6.1+ — raise error if association not eager loaded

# Per-query strict loading
posts = Post.strict_loading.all
posts.first.author  # Raises ActiveRecord::StrictLoadingViolationError

# Per-model strict loading (all queries)
class Post < ApplicationRecord
  self.strict_loading_by_default = true
end

# Per-association strict loading
class Post < ApplicationRecord
  has_many :comments, strict_loading: true
end

# Disable strict loading when you've eager loaded
posts = Post.strict_loading.includes(:author)
posts.first.author  # Works fine — already loaded
```

---

## Eager Loading

### Three Methods: `includes`, `preload`, `eager_load`

```ruby
# includes — Rails decides strategy (preload or eager_load)
# Use this by default
Post.includes(:author, :comments)

# preload — Always separate queries (cannot filter on association)
Post.preload(:author)
# SELECT "posts".* FROM "posts"
# SELECT "authors".* FROM "authors" WHERE "authors"."id" IN (1, 2, 3)

# eager_load — Always LEFT OUTER JOIN (can filter on association)
Post.eager_load(:author).where(authors: { active: true })
# SELECT "posts".*, "authors".* FROM "posts"
#   LEFT OUTER JOIN "authors" ON "authors"."id" = "posts"."author_id"
#   WHERE "authors"."active" = TRUE
```

### When to Use Each

```ruby
# includes — default choice, Rails optimizes automatically
Post.includes(:author)

# includes with where on association — Rails auto-switches to JOIN
Post.includes(:author).where(authors: { verified: true })

# preload — when you need separate queries (avoids large JOINs)
Post.preload(:comments)  # Good when comments are many

# eager_load — when you MUST filter/order by association columns
Post.eager_load(:author).order("authors.name ASC")
```

### Nested Eager Loading

```ruby
# Load nested associations
Post.includes(comments: :author)
Post.includes(comments: { author: :profile })

# Multiple nested associations
Post.includes(:author, comments: [:author, :likes])

# Deep nesting
Post.includes(
  :author,
  :tags,
  comments: {
    author: :profile,
    replies: :author
  }
)
```

### Conditional Eager Loading

```ruby
# Only eager load what you need
class PostsController < ApplicationController
  def index
    @posts = Post.includes(:author)  # List needs author name
  end

  def show
    @post = Post.includes(:author, comments: :author)  # Show needs everything
  end
end
```

### Counter Caches

Avoid `COUNT(*)` queries by caching the count in the parent.

```ruby
# Migration
class AddCommentsCountToPosts < ActiveRecord::Migration[7.1]
  def change
    add_column :posts, :comments_count, :integer, default: 0, null: false

    # Backfill existing counts
    reversible do |dir|
      dir.up do
        Post.find_each do |post|
          Post.reset_counters(post.id, :comments)
        end
      end
    end
  end
end

# Model
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end

# Now post.comments_count is a column read, not a COUNT query
# post.comments.size uses counter cache automatically
```

### Custom Counter Cache Column

```ruby
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: :total_comments
end

# Migration must add :total_comments column to posts
```

---

## Caching Strategies

### Fragment Caching (View Caching)

```erb
<%# Cache a partial based on the record's cache_key_with_version %>
<% @posts.each do |post| %>
  <% cache post do %>
    <%= render post %>
  <% end %>
<% end %>

<%# Cache with explicit key %>
<% cache ["v1", post, current_user.admin?] do %>
  <%= render post %>
<% end %>

<%# Collection caching — much faster, single cache read %>
<%= render partial: "posts/post", collection: @posts, cached: true %>
```

### Russian Doll Caching

Nested cache fragments that auto-expire from inner to outer.

```ruby
# Model — ensure touch propagates
class Comment < ApplicationRecord
  belongs_to :post, touch: true  # Updates post.updated_at when comment changes
end

class Post < ApplicationRecord
  belongs_to :blog, touch: true
end
```

```erb
<%# Outer cache expires when post or any comment changes %>
<% cache post do %>
  <h2><%= post.title %></h2>

  <% post.comments.each do |comment| %>
    <%# Inner cache expires only when this comment changes %>
    <% cache comment do %>
      <%= render comment %>
    <% end %>
  <% end %>
<% end %>
```

### Low-Level Caching (`Rails.cache`)

```ruby
# Read/write cache with expiration
class Product < ApplicationRecord
  def self.trending
    Rails.cache.fetch("products/trending", expires_in: 1.hour) do
      where("created_at > ?", 1.week.ago)
        .order(views_count: :desc)
        .limit(10)
        .to_a  # Important: materialize the query
    end
  end
end

# Fetch with race condition TTL (prevents thundering herd)
Rails.cache.fetch("popular_posts", expires_in: 1.hour, race_condition_ttl: 10.seconds) do
  Post.popular.to_a
end

# Manual cache operations
Rails.cache.write("key", value, expires_in: 30.minutes)
Rails.cache.read("key")
Rails.cache.delete("key")
Rails.cache.exist?("key")

# Increment/decrement (atomic with Memcached/Redis)
Rails.cache.increment("page_views:#{post.id}")
Rails.cache.decrement("remaining_credits:#{user.id}")
```

### Cache Store Configuration

```ruby
# config/environments/production.rb

# Solid Cache — Rails 8 default (database-backed)
config.cache_store = :solid_cache_store

# Redis
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  expires_in: 1.day,
  namespace: "myapp",
  pool_size: ENV.fetch("RAILS_MAX_THREADS", 5).to_i,
  error_handler: ->(method:, returning:, exception:) {
    Rails.logger.error("Redis error: #{exception}")
    Sentry.capture_exception(exception)
  }
}

# Memcached
config.cache_store = :mem_cache_store, ENV["MEMCACHED_URL"]

# Memory store (development/test only)
config.cache_store = :memory_store, { size: 64.megabytes }

# File store (simple deployments)
config.cache_store = :file_store, Rails.root.join("tmp/cache")
```

### HTTP Caching

```ruby
class PostsController < ApplicationController
  # Conditional GET — returns 304 Not Modified if unchanged
  def show
    @post = Post.find(params[:id])
    fresh_when(@post)  # Uses ETag + Last-Modified
  end

  # With explicit rendering
  def show
    @post = Post.find(params[:id])
    if stale?(@post)
      respond_to do |format|
        format.html
        format.json { render json: @post }
      end
    end
  end

  # Cache-Control headers
  def index
    @posts = Post.published
    expires_in 5.minutes, public: true  # CDN-cacheable
  end

  # Stale with collection
  def index
    @posts = Post.published
    fresh_when(@posts)
  end
end
```

### Query Caching

```ruby
# Rails automatically caches identical SQL queries within a single request
# No configuration needed — it's on by default in controllers

# Manual query cache in other contexts
ActiveRecord::Base.cache do
  Post.find(1)  # Hits database
  Post.find(1)  # Returns cached result
end
```

### Cache Key Design

```ruby
# ActiveRecord cache keys
post.cache_key                    # "posts/1"
post.cache_key_with_version       # "posts/1-20240115123456789"
Post.all.cache_key_with_version   # Based on count + max updated_at

# Custom cache keys
def dashboard_cache_key
  [
    "dashboard",
    current_user.id,
    current_user.updated_at.to_i,
    Post.maximum(:updated_at).to_i,
    Date.current.to_s
  ].join("/")
end
```

---

## Database Optimization

### Indexing Best Practices

```ruby
# Index columns used in WHERE, ORDER, JOIN
class AddIndexes < ActiveRecord::Migration[7.1]
  def change
    # Foreign keys — ALWAYS index
    add_index :comments, :post_id
    add_index :comments, :author_id

    # Columns frequently queried
    add_index :users, :email, unique: true
    add_index :posts, :published_at
    add_index :posts, :status

    # Composite index — column order matters!
    # Supports queries on (user_id), (user_id, created_at)
    # Does NOT support queries on (created_at) alone
    add_index :posts, [:user_id, :created_at]

    # Partial index — index only subset of rows
    add_index :posts, :published_at, where: "status = 'published'",
              name: "index_posts_on_published_at_when_published"

    # GIN index for array/jsonb columns (PostgreSQL)
    add_index :posts, :tags, using: :gin
    add_index :products, :metadata, using: :gin

    # Expression index
    add_index :users, "lower(email)", unique: true,
              name: "index_users_on_lower_email"
  end
end
```

### SELECT Only What You Need

```ruby
# BAD — loads all columns
users = User.all

# GOOD — select only needed columns
users = User.select(:id, :name, :email)

# GOOD — pluck for simple values (returns arrays, not AR objects)
emails = User.where(active: true).pluck(:email)
# => ["alice@example.com", "bob@example.com"]

# GOOD — ids shortcut
user_ids = User.active.ids
# => [1, 2, 3]

# GOOD — pick for single value from single row
User.where(email: "alice@example.com").pick(:id)
# => 1
```

### Batch Processing

```ruby
# BAD — loads entire table into memory
User.all.each { |user| user.update(processed: true) }

# GOOD — process in batches of 1000 (default)
User.find_each do |user|
  user.update(processed: true)
end

# Custom batch size
User.find_each(batch_size: 500) do |user|
  UserMailer.weekly_digest(user).deliver_later
end

# find_in_batches — yields arrays instead of individual records
User.find_in_batches(batch_size: 1000) do |batch|
  SomeExternalService.bulk_update(batch.map(&:email))
end

# in_batches — yields ActiveRecord::Relation (supports bulk operations)
User.in_batches(of: 1000) do |batch|
  batch.update_all(newsletter: true)  # Single UPDATE per batch
end

# With order and cursor-based pagination (Rails 7.1+)
Post.in_batches(cursor: [:created_at, :id]) do |batch|
  batch.update_all(archived: true)
end
```

### Bulk Operations

```ruby
# BAD — N individual INSERT statements
users.each { |attrs| User.create(attrs) }

# GOOD — single INSERT statement (Rails 6+)
User.insert_all([
  { name: "Alice", email: "alice@example.com", created_at: Time.current, updated_at: Time.current },
  { name: "Bob", email: "bob@example.com", created_at: Time.current, updated_at: Time.current }
])

# insert_all! — raises on duplicate
# upsert_all — insert or update on conflict
User.upsert_all(
  [{ email: "alice@example.com", name: "Alice Updated" }],
  unique_by: :email
)

# BAD — N individual UPDATE statements
posts.each { |post| post.update(status: "archived") }

# GOOD — single UPDATE statement
Post.where("created_at < ?", 1.year.ago).update_all(status: "archived")

# BAD — N individual DELETE statements
old_posts.each(&:destroy)

# GOOD — single DELETE statement (skips callbacks)
Post.where("created_at < ?", 1.year.ago).delete_all

# GOOD — single DELETE with callbacks
Post.where("created_at < ?", 1.year.ago).destroy_all  # Still loads records
```

### Database-Level Optimizations

```ruby
# Use database functions instead of Ruby
# BAD
User.all.select { |u| u.name.downcase == "alice" }

# GOOD
User.where("LOWER(name) = ?", "alice")

# BAD — sorting in Ruby
User.all.sort_by(&:created_at)

# GOOD — sorting in database
User.order(:created_at)

# BAD — counting in Ruby
User.all.length

# GOOD — counting in database
User.count

# Use EXISTS instead of loading records
# BAD
User.where(admin: true).any?  # Loads records, then checks

# GOOD
User.where(admin: true).exists?  # SELECT 1 ... LIMIT 1

# Use calculations in database
User.average(:age)
User.sum(:balance)
User.minimum(:created_at)
User.maximum(:updated_at)
Order.group(:status).count
```

### EXPLAIN and Query Analysis

```ruby
# Analyze query plan
puts Post.where(status: "published").explain

# In Rails console
Post.where(status: "published").explain(:analyze)

# PostgreSQL specific
Post.where(status: "published").explain(:analyze, :buffers, :verbose)
```

### Connection Pool Configuration

```ruby
# config/database.yml
production:
  adapter: postgresql
  pool: <%= ENV.fetch("RAILS_MAX_THREADS", 5) %>
  timeout: 5000
  prepared_statements: true    # Reuse query plans
  advisory_locks: true         # For migrations

# For Puma — pool should match threads
# config/puma.rb
threads_count = ENV.fetch("RAILS_MAX_THREADS", 5)
threads threads_count, threads_count
```

---

## Memory Management

### Avoid Loading Large Datasets

```ruby
# BAD — loads all records into memory
Post.all.map(&:title)

# GOOD — pluck returns raw values
Post.pluck(:title)

# BAD — builds large array in memory
results = []
User.find_each { |user| results << user.serialize }

# GOOD — stream results
User.find_each do |user|
  yield user.serialize
end
```

### String Frozen Literals

```ruby
# Freeze string literals to reduce object allocation
# Add to top of every Ruby file:
# frozen_string_literal: true

# Or configure globally in .rubocop.yml
Style/FrozenStringLiteralComment:
  EnforcedStyle: always
```

### Symbol vs String Keys

```ruby
# Prefer symbols for hash keys (shared in memory)
# BAD
{ "name" => "Alice", "age" => 30 }

# GOOD
{ name: "Alice", age: 30 }
```

### Garbage Collection Tuning

```ruby
# Environment variables for GC tuning (set in deployment)
# These are starting points — tune based on profiling

# RUBY_GC_HEAP_INIT_SLOTS=600000
# RUBY_GC_HEAP_FREE_SLOTS_MIN_RATIO=0.20
# RUBY_GC_HEAP_FREE_SLOTS_GOAL_RATIO=0.40
# RUBY_GC_HEAP_FREE_SLOTS_MAX_RATIO=0.65

# Jemalloc — better memory allocator
# In Dockerfile:
# RUN apt-get install -y libjemalloc2
# ENV LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2
# ENV MALLOC_CONF="dirty_decay_ms:1000,narenas:2"
```

---

## Background Processing

### Move Slow Work Out of Requests

```ruby
# BAD — slow operations in request cycle
class OrdersController < ApplicationController
  def create
    @order = Order.create!(order_params)
    OrderMailer.confirmation(@order).deliver_now    # Slow!
    PdfGenerator.generate_invoice(@order)           # Slow!
    AnalyticsService.track("order_created", @order) # Slow!
    redirect_to @order
  end
end

# GOOD — offload to background jobs
class OrdersController < ApplicationController
  def create
    @order = Order.create!(order_params)
    OrderMailer.confirmation(@order).deliver_later
    GenerateInvoiceJob.perform_later(@order)
    TrackAnalyticsJob.perform_later("order_created", @order.id)
    redirect_to @order
  end
end
```

### Debounce Expensive Operations

```ruby
class UpdateSearchIndexJob < ApplicationJob
  # Debounce: only run once even if enqueued multiple times
  def perform(post_id)
    post = Post.find(post_id)
    SearchIndex.update(post)
  end
end

# Enqueue with deduplication (Solid Queue / Sidekiq)
# Use unique jobs or check before enqueuing
```

---

## Frontend Performance

### Turbo Performance

```ruby
# Use Turbo Frames for partial page updates
# Instead of full page reloads, only update the frame

# Lazy-load frames
<%= turbo_frame_tag "comments", src: post_comments_path(@post), loading: :lazy do %>
  <p>Loading comments...</p>
<% end %>

# Prefetch links (Turbo Drive)
<%= link_to "Post", post_path(@post), data: { turbo_prefetch: true } %>
```

### Asset Optimization

```ruby
# Propshaft — assets are fingerprinted by default
# Use asset helpers in views
<%= image_tag "logo.png" %>
<%= stylesheet_link_tag "application" %>
<%= javascript_importmap_tags %>

# Preload critical assets
<%= preload_link_tag "application.css" %>

# Lazy load images
<%= image_tag "hero.jpg", loading: :lazy %>
```

### Pagination

```ruby
# Never load unbounded collections
# Use pagy gem (fastest pagination)

# Gemfile
gem "pagy"

# Controller
class PostsController < ApplicationController
  include Pagy::Backend

  def index
    @pagy, @posts = pagy(Post.published.order(created_at: :desc), items: 25)
  end
end

# View
<%# app/views/posts/index.html.erb %>
<%= render @posts %>
<%== pagy_nav(@pagy) %>

# API controller
class Api::V1::PostsController < ApiController
  include Pagy::Backend

  def index
    @pagy, @posts = pagy(Post.published)
    render json: {
      data: @posts,
      pagination: pagy_metadata(@pagy)
    }
  end
end
```

---

## Profiling & Benchmarking

### rack-mini-profiler

```ruby
# Gemfile
group :development do
  gem "rack-mini-profiler"
  gem "memory_profiler"       # For memory profiling
  gem "stackprof"             # For CPU profiling
end

# Shows timing badge on every page in development
# Append ?pp=help to any URL for profiling options
# ?pp=profile-memory — memory allocation report
# ?pp=profile-gc — GC stats
# ?pp=flamegraph — CPU flamegraph
```

### Benchmark in Code

```ruby
require "benchmark"

# Simple timing
Benchmark.measure { Post.where(status: "published").to_a }

# Compare approaches
Benchmark.bm do |x|
  x.report("includes") { Post.includes(:author).to_a }
  x.report("joins")    { Post.joins(:author).to_a }
  x.report("preload")  { Post.preload(:author).to_a }
end

# IPS (iterations per second) — more accurate
require "benchmark/ips"

Benchmark.ips do |x|
  x.report("map")   { users.map(&:name) }
  x.report("pluck") { User.pluck(:name) }
  x.compare!
end
```

### Log Slow Queries

```ruby
# config/initializers/slow_query_logger.rb
ActiveSupport::Notifications.subscribe("sql.active_record") do |*, payload|
  duration = payload[:duration]
  if duration > 100  # milliseconds
    Rails.logger.warn(
      "SLOW QUERY (#{duration.round(1)}ms): #{payload[:sql]}"
    )
  end
end

# Or use database-level logging
# PostgreSQL: log_min_duration_statement = 100
```

### Application Performance Monitoring (APM)

```ruby
# Production monitoring services
# Choose one:
gem "scout_apm"       # Scout APM
gem "newrelic_rpm"    # New Relic
gem "ddtrace"         # Datadog
gem "sentry-rails"    # Sentry (errors + performance)

# Sentry setup example
# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = ENV["SENTRY_DSN"]
  config.traces_sample_rate = 0.1  # 10% of transactions
  config.profiles_sample_rate = 0.1
end
```

---

## Scaling Patterns

### Database Read Replicas (Rails 6+)

```ruby
# config/database.yml
production:
  primary:
    <<: *default
    host: primary-db.example.com
  primary_replica:
    <<: *default
    host: replica-db.example.com
    replica: true

# Automatic read/write splitting
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
  connects_to database: { writing: :primary, reading: :primary_replica }
end

# Rails automatically routes:
# - GET requests → reading (replica)
# - POST/PUT/PATCH/DELETE → writing (primary)

# Manual override
ActiveRecord::Base.connected_to(role: :reading) do
  Post.first  # Uses replica
end

ActiveRecord::Base.connected_to(role: :writing) do
  Post.create!(title: "New")  # Uses primary
end
```

### Sharding (Rails 7+)

```ruby
# config/database.yml
production:
  primary:
    <<: *default
    host: primary-db.example.com
  primary_shard_one:
    <<: *default
    host: shard1-db.example.com
  primary_shard_two:
    <<: *default
    host: shard2-db.example.com

class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
  connects_to shards: {
    default: { writing: :primary },
    shard_one: { writing: :primary_shard_one },
    shard_two: { writing: :primary_shard_two }
  }
end

# Route to shard
ActiveRecord::Base.connected_to(shard: :shard_one) do
  User.find(123)
end
```

### CDN and Static Assets

```ruby
# config/environments/production.rb
config.action_controller.asset_host = ENV["CDN_HOST"]
# e.g., "https://d1234567890.cloudfront.net"

# All asset helpers automatically use CDN
<%= image_tag "logo.png" %>
# => <img src="https://d1234567890.cloudfront.net/assets/logo-abc123.png" />
```

### Rate Limiting (Rails 8+)

```ruby
class PostsController < ApplicationController
  # Built-in rate limiting
  rate_limit to: 10, within: 1.minute, only: :create

  # With custom response
  rate_limit to: 100, within: 1.hour,
             by: -> { request.remote_ip },
             with: -> { render json: { error: "Rate limited" }, status: :too_many_requests }
end
```

---

## Performance Checklist

### Before Every Deploy

- [ ] Run `bullet` in development — no N+1 queries
- [ ] Check `rack-mini-profiler` — no slow pages
- [ ] Database indexes exist for all foreign keys
- [ ] Database indexes exist for all `WHERE` / `ORDER` columns
- [ ] No unbounded queries (pagination on all collections)
- [ ] Background jobs for slow operations (email, PDF, API calls)
- [ ] Fragment caching on expensive view partials
- [ ] `counter_cache` for frequently counted associations

### Database

- [ ] Use `includes` / `preload` for all association access in loops
- [ ] Use `select` / `pluck` when you don't need full AR objects
- [ ] Use `find_each` / `in_batches` for large dataset processing
- [ ] Use `insert_all` / `upsert_all` for bulk writes
- [ ] Use `update_all` / `delete_all` for bulk mutations
- [ ] Use database calculations (`count`, `sum`, `average`) over Ruby
- [ ] Use `exists?` instead of `any?` / `present?` for existence checks

### Caching

- [ ] Fragment cache expensive partials with `cache` helper
- [ ] Collection caching with `cached: true`
- [ ] Russian doll caching with `touch: true` on associations
- [ ] Low-level caching for expensive computations
- [ ] HTTP caching headers (`fresh_when`, `stale?`, `expires_in`)
- [ ] Cache store configured for production (Redis / Solid Cache)

### Frontend

- [ ] Lazy-load below-the-fold content (Turbo Frames, `loading: :lazy`)
- [ ] Asset fingerprinting (automatic with Propshaft)
- [ ] CDN for static assets
- [ ] Preload critical assets
- [ ] Pagination on all list views

### Monitoring

- [ ] APM tool in production (Scout, New Relic, Datadog, or Sentry)
- [ ] Slow query logging enabled
- [ ] Error tracking configured
- [ ] Memory usage monitored
- [ ] Background job queue monitored

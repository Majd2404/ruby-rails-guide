# Ruby Fundamentals — 06 & 07: Concurrency & Design Patterns

---

## Concurrency

### Threads

```ruby
# Basic thread
thread = Thread.new do
  puts "Running in background"
  sleep(1)
  puts "Done"
end

thread.join   # wait for thread to finish

# Thread with return value
t = Thread.new { expensive_calculation() }
result = t.value   # blocks until done, returns result

# Thread safety: Mutex
counter = 0
mutex   = Mutex.new

threads = 10.times.map do
  Thread.new do
    1000.times do
      mutex.synchronize { counter += 1 }
    end
  end
end
threads.each(&:join)
puts counter  # => 10000 (safe with mutex)

# GIL (Global Interpreter Lock) in MRI Ruby:
# Only one thread runs Ruby code at a time
# BUT: I/O operations release the GIL
# So threads ARE useful for I/O-bound tasks (HTTP requests, DB queries)
# For CPU-bound tasks, use Ractors or multiple processes

# Parallel HTTP requests with threads
require 'net/http'

urls = ["https://example.com", "https://api.github.com", "https://ruby-lang.org"]
results = urls.map do |url|
  Thread.new { Net::HTTP.get(URI(url)) }
end.map(&:value)
```

### Fibers

```ruby
# Fibers are cooperative (manually yield control), not preemptive
fiber = Fiber.new do
  puts "Step 1"
  Fiber.yield   # pause, return control to caller
  puts "Step 2"
  Fiber.yield
  puts "Step 3"
end

fiber.resume  # "Step 1"
fiber.resume  # "Step 2"
fiber.resume  # "Step 3"

# Fiber with values
producer = Fiber.new do
  5.times do |i|
    Fiber.yield i * 2
  end
  nil  # signal done
end

while (value = producer.resume)
  puts value  # 0, 2, 4, 6, 8
end

# Custom Enumerator using Fiber
def infinite_counter(start = 0)
  Enumerator.new do |yielder|
    n = start
    loop { yielder << n; n += 1 }
  end
end

counter = infinite_counter(10)
counter.first(5)  # => [10, 11, 12, 13, 14]
```

---

## Design Patterns

### Singleton

```ruby
require 'singleton'

class Configuration
  include Singleton

  attr_accessor :database_url, :redis_url, :log_level

  def initialize
    @database_url = ENV["DATABASE_URL"]
    @redis_url    = ENV["REDIS_URL"]
    @log_level    = :info
  end
end

# Only one instance ever created
config = Configuration.instance
config.log_level = :debug
Configuration.instance.log_level  # => :debug  (same object)
Configuration.new   # NoMethodError — new is private
```

### Builder Pattern

```ruby
class QueryBuilder
  def initialize
    @conditions = []
    @order      = nil
    @limit      = nil
    @includes   = []
  end

  def where(**conditions)
    @conditions << conditions
    self  # return self for chaining
  end

  def order(field, direction = :asc)
    @order = { field => direction }
    self
  end

  def limit(n)
    @limit = n
    self
  end

  def include(*associations)
    @includes += associations
    self
  end

  def build
    scope = User.all
    @conditions.each { |cond| scope = scope.where(cond) }
    scope = scope.order(@order) if @order
    scope = scope.limit(@limit) if @limit
    scope = scope.includes(@includes) unless @includes.empty?
    scope
  end
end

# Usage
users = QueryBuilder.new
          .where(active: true)
          .where(role: :user)
          .order(:name)
          .limit(20)
          .include(:posts)
          .build
```

### Command Pattern

```ruby
# Encapsulate a request as an object
# Enables undo, queuing, logging

class Command
  def execute; raise NotImplementedError; end
  def undo;    raise NotImplementedError; end
end

class CreatePostCommand < Command
  attr_reader :post

  def initialize(user, params)
    @user   = user
    @params = params
  end

  def execute
    @post = @user.posts.create!(@params)
  end

  def undo
    @post&.destroy
  end
end

class PublishPostCommand < Command
  def initialize(post)
    @post      = post
    @was_published = post.published
  end

  def execute
    @post.update!(published: true)
  end

  def undo
    @post.update!(published: @was_published)
  end
end

# Command history for undo
class CommandInvoker
  def initialize
    @history = []
  end

  def execute(command)
    command.execute
    @history << command
  end

  def undo_last
    command = @history.pop
    command&.undo
  end
end

invoker = CommandInvoker.new
cmd = CreatePostCommand.new(user, title: "Hello", body: "World")
invoker.execute(cmd)
invoker.undo_last  # destroys the post
```

### Strategy Pattern

```ruby
# Encapsulate interchangeable algorithms

class TextExporter
  def initialize(strategy)
    @strategy = strategy
  end

  def export(data)
    @strategy.call(data)
  end
end

json_strategy = ->(data) { JSON.generate(data) }
csv_strategy  = ->(data) {
  headers = data.first.keys.join(",")
  rows    = data.map { |row| row.values.join(",") }
  ([headers] + rows).join("\n")
}
xml_strategy = ->(data) {
  Nokogiri::XML::Builder.new { |xml| xml.root { data.each { |row| xml.item(row) } } }.to_xml
}

exporter = TextExporter.new(json_strategy)
exporter.export([{ name: "Alice", age: 30 }])

exporter = TextExporter.new(csv_strategy)
exporter.export([{ name: "Alice", age: 30 }])
```

### Template Method Pattern

```ruby
# Define skeleton algorithm in base class, subclasses fill in steps

class DataImporter
  def import(file_path)
    data = read_file(file_path)        # step 1
    data = validate(data)              # step 2
    data = transform(data)             # step 3
    save(data)                         # step 4
    notify_completion(data.count)      # step 5
  end

  private

  def read_file(path)
    raise NotImplementedError
  end

  def validate(data)
    data  # default: no validation
  end

  def transform(data)
    data  # default: no transformation
  end

  def save(data)
    raise NotImplementedError
  end

  def notify_completion(count)
    puts "Imported #{count} records"
  end
end

class CsvUserImporter < DataImporter
  private

  def read_file(path)
    CSV.foreach(path, headers: true).map(&:to_h)
  end

  def validate(data)
    data.select { |row| row["email"].present? }
  end

  def transform(data)
    data.map { |row| row.merge("email" => row["email"].downcase) }
  end

  def save(data)
    User.insert_all(data)
  end
end
```

### Observer / Pub-Sub Pattern

```ruby
# Using ActiveSupport::Notifications (Rails built-in)
# Publishing events
ActiveSupport::Notifications.instrument("user.created", user: user) do
  user.save!
end

# Subscribing to events
ActiveSupport::Notifications.subscribe("user.created") do |event|
  user = event.payload[:user]
  Analytics.track("user_signed_up", user_id: user.id)
  Slack.notify("#signups", "New user: #{user.email}")
end

# Or use a custom pub/sub
class EventBus
  @subscribers = Hash.new { |h, k| h[k] = [] }

  class << self
    def subscribe(event, &handler)
      @subscribers[event] << handler
    end

    def publish(event, payload = {})
      @subscribers[event].each { |handler| handler.call(payload) }
    end
  end
end

EventBus.subscribe(:user_created) { |data| puts "Welcome #{data[:name]}!" }
EventBus.subscribe(:user_created) { |data| Analytics.track(data[:id]) }

EventBus.publish(:user_created, name: "Alice", id: 1)
# => "Welcome Alice!"
# => (analytics tracked)
```

# Interview Prep — Coding Challenges

> Common Ruby coding challenges asked in technical interviews, with clean solutions.

---

## String Manipulation

### Palindrome check
```ruby
def palindrome?(str)
  cleaned = str.downcase.gsub(/[^a-z0-9]/, "")
  cleaned == cleaned.reverse
end

palindrome?("racecar")          # => true
palindrome?("A man a plan Panama")  # => true
palindrome?("hello")            # => false
```

### Anagram check
```ruby
def anagram?(str1, str2)
  normalize = ->(s) { s.downcase.chars.sort }
  normalize.call(str1) == normalize.call(str2)
end

anagram?("listen", "silent")   # => true
anagram?("hello", "world")     # => false

# Count-based approach (faster for long strings):
def anagram?(str1, str2)
  return false if str1.length != str2.length
  str1.downcase.chars.tally == str2.downcase.chars.tally
end
```

### Most frequent character
```ruby
def most_frequent_char(str)
  str.chars
     .tally
     .max_by { |char, count| count }
     .first
end

most_frequent_char("aabbbbccc")  # => "b"
```

---

## Arrays & Collections

### Two Sum
```ruby
# Find two numbers that add up to target, return their indices
def two_sum(nums, target)
  seen = {}  # value => index
  nums.each_with_index do |num, i|
    complement = target - num
    if seen.key?(complement)
      return [seen[complement], i]
    end
    seen[num] = i
  end
  []
end

two_sum([2, 7, 11, 15], 9)   # => [0, 1]
two_sum([3, 2, 4], 6)        # => [1, 2]
```

### Remove duplicates while preserving order
```ruby
def unique_preserving_order(arr)
  arr.each_with_object([]) do |item, result|
    result << item unless result.include?(item)
  end
end

# Simpler (order is preserved in modern Ruby):
[1, 2, 2, 3, 1, 4].uniq  # => [1, 2, 3, 4]
```

### Flatten nested array without `flatten`
```ruby
def flatten_array(arr)
  result = []
  arr.each do |element|
    if element.is_a?(Array)
      result.concat(flatten_array(element))
    else
      result << element
    end
  end
  result
end

flatten_array([1, [2, [3, 4], 5], 6])  # => [1, 2, 3, 4, 5, 6]
```

### Find missing number (1..n)
```ruby
def find_missing(arr, n)
  expected_sum = n * (n + 1) / 2
  expected_sum - arr.sum
end

find_missing([1, 2, 4, 5], 5)  # => 3
```

---

## Hashes

### Group and count words
```ruby
def word_frequency(text)
  text.downcase
      .scan(/[a-z]+/)
      .tally
      .sort_by { |_, count| -count }
      .to_h
end

word_frequency("the quick brown fox the fox")
# => {"the"=>2, "fox"=>2, "quick"=>1, "brown"=>1}
```

### Flatten nested hash
```ruby
def flatten_hash(hash, prefix = "")
  hash.each_with_object({}) do |(key, value), result|
    full_key = prefix.empty? ? key.to_s : "#{prefix}.#{key}"
    if value.is_a?(Hash)
      result.merge!(flatten_hash(value, full_key))
    else
      result[full_key] = value
    end
  end
end

flatten_hash({ a: 1, b: { c: 2, d: { e: 3 } } })
# => {"a"=>1, "b.c"=>2, "b.d.e"=>3}
```

---

## Numbers & Math

### FizzBuzz
```ruby
def fizzbuzz(n)
  (1..n).map do |i|
    if i % 15 == 0    then "FizzBuzz"
    elsif i % 3 == 0  then "Fizz"
    elsif i % 5 == 0  then "Buzz"
    else i.to_s
    end
  end
end

# Elegant Ruby version:
def fizzbuzz(n)
  (1..n).map do |i|
    [(i % 3 == 0 ? "Fizz" : ""), (i % 5 == 0 ? "Buzz" : "")].join
       .then { |s| s.empty? ? i.to_s : s }
  end
end
```

### Prime check
```ruby
def prime?(n)
  return false if n < 2
  (2..Math.sqrt(n).to_i).none? { |i| n % i == 0 }
end

def primes_up_to(n)
  (2..n).select { |num| prime?(num) }
end

primes_up_to(20)  # => [2, 3, 5, 7, 11, 13, 17, 19]
```

### Fibonacci
```ruby
# Recursive (simple but slow)
def fib(n)
  return n if n <= 1
  fib(n - 1) + fib(n - 2)
end

# Iterative (fast)
def fib(n)
  return n if n <= 1
  a, b = 0, 1
  (n - 1).times { a, b = b, a + b }
  b
end

# Using Enumerator (lazy, infinite)
fibs = Enumerator.new do |y|
  a, b = 0, 1
  loop do
    y << a
    a, b = b, a + b
  end
end

fibs.first(10)  # => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Memoization approach:
def fib(n, memo = {})
  return n if n <= 1
  memo[n] ||= fib(n - 1, memo) + fib(n - 2, memo)
end
```

---

## OOP Challenges

### Implement a Stack
```ruby
class Stack
  def initialize
    @data = []
  end

  def push(item)
    @data.push(item)
    self  # allows chaining
  end

  def pop
    raise "Stack underflow" if empty?
    @data.pop
  end

  def peek
    raise "Stack is empty" if empty?
    @data.last
  end

  def empty?
    @data.empty?
  end

  def size
    @data.size
  end

  def to_s
    "Stack(#{@data.join(', ')})"
  end
end

stack = Stack.new
stack.push(1).push(2).push(3)
stack.peek  # => 3
stack.pop   # => 3
stack.size  # => 2
```

### Implement a Queue using two Stacks
```ruby
class QueueWithStacks
  def initialize
    @inbox  = []
    @outbox = []
  end

  def enqueue(item)
    @inbox.push(item)
  end

  def dequeue
    if @outbox.empty?
      @outbox.push(@inbox.pop) until @inbox.empty?
    end
    raise "Queue is empty" if @outbox.empty?
    @outbox.pop
  end

  def peek
    if @outbox.empty?
      @outbox.push(@inbox.pop) until @inbox.empty?
    end
    @outbox.last
  end

  def empty?
    @inbox.empty? && @outbox.empty?
  end
end
```

### Implement LRU Cache
```ruby
class LRUCache
  def initialize(capacity)
    @capacity = capacity
    @cache    = {}  # Ruby Hash maintains insertion order
  end

  def get(key)
    return -1 unless @cache.key?(key)
    # Move to end (most recently used)
    value = @cache.delete(key)
    @cache[key] = value
    value
  end

  def put(key, value)
    if @cache.key?(key)
      @cache.delete(key)
    elsif @cache.size >= @capacity
      @cache.shift   # remove least recently used (first item)
    end
    @cache[key] = value
  end
end

cache = LRUCache.new(3)
cache.put(1, "one")
cache.put(2, "two")
cache.put(3, "three")
cache.get(1)          # => "one" (moves 1 to end)
cache.put(4, "four")  # evicts 2 (LRU after get moved 1)
cache.get(2)          # => -1 (evicted)
```

---

## ActiveRecord Challenges

### Find users who have written posts in multiple categories
```ruby
User.joins(posts: :categories)
    .group("users.id")
    .having("COUNT(DISTINCT categories.id) > 1")
    .distinct

# Or with subquery:
User.where(
  id: Post.joins(:categories)
          .group(:user_id)
          .having("COUNT(DISTINCT categories.id) > 1")
          .select(:user_id)
)
```

### Get top N users by post count in the last month
```ruby
User.joins(:posts)
    .where(posts: { created_at: 1.month.ago.. })
    .group("users.id")
    .order("COUNT(posts.id) DESC")
    .limit(10)
    .select("users.*, COUNT(posts.id) as posts_count")
```

### Find all records that have no associations
```ruby
# Users with no posts (Rails 6.1+)
User.where.missing(:posts)

# Older approach:
User.left_outer_joins(:posts).where(posts: { id: nil })
```

---

## Design Patterns

### Observer Pattern
```ruby
module Observable
  def self.included(base)
    base.instance_variable_set(:@observers, [])
    base.extend(ClassMethods)
  end

  module ClassMethods
    def add_observer(observer)
      @observers << observer
    end

    def observers
      @observers
    end
  end

  def notify_observers(event, data = nil)
    self.class.observers.each { |o| o.update(event, data) }
  end
end

class Order
  include Observable

  attr_reader :status

  def complete!
    @status = :completed
    notify_observers(:order_completed, self)
  end
end

class EmailNotifier
  def update(event, data)
    case event
    when :order_completed
      puts "Sending confirmation email for order #{data.id}"
    end
  end
end

Order.add_observer(EmailNotifier.new)
order = Order.new
order.complete!  # => "Sending confirmation email for order ..."
```

### Decorator Pattern
```ruby
class UserDecorator
  def initialize(user)
    @user = user
  end

  def display_name
    "#{@user.first_name} #{@user.last_name}".strip
  end

  def avatar_url
    @user.avatar.attached? ? @user.avatar.url : "/images/default_avatar.png"
  end

  def formatted_joined_at
    @user.created_at.strftime("%B %Y")
  end

  def method_missing(name, *args, &block)
    @user.send(name, *args, &block)
  end

  def respond_to_missing?(name, include_private = false)
    @user.respond_to?(name, include_private) || super
  end
end

# Using Draper gem (popular Rails decorator):
# class UserDecorator < Draper::Decorator
#   delegate_all
# end
```

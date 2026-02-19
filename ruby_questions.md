# Interview Prep — Ruby Questions

> Common interview questions with answers and code examples.

---

## 🔴 Core Language

### Q: What is the difference between `==`, `eql?`, and `equal?`?

```ruby
# == (value equality — often overridden by classes)
1 == 1.0       # => true  (Integer == Float)
"abc" == "abc" # => true  (different objects, same value)

# eql? (value equality WITH type checking)
1.eql?(1.0)    # => false (different types!)
1.eql?(1)      # => true

# equal? (object identity — same object in memory)
a = "hello"
b = "hello"
c = a
a.equal?(b)    # => false (different objects)
a.equal?(c)    # => true  (same object!)
a.object_id == c.object_id  # => true

# In practice:
# Use == for value comparison
# Use equal? to check if same object
# Use eql? for Hash key comparison (Hash uses eql? internally)
```

---

### Q: What is the difference between `dup` and `clone`?

```ruby
original = "hello".freeze

# dup — creates unfrozen copy, doesn't copy singleton methods
copy = original.dup
copy.frozen?     # => false ✓
copy << " world" # => "hello world" ✓

# clone — preserves frozen state AND singleton methods
cloned = original.clone
cloned.frozen?   # => true (still frozen!)
cloned << " world" # => FrozenError!

# Both are SHALLOW copies:
original = [[1, 2], [3, 4]]
shallow  = original.dup
shallow[0] << 99
original  # => [[1, 2, 99], [3, 4]]  (inner arrays shared!)

# Deep copy:
require 'deep_clone'
deep = original.deep_clone
# or: Marshal.load(Marshal.dump(original))  (slow but works)
```

---

### Q: Explain `freeze` and when to use it

```ruby
# freeze makes an object immutable
str = "hello".freeze
str << " world"  # => FrozenError!
str.upcase       # ✓ fine — returns new string
str.upcase!      # => FrozenError! (modifies in place)

# Symbols and Integers are always frozen
:hello.frozen?   # => true
42.frozen?       # => true

# Use cases:
# 1. Constants that should never change
ALLOWED_ROLES = %w[admin user moderator].freeze

# 2. Performance — frozen strings can be shared (interned)
# 3. Safety in concurrent code

# Freeze doesn't freeze nested objects!
arr = [[1, 2], [3, 4]].freeze
arr << [5, 6]      # => FrozenError!
arr[0] << 99       # ✓ arr is frozen but arr[0] is not
```

---

### Q: What are the differences between Proc and Lambda?

```ruby
# 1. RETURN BEHAVIOR
def proc_return
  p = Proc.new { return "from proc" }
  p.call
  "from method"  # NEVER reached!
end

def lambda_return
  l = lambda { return "from lambda" }
  l.call
  "from method"  # ALWAYS reached
end

proc_return   # => "from proc"
lambda_return # => "from method"

# 2. ARGUMENT STRICTNESS
strict = lambda { |a, b| a + b }
strict.call(1, 2, 3)   # ArgumentError!

lenient = Proc.new { |a, b| [a, b] }
lenient.call(1, 2, 3)  # => [1, 2]  (extra ignored)
lenient.call(1)         # => [1, nil] (missing = nil)

# 3. LAMBDA CHECK
lambda {}.lambda?         # => true
Proc.new {}.lambda?       # => false
method(:puts).to_proc.lambda?  # => true
```

---

### Q: What is `method_missing` and what are the risks?

```ruby
class Ghost
  def method_missing(name, *args)
    if name.to_s =~ /^say_(.*)/
      puts $1
    else
      super  # CRITICAL: always call super!
    end
  end

  def respond_to_missing?(name, include_private = false)
    name.to_s.start_with?("say_") || super
  end
end

g = Ghost.new
g.say_hello   # => "hello"
g.say_world   # => "world"
g.foo         # => NoMethodError (passes through super)

# Risks:
# 1. Debugging is harder — stack traces are confusing
# 2. Performance — slower than defined methods
# 3. Forgetting to call super = swallowed NoMethodErrors
# 4. Forgetting respond_to_missing? breaks duck typing
```

---

### Q: What is the difference between `include`, `extend`, and `prepend`?

```ruby
module Greetable
  def greet
    "Hello from #{self.class}"
  end
end

class WithInclude
  include Greetable    # adds as INSTANCE methods
end
WithInclude.new.greet  # ✓

class WithExtend
  extend Greetable     # adds as CLASS methods
end
WithExtend.greet       # ✓ class method
WithExtend.new.greet   # NoMethodError

class WithPrepend
  prepend Greetable    # adds BEFORE the class in method lookup

  def greet
    "Hello from base class"
  end
end
# Method lookup: Greetable first, then WithPrepend
WithPrepend.new.greet  # => "Hello from #{self.class}" — Greetable wins!

# Use prepend for:
# - Aspect-oriented programming / decorators
# - Wrapping existing methods
module Logging
  def save
    puts "Saving #{self.class}..."
    result = super
    puts "Saved!"
    result
  end
end

class User < ApplicationRecord
  prepend Logging   # Logging#save wraps User#save
end
```

---

### Q: Explain Ruby's object model

```ruby
# Everything is an object, every object has a class
1.class           # => Integer
Integer.class     # => Class
Class.class       # => Class (Class is its own class!)
Object.class      # => Class
BasicObject.class # => Class

# Method lookup chain (ancestors)
class MyClass
  include ModuleA
  include ModuleB
end
MyClass.ancestors
# => [MyClass, ModuleB, ModuleA, Object, Kernel, BasicObject]
# Note: last included module is first in chain

# Eigenclass (singleton class) — unique class per object
obj = Object.new
def obj.hello         # defined only on this object
  "Hello from singleton"
end
obj.hello             # => "Hello from singleton"
Object.new.hello      # NoMethodError

# Access eigenclass
obj.singleton_class
# or: class << obj; self; end
```

---

## 🔴 Common Gotchas

### Q: Why does this fail? (Common pitfall)

```ruby
# GOTCHA: Assignment in condition
user = nil
if user = User.find_by(email: "alice@x.com")  # assignment, not comparison!
  # always true if User exists
end

# GOTCHA: Frozen string mutation
str = "hello"
str.freeze
str.upcase!   # FrozenError — use str.upcase instead

# GOTCHA: Symbol vs string keys
h = { "name" => "Alice" }
h[:name]    # => nil! (symbol key, string stored)
h["name"]   # => "Alice"

# Use HashWithIndifferentAccess:
h = ActiveSupport::HashWithIndifferentAccess.new("name" => "Alice")
h[:name]   # => "Alice"
h["name"]  # => "Alice"

# GOTCHA: Modifying array while iterating
arr = [1, 2, 3, 4, 5]
arr.each { |n| arr.delete(n) }  # skips elements!
arr  # => [2, 4]  (not empty!)

# Use dup or select instead:
arr = [1, 2, 3, 4, 5]
arr.reject! { |n| n.odd? }  # safer

# GOTCHA: return in a block
def process
  [1, 2, 3].each do |n|
    return n if n == 2    # returns from process(), not just block!
  end
  "finished"  # never reached!
end
process  # => 2

# GOTCHA: && vs and (different precedence)
result = true && false    # result = false  (expected)
result = true and false   # result = true!  (= has higher precedence than and)
```

---

## 🔴 Performance Questions

### Q: How do you handle memory in Ruby?

```ruby
# 1. Use lazy enumerators for large collections
(1..1_000_000).lazy.select(&:odd?).first(100)
# vs:
(1..1_000_000).select(&:odd?).first(100)  # loads 1M into memory!

# 2. Prefer symbols over strings for repeated identifiers
# Strings create new objects each time
"status"  # new object
"status"  # another new object
:status   # same object always

# 3. Avoid creating unnecessary objects in hot paths
def expensive_wrong
  [*1..1000].map { |n| n * 2 }  # creates intermediate array
end

def expensive_better
  (1..1000).map { |n| n * 2 }   # range is lazy
end

# 4. String concatenation: use << instead of +
result = ""
1000.times { result << "x" }  # ✓ modifies in place
1000.times { result = result + "x" }  # ❌ creates new string each time

# 5. freeze string literals
# frozen_string_literal: true  (at top of file)
```

---

### Q: What's the difference between `map`, `each`, `collect`, `select`?

```ruby
arr = [1, 2, 3, 4, 5]

# each — iterate, returns original array
arr.each { |n| puts n }  # side effects only, returns [1,2,3,4,5]

# map / collect — transform, returns new array
arr.map { |n| n * 2 }    # => [2, 4, 6, 8, 10]
arr.collect { |n| n * 2 } # alias for map

# select / filter — filter, returns matching elements
arr.select { |n| n > 3 }  # => [4, 5]
arr.filter { |n| n > 3 }  # alias for select

# reject — opposite of select
arr.reject { |n| n > 3 }  # => [1, 2, 3]

# find / detect — returns first match
arr.find { |n| n > 3 }    # => 4

# reduce / inject — accumulate
arr.reduce(:+)             # => 15
arr.inject(0) { |sum, n| sum + n }  # => 15

# flat_map — map + flatten
[[1,2],[3,4]].flat_map { |a| a.map { |n| n * 2 } }  # => [2,4,6,8]

# filter_map (Ruby 2.7+) — select + transform in one
arr.filter_map { |n| n * 2 if n > 3 }  # => [8, 10]
```

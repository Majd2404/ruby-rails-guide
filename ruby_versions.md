# Version Differences — Ruby 2.x to 3.x

## Quick Summary

| Version | Key Features |
|---|---|
| Ruby 2.3 | Safe navigation `&.`, frozen strings |
| Ruby 2.4 | Hash#transform_values, Integer unified |
| Ruby 2.5 | Hash#slice, Array#append/prepend, yield_self |
| Ruby 2.6 | Endless range (1..), Array#union/difference, then alias |
| Ruby 2.7 | Pattern matching (experimental), numbered params `_1`, Hash/Array deconstruct |
| Ruby 3.0 | Pattern matching stable, Ractors, Fibers scheduler, rightward assignment, no keyword arg warnings |
| Ruby 3.1 | Find pattern, hash shorthand `{x:}`, one-line pattern matching |
| Ruby 3.2 | Data class, Regexp optimizations, WASM support |
| Ruby 3.3 | Prism parser, performance, YJIT improvements |
| Ruby 3.4 | `it` block variable, improved error messages |

---

## Ruby 2.3

```ruby
# Safe navigation operator (&.) — avoids NoMethodError on nil
user = nil
user&.name        # => nil (no crash!)
user&.name&.upcase # => nil

# Without it:
user && user.name && user.name.upcase  # old way

# Frozen string literals (performance optimization)
# frozen_string_literal: true
# Add to top of file — all string literals become frozen

str = "hello"
str.frozen?  # => true if magic comment is present
str << " world"  # => FrozenError!
```

---

## Ruby 2.4

```ruby
# Hash#transform_values
{ a: 1, b: 2 }.transform_values { |v| v * 2 }   # => {a: 2, b: 4}
{ a: 1, b: 2 }.transform_values(&:to_s)          # => {a: "1", b: "2"}

# Integer unification — Fixnum and Bignum merged into Integer
1.class   # => Integer (was Fixnum before 2.4)

# String#match? (returns boolean, no MatchData capture)
"hello".match?(/e/)   # => true (faster than =~)
```

---

## Ruby 2.5

```ruby
# Hash#slice — extract subset of keys
h = { a: 1, b: 2, c: 3, d: 4 }
h.slice(:a, :c)   # => {a: 1, c: 3}

# Hash#transform_keys
{ "name" => "Alice" }.transform_keys(&:to_sym)  # => {name: "Alice"}

# yield_self / then (pipeline operator)
"hello"
  .yield_self { |s| s.upcase }
  .yield_self { |s| "#{s} WORLD" }
# => "HELLO WORLD"

# Array#prepend / #append
arr = [3, 4]
arr.prepend(1, 2)    # => [1, 2, 3, 4]  (alias for unshift)
arr.append(5, 6)     # => [1, 2, 3, 4, 5, 6] (alias for push)

# Kernel#pp returns value (not nil)
result = pp "hello"  # prints and returns "hello"
```

---

## Ruby 2.6

```ruby
# Endless range (beginless in 2.7)
(1..)             # 1 to infinity
arr = [1,2,3,4,5]
arr[2..]          # => [3, 4, 5]  (from index 2 to end)
arr[..2]          # => [1, 2, 3]  (from start to index 2) [Ruby 2.7]

(1..).each { |n| break if n > 5 }  # safe with break

# Array.new with union and difference
[1,2,3].union([3,4,5])        # => [1,2,3,4,5] (unique elements)
[1,2,3].difference([2,3])     # => [1]

# then as alias for yield_self
"hello".then { |s| s.upcase }  # cleaner name

# Proc#<< and Proc#>> (composition)
double = ->(n) { n * 2 }
add_one = ->(n) { n + 1 }

(double >> add_one).call(5)   # double then add_one: (5*2)+1 = 11
(double << add_one).call(5)   # add_one then double: (5+1)*2 = 12

# Enumerable#chain
(1..3).chain(5..7).to_a   # => [1, 2, 3, 5, 6, 7]
```

---

## Ruby 2.7 ⭐ Major Release

```ruby
# Pattern Matching (experimental in 2.7, stable in 3.0)
case user
in { name: String => name, age: (18..) => age }
  puts "Adult user: #{name}, age #{age}"
in { name: String => name }
  puts "User: #{name}"
end

# Array pattern
case data
in [Integer => first, *rest]
  puts "First integer: #{first}, rest: #{rest}"
end

# Numbered block parameters (_1, _2, etc.)
[1, 2, 3].map { _1 * 2 }             # => [2, 4, 6]
{ a: 1, b: 2 }.map { "#{_1}=#{_2}" } # => ["a=1", "b=2"]

# Hash#filter_map (select + map in one)
[1, 2, 3, nil, 4].filter_map { |n| n && n * 2 }  # => [2, 4, 6, 8]
User.all.filter_map { |u| u.email if u.active? }

# Beginless range
arr = [1, 2, 3, 4, 5]
arr[..2]           # => [1, 2, 3]
arr[...2]          # => [1, 2]
(..5) === 3        # => true

# Warning: Keyword argument changes (preparing for 3.0)
# Using positional hash as keyword args gives warning in 2.7
def greet(name:); end
options = { name: "Alice" }
greet(options)  # works in 2.7 with warning, removed in 3.0
```

---

## Ruby 3.0 ⭐⭐ Major Breaking Release

```ruby
# Keyword argument separation (BREAKING CHANGE from 2.7)
def process(name:, age:)
  "#{name}, #{age}"
end

hash = { name: "Alice", age: 30 }
process(**hash)    # ✓ must explicitly splat with **
process(hash)      # ❌ ArgumentError in 3.0!

# Pattern matching is now stable
response = { status: 200, body: { user: { name: "Alice" } } }

case response
in { status: 200, body: { user: { name: String => name } } }
  puts "Got user: #{name}"
in { status: 404 }
  puts "Not found"
in { status: 500 }
  puts "Server error"
end

# Find pattern (3.0)
case [1, 2, "hello", 3, 4]
in [*, String => s, *]
  puts "Found string: #{s}"
end

# Rightward assignment (fun but rarely used)
"hello".upcase => result
puts result  # => "HELLO"

# Ractors (parallel execution, experimental)
ractor = Ractor.new do
  Ractor.recv.upcase
end
ractor.send("hello")
ractor.take  # => "HELLO"

# Hash#except (opposite of slice)
{ a: 1, b: 2, c: 3 }.except(:b)   # => {a: 1, c: 3}

# Performance improvements
# YJIT (Yet Another JIT) introduced as optional flag
# ruby --yjit app.rb
```

---

## Ruby 3.1

```ruby
# Hash shorthand syntax  {x:} instead of {x: x}
name = "Alice"
age  = 30
{ name:, age: }           # => {name: "Alice", age: 30}
User.new(name:, age:)     # same as User.new(name: name, age: age)

# One-line pattern matching with =>
response => { status:, body: }  # destructures and assigns variables
puts status   # => 200

# Find pattern stabilized
# Pin operator (^) in patterns
expected_name = "Alice"
case user
in { name: ^expected_name }
  puts "Found Alice"
end

# error backtrace improvements — highlights exact error location

# Proc#arity returns correct value now for numbered params
triple = -> { _1 * 3 }
triple.arity  # => 0 (was -1 before)
```

---

## Ruby 3.2

```ruby
# Data class — immutable value object
Point = Data.define(:x, :y)
p = Point.new(x: 1, y: 2)
p.x          # => 1
p.frozen?    # => true
p.with(x: 5) # => Point(x: 5, y: 2)  — creates modified copy

# Compare to Struct:
# Struct is mutable, Data is immutable
PointStruct = Struct.new(:x, :y, keyword_init: true)
ps = PointStruct.new(x: 1, y: 2)
ps.x = 10  # ✓ can mutate

# Data#with  (non-destructive update)
p1 = Point.new(x: 1, y: 2)
p2 = p1.with(y: 5)
p1.y  # => 2 (unchanged)
p2.y  # => 5

# Pattern matching improvements
# Deconstruct_keys for custom classes
class UserVO
  attr_reader :name, :email

  def initialize(name, email)
    @name  = name
    @email = email
  end

  def deconstruct_keys(keys)
    { name: @name, email: @email }
  end
end

user = UserVO.new("Alice", "alice@x.com")
case user
in { name: "Alice", email: }
  puts "Alice's email: #{email}"
end

# Regexp optimizations — Regexes now much faster and no ReDoS vulnerability
```

---

## Ruby 3.3 / 3.4

```ruby
# Ruby 3.3
# Prism parser — new default parser (faster, better errors)
# YJIT enabled by default
# Fiber scheduler improvements

# Ruby 3.4 (latest)
# `it` as block variable (replaces _1 for single param blocks)
[1, 2, 3].map { it * 2 }          # => [2, 4, 6]
{ a: 1, b: 2 }.select { it > 1 }  # => {b: 2}

# Better error messages with "did you mean?" suggestions
# loc references in error messages

# Happy Eyeballs for network connections
# Fiber scheduler built-in event loop
```

---

## Migration Tips

### 2.x → 3.0 (Common Breaking Changes)

```ruby
# 1. Keyword argument splatting
# Before (2.x):
def foo(a:, b:); end
opts = {a: 1, b: 2}
foo(opts)  # worked (with warning in 2.7)

# After (3.0):
foo(**opts)  # must use double splat

# 2. Hash pattern has stricter rules
def bar(a, **opts); end
bar({a: 1})   # Error in 3.0 — use bar(a: 1)

# 3. Check Ruby compatibility
ruby --version
# In .ruby-version or Gemfile:
ruby ">= 3.0.0"
```

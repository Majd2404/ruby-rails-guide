# Ruby Fundamentals — 03: Blocks, Procs & Lambdas

> This is one of the most important and frequently tested topics in Ruby interviews!

## Table of Contents
- [Blocks](#blocks)
- [Procs](#procs)
- [Lambdas](#lambdas)
- [Key Differences: Proc vs Lambda](#key-differences-proc-vs-lambda)
- [Closures](#closures)
- [yield](#yield)
- [Practical Patterns](#practical-patterns)

---

## Blocks

A **block** is an anonymous piece of code attached to a method call. It's not an object.

```ruby
# Block with do...end (multi-line)
[1, 2, 3].each do |n|
  puts n * 2
end

# Block with {} (single-line)
[1, 2, 3].each { |n| puts n * 2 }

# Accepting a block in your own method
def run_block
  yield if block_given?   # yield executes the block
end

run_block { puts "Block ran!" }  # => "Block ran!"
run_block                        # no error, block_given? is false

# Passing values to yield
def calculate(a, b)
  result = yield(a, b)
  puts "Result: #{result}"
end

calculate(4, 5) { |x, y| x + y }  # => "Result: 9"
calculate(4, 5) { |x, y| x * y }  # => "Result: 20"

# Capturing a block as a parameter with &
def run_twice(&block)
  block.call
  block.call
end

run_twice { puts "Hello!" }
# Hello!
# Hello!
```

---

## Procs

A **Proc** is an object that wraps a block. It can be stored in a variable.

```ruby
# Creating a Proc
doubler = Proc.new { |n| n * 2 }
squarer = proc { |n| n ** 2 }   # proc{} is shorthand

# Calling a Proc
doubler.call(5)   # => 10
doubler.(5)       # => 10  (alternative syntax)
doubler[5]        # => 10  (alternative syntax)

# Proc stored and reused
formatter = Proc.new { |name, age| "#{name} is #{age} years old" }
formatter.call("Alice", 30)  # => "Alice is 30 years old"
formatter.call("Bob", 25)    # => "Bob is 25 years old"

# Using & to convert Proc to block
evens = [1, 2, 3, 4, 5].select(&method(:even?))  # won't work, but:
double = proc { |n| n * 2 }
[1, 2, 3].map(&double)   # => [2, 4, 6]

# Symbol#to_proc — very common pattern!
[1, 2, 3].map(&:to_s)       # => ["1", "2", "3"]
["alice", "bob"].map(&:capitalize)  # => ["Alice", "Bob"]
[1, nil, 2, nil].select(&:itself)   # => [1, 2]

# What &:to_s does:
# :to_s.to_proc  =>  proc { |obj| obj.to_s }
```

---

## Lambdas

A **lambda** is a stricter, method-like Proc.

```ruby
# Creating lambdas
square   = lambda { |n| n ** 2 }
cube     = ->(n) { n ** 3 }            # stabby lambda (Ruby 1.9+)
add      = ->(a, b) { a + b }

# Calling lambdas
square.call(4)   # => 16
cube.(3)         # => 27
add.(2, 3)       # => 5

# Multi-line lambda
greet = ->(name, greeting: "Hello") {
  "#{greeting}, #{name}!"
}
greet.call("Alice")                    # => "Hello, Alice!"
greet.call("Bob", greeting: "Hi")     # => "Hi, Bob!"

# Lambda used as a strategy pattern
operations = {
  add:      ->(a, b) { a + b },
  subtract: ->(a, b) { a - b },
  multiply: ->(a, b) { a * b },
  divide:   ->(a, b) { b != 0 ? a.to_f / b : "Error" }
}

operations[:add].call(5, 3)       # => 8
operations[:multiply].call(4, 7)  # => 28
```

---

## Key Differences: Proc vs Lambda

### 1. Argument Handling

```ruby
# Lambda STRICT about arguments
strict = lambda { |a, b| a + b }
strict.call(1, 2)      # => 3
strict.call(1, 2, 3)   # ArgumentError! too many arguments
strict.call(1)         # ArgumentError! too few arguments

# Proc LENIENT about arguments
lenient = proc { |a, b| "#{a}, #{b}" }
lenient.call(1, 2)     # => "1, 2"
lenient.call(1, 2, 3)  # => "1, 2"  (extra ignored)
lenient.call(1)        # => "1, "   (missing = nil)
lenient.call           # => ", "    (all nil)
```

### 2. Return Behavior

```ruby
# Lambda return exits the LAMBDA only
def lambda_test
  lam = lambda { return 10 }
  result = lam.call
  puts "After lambda: #{result}"  # => "After lambda: 10"
  puts "Method continues"         # => "Method continues"
end
lambda_test

# Proc return exits the ENCLOSING METHOD
def proc_test
  p = proc { return 10 }
  p.call
  puts "After proc"  # NEVER reached!
end
proc_test  # returns 10, "After proc" never prints

# This is why Procs in blocks can cause surprising exits!
def find_first_even(numbers)
  numbers.each do |n|
    return n if n.even?   # this returns from the METHOD
  end
  nil
end
```

### 3. Summary Table

| Feature | Block | Proc | Lambda |
|---|---|---|---|
| Is an object | ❌ | ✅ | ✅ |
| Argument strictness | N/A | Lenient | Strict |
| `return` behavior | Returns from method | Returns from method | Returns from lambda |
| Created with | `{}` or `do..end` | `proc {}` or `Proc.new {}` | `lambda {}` or `->() {}` |
| Check with | `block_given?` | `obj.lambda?` → false | `obj.lambda?` → true |

---

## Closures

Procs and Lambdas are **closures** — they capture and remember their surrounding scope.

```ruby
def make_counter
  count = 0
  increment = -> { count += 1 }
  get_count  = -> { count }
  [increment, get_count]
end

increment, get_count = make_counter

increment.call
increment.call
increment.call
get_count.call  # => 3 — it remembers `count` even after make_counter returned!

# Closures share the same binding
def make_adder(n)
  ->(x) { x + n }   # captures `n` from outer scope
end

add5  = make_adder(5)
add10 = make_adder(10)

add5.call(3)   # => 8
add10.call(3)  # => 13

# Closures can modify outer variables (be careful!)
x = 10
modify = -> { x = 99 }
modify.call
puts x  # => 99
```

---

## yield

```ruby
# Basic yield
def measure
  start = Time.now
  yield
  elapsed = Time.now - start
  puts "Elapsed: #{elapsed.round(4)}s"
end

measure { sleep(0.1) }  # => "Elapsed: 0.1001s"

# yield with value
def transform(data)
  yield(data)
end

transform("hello") { |s| s.upcase }  # => "HELLO"

# yield with multiple values
def each_pair(arr)
  arr.each_slice(2) { |a, b| yield(a, b) }
end

each_pair([1, 2, 3, 4]) { |a, b| puts "#{a} + #{b} = #{a + b}" }
# 1 + 2 = 3
# 3 + 4 = 7

# Explicit block parameter (captures the block as a Proc)
def run_with_logging(&block)
  puts "Starting..."
  result = block.call
  puts "Done. Result: #{result}"
  result
end

run_with_logging { 2 + 2 }
# Starting...
# Done. Result: 4
```

---

## Practical Patterns

```ruby
# Strategy Pattern
class Report
  def initialize(formatter)
    @formatter = formatter
  end

  def generate(data)
    @formatter.call(data)
  end
end

json_formatter = ->(data) { data.to_json }
csv_formatter  = ->(data) { data.map(&:values).map { |row| row.join(",") }.join("\n") }

report = Report.new(json_formatter)
report.generate([{ name: "Alice" }])

# Memoization with lazy block
class ExpensiveCalculator
  def result
    @result ||= compute
  end

  private

  def compute
    puts "Computing..."
    # ... heavy work
    42
  end
end

# Tap for debugging in chains
result = [1, 2, 3, 4, 5]
  .tap { |a| puts "Original: #{a}" }
  .select(&:odd?)
  .tap { |a| puts "After filter: #{a}" }
  .map { |n| n ** 2 }
  .tap { |a| puts "After map: #{a}" }

# then/yield_self for pipeline transformations
"hello world"
  .then { |s| s.split }          # => ["hello", "world"]
  .then { |arr| arr.map(&:capitalize) }  # => ["Hello", "World"]
  .then { |arr| arr.join(" ") }  # => "Hello World"
```

# Ruby Fundamentals — 01: Basics

## Table of Contents
- [Data Types](#data-types)
- [Variables & Scope](#variables--scope)
- [String Operations](#string-operations)
- [Control Flow](#control-flow)
- [Methods](#methods)
- [Symbols vs Strings](#symbols-vs-strings)
- [Ranges](#ranges)
- [Truthiness & Nil](#truthiness--nil)

---

## Data Types

```ruby
# Integer
age = 25
big_num = 1_000_000      # underscore for readability

# Float
price = 19.99

# String
name = "Alice"
greeting = 'Hello'       # single quotes = no interpolation

# Boolean
is_active = true
is_deleted = false

# Nil
value = nil              # represents absence of value

# Symbol (immutable, reusable identifier)
status = :pending
role   = :admin

# Array
fruits = ["apple", "banana", "cherry"]
mixed  = [1, "two", :three, nil, true]

# Hash
user = { name: "Alice", age: 30, role: :admin }
# old-style hash rocket syntax (still valid):
user = { :name => "Alice", :age => 30 }

# Range
(1..5)    # inclusive: 1,2,3,4,5
(1...5)   # exclusive: 1,2,3,4
('a'..'e') # alphabetical range
```

---

## Variables & Scope

```ruby
local_var   = "only in this scope"   # local variable
@instance   = "belongs to object"    # instance variable
@@class_var = "shared across class"  # class variable
$global     = "app-wide (avoid!)"    # global variable
CONSTANT    = "immutable value"      # constant (PascalCase for classes)

# Scope example
def example_method
  local = "local to method"
  @instance_var = "accessible anywhere in object"
end

class Dog
  @@count = 0                    # shared across ALL Dog instances

  def initialize(name)
    @name = name                 # unique per instance
    @@count += 1
  end

  def self.count
    @@count
  end
end

Dog.new("Rex")
Dog.new("Buddy")
Dog.count  # => 2
```

---

## String Operations

```ruby
name = "Alice"

# Interpolation (double quotes only)
puts "Hello, #{name}!"          # => Hello, Alice!
puts "1 + 1 = #{1 + 1}"        # => 1 + 1 = 2

# Common methods
"hello".upcase                  # => "HELLO"
"HELLO".downcase                # => "hello"
"hello world".capitalize        # => "Hello world"
"hello".length                  # => 5
"hello".reverse                 # => "olleh"
"  hello  ".strip               # => "hello"
"hello world".split(" ")        # => ["hello", "world"]
"hello".include?("ell")         # => true
"hello world".gsub("world", "Ruby") # => "hello Ruby"
"ha" * 3                        # => "hahaha"

# Heredoc (multi-line string)
text = <<~HEREDOC
  This is a
  multi-line string
HEREDOC

# String formatting
"Price: %.2f" % 9.5             # => "Price: 9.50"
"%s is %d years old" % ["Bob", 25]  # => "Bob is 25 years old"
```

---

## Control Flow

```ruby
# if / elsif / else
score = 85

if score >= 90
  puts "A"
elsif score >= 80
  puts "B"
elsif score >= 70
  puts "C"
else
  puts "F"
end

# One-liner (postfix if)
puts "Pass" if score >= 60
puts "Fail" unless score >= 60

# ternary
result = score >= 60 ? "Pass" : "Fail"

# case / when  (like switch)
grade = case score
        when 90..100 then "A"
        when 80..89  then "B"
        when 70..79  then "C"
        else              "F"
        end

# case with no variable (acts like if/elsif)
case
when score > 90 then puts "Excellent"
when score > 70 then puts "Good"
else puts "Needs improvement"
end

# Loops
5.times { |i| puts i }          # 0 1 2 3 4

(1..5).each { |n| puts n }      # 1 2 3 4 5

i = 0
while i < 3
  puts i
  i += 1
end

i = 0
until i >= 3
  puts i
  i += 1
end

# loop with break
loop do
  input = gets.chomp
  break if input == "quit"
end
```

---

## Methods

```ruby
# Basic method
def greet(name)
  "Hello, #{name}!"              # last expression is returned (implicit return)
end

greet("Alice")  # => "Hello, Alice!"

# Default parameters
def greet(name = "World")
  "Hello, #{name}!"
end

greet           # => "Hello, World!"
greet("Bob")    # => "Hello, Bob!"

# Keyword arguments (preferred in modern Ruby)
def create_user(name:, age:, role: :user)
  { name: name, age: age, role: role }
end

create_user(name: "Alice", age: 30)
create_user(age: 25, name: "Bob", role: :admin)  # order doesn't matter

# Splat operators
def sum(*numbers)               # collects into array
  numbers.sum
end
sum(1, 2, 3, 4)  # => 10

def log(message, **options)     # double splat = keyword hash
  puts "[#{options[:level]}] #{message}"
end
log("Error!", level: :error)

# Explicit return
def divide(a, b)
  return "Can't divide by zero" if b == 0
  a / b
end

# Method with block
def repeat(n)
  n.times { yield }             # yield calls the block
end
repeat(3) { puts "Hello!" }

# Predicate methods (return boolean, end with ?)
def valid?(value)
  !value.nil? && !value.empty?
end

# Bang methods (mutate in place, end with !)
arr = [3, 1, 2]
arr.sort    # returns new sorted array, arr unchanged
arr.sort!   # sorts arr in place
```

---

## Symbols vs Strings

```ruby
# Strings are mutable, created fresh each time
"hello".object_id  # different each time
"hello".object_id  # different again

# Symbols are immutable, same object_id always
:hello.object_id   # same
:hello.object_id   # same — more memory efficient for identifiers

# Use Symbols for:
# - Hash keys
# - Method names / identifiers
# - Enumerations / fixed values
config = { host: "localhost", port: 3000 }
status = :pending

# Convert between them
:hello.to_s   # => "hello"
"hello".to_sym # => :hello
```

---

## Ranges

```ruby
(1..10).to_a            # => [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
(1...10).to_a           # => [1, 2, 3, 4, 5, 6, 7, 8, 9]

(1..10).include?(5)     # => true
(1..10).min             # => 1
(1..10).max             # => 10
(1..10).sum             # => 55

# Useful in case/when
case age
when 0..12  then "child"
when 13..17 then "teen"
when 18..64 then "adult"
else             "senior"
end

# Step
(1..10).step(2).to_a    # => [1, 3, 5, 7, 9]
```

---

## Truthiness & Nil

```ruby
# In Ruby, ONLY nil and false are falsy
# Everything else (0, "", [], {}) is TRUTHY

puts "truthy" if 0          # prints (0 is truthy in Ruby!)
puts "truthy" if ""         # prints ("" is truthy!)
puts "truthy" if []         # prints ([] is truthy!)

puts "falsy" unless nil     # prints
puts "falsy" unless false   # prints

# Safe navigation operator &. (Ruby 2.3+)
user = nil
user&.name    # => nil (no NoMethodError!)
user.name     # => NoMethodError!

# nil checks
value = nil
value.nil?          # => true
value.is_a?(NilClass) # => true

# or-assignment
config = nil
config ||= {}        # assign only if nil/false
config[:timeout] = 30

# Conditional assignment
x = 5
x ||= 10            # won't reassign, x is already truthy
x &&= x * 2         # assigns only if x is truthy: x = 10
```

---

## ⚡ Quick Reference

| Concept | Ruby |
|---|---|
| Print with newline | `puts "hello"` |
| Print without newline | `print "hello"` |
| String interpolation | `"Hello #{name}"` |
| Check type | `42.is_a?(Integer)` |
| Convert to string | `42.to_s` |
| Convert to integer | `"42".to_i` |
| Convert to float | `"3.14".to_f` |
| Convert to array | `(1..5).to_a` |
| Nil check | `value.nil?` |
| Frozen check | `str.frozen?` |

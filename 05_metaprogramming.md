# Ruby Fundamentals — 05: Metaprogramming

> Writing code that writes code. Ruby's most powerful and distinctive feature.

## Table of Contents
- [method_missing](#method_missing)
- [define_method](#define_method)
- [send & public_send](#send--public_send)
- [respond_to_missing?](#respond_to_missing)
- [open Classes & Monkey Patching](#open-classes--monkey-patching)
- [Hooks: included, extended, inherited](#hooks-included-extended-inherited)
- [class_eval & instance_eval](#class_eval--instance_eval)
- [Object Model Recap](#object-model-recap)

---

## method_missing

Called when a method is not found. Lets you intercept undefined method calls.

```ruby
class DynamicFinder
  def initialize(data)
    @data = data
  end

  def method_missing(method_name, *args)
    method_str = method_name.to_s

    if method_str.start_with?("find_by_")
      attribute = method_str.sub("find_by_", "")
      value = args.first
      @data.find { |item| item[attribute.to_sym] == value }
    else
      super  # ALWAYS call super if you can't handle it!
    end
  end

  def respond_to_missing?(method_name, include_private = false)
    method_name.to_s.start_with?("find_by_") || super
  end
end

users = [
  { id: 1, name: "Alice", role: :admin },
  { id: 2, name: "Bob",   role: :user },
  { id: 3, name: "Charlie", role: :user }
]

finder = DynamicFinder.new(users)
finder.find_by_name("Alice")   # => {id:1, name:"Alice", role: :admin}
finder.find_by_role(:user)     # => {id:2, name:"Bob", role: :user}
finder.respond_to?(:find_by_name)  # => true (thanks to respond_to_missing?)

# ActiveRecord uses this pattern for find_by_email, find_by_name etc.
```

---

## define_method

Dynamically define methods at class/module level.

```ruby
class Person
  ATTRIBUTES = [:name, :age, :email]

  # Dynamically generate getters and setters
  ATTRIBUTES.each do |attr|
    define_method(attr) do
      instance_variable_get("@#{attr}")
    end

    define_method("#{attr}=") do |value|
      instance_variable_set("@#{attr}", value)
    end
  end

  def initialize(attrs = {})
    attrs.each { |k, v| send("#{k}=", v) }
  end
end

person = Person.new(name: "Alice", age: 30, email: "alice@example.com")
person.name   # => "Alice"
person.age    # => 30

# Real-world example: generating state methods
class Order
  STATES = [:pending, :processing, :shipped, :delivered, :cancelled]

  attr_reader :state

  def initialize
    @state = :pending
  end

  # Generates: pending?, processing?, shipped?, etc.
  STATES.each do |state|
    define_method("#{state}?") do
      @state == state
    end

    # Generates: transition_to_pending!, etc.
    define_method("transition_to_#{state}!") do
      @state = state
    end
  end
end

order = Order.new
order.pending?             # => true
order.transition_to_shipped!
order.shipped?             # => true
```

---

## send & public_send

Call methods dynamically by name.

```ruby
"hello".send(:upcase)          # => "HELLO"
"hello".send(:[], 1)           # => "e"
[1,2,3].send(:push, 4)        # => [1,2,3,4]

# Can call private methods (dangerous!)
class Secret
  private
  def hidden_info
    "Super secret"
  end
end

s = Secret.new
s.send(:hidden_info)          # => "Super secret" — bypasses private!
s.public_send(:hidden_info)   # => NoMethodError — respects private

# Useful for dynamic dispatch
class Formatter
  def format_as_json(data)
    data.to_json
  end

  def format_as_csv(data)
    data.map(&:values).map { |r| r.join(",") }.join("\n")
  end
end

formatter = Formatter.new
format_type = :json
formatter.send("format_as_#{format_type}", data)

# Dynamic attribute setting
class Config
  attr_accessor :host, :port, :timeout

  def configure(options = {})
    options.each do |key, value|
      setter = "#{key}="
      send(setter, value) if respond_to?(setter)
    end
  end
end

config = Config.new
config.configure(host: "localhost", port: 3000, timeout: 30)
```

---

## respond_to_missing?

Always define this alongside `method_missing`.

```ruby
class Proxy
  def initialize(target)
    @target = target
  end

  def method_missing(name, *args, &block)
    if @target.respond_to?(name)
      puts "Delegating #{name} to target"
      @target.send(name, *args, &block)
    else
      super
    end
  end

  # Must define this too!
  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private) || super
  end
end

proxy = Proxy.new("Hello World")
proxy.upcase      # => "HELLO WORLD"
proxy.length      # => 11
proxy.respond_to?(:upcase)  # => true (works correctly)
```

---

## Open Classes & Monkey Patching

In Ruby, any class can be reopened and modified — including built-ins.

```ruby
# Adding methods to built-in classes
class Integer
  def factorial
    return 1 if self <= 1
    self * (self - 1).factorial
  end

  def times_do
    i = 0
    while i < self
      yield(i)
      i += 1
    end
  end

  def to_roman
    # ... roman numeral conversion
  end
end

5.factorial   # => 120
4.factorial   # => 24

class String
  def palindrome?
    self == self.reverse
  end

  def word_count
    split.count
  end

  def truncate(max_length, suffix: "...")
    return self if length <= max_length
    self[0, max_length - suffix.length] + suffix
  end
end

"racecar".palindrome?      # => true
"hello world".word_count   # => 2
"Long text here".truncate(10)  # => "Long te..."

# Using Refinements (safer monkey-patching, Ruby 2.0+)
module StringExtensions
  refine String do
    def shout
      upcase + "!!!"
    end
  end
end

# Only active within the file/class that uses it
using StringExtensions
"hello".shout   # => "HELLO!!!"
```

---

## Hooks: included, extended, inherited

```ruby
module Callbacks
  def self.included(base)
    puts "#{self} was included into #{base}"
    base.extend(ClassMethods)     # add class methods when included
    base.include(InstanceMethods) # add instance methods
  end

  module ClassMethods
    def all
      @instances ||= []
    end
  end

  module InstanceMethods
    def initialize(*)
      super
      self.class.all << self
    end
  end
end

class User
  include Callbacks
  # When included, User gets both class and instance methods
end

# inherited hook
class Base
  def self.inherited(subclass)
    puts "#{subclass} just inherited from #{self}"
    @subclasses ||= []
    @subclasses << subclass
  end

  def self.subclasses
    @subclasses || []
  end
end

class Admin < Base;  end  # => "Admin just inherited from Base"
class Guest < Base;  end  # => "Guest just inherited from Base"
Base.subclasses  # => [Admin, Guest]
```

---

## class_eval & instance_eval

```ruby
# class_eval — execute code in context of a class
String.class_eval do
  def yell
    upcase + "!"
  end
end
"hello".yell  # => "HELLO!"

# Define methods on specific instances with instance_eval
obj = Object.new
obj.instance_eval do
  def hello
    "Hello from this specific object!"
  end
end
obj.hello  # => "Hello from this specific object!"

# DSL-style builders use this pattern
class HtmlBuilder
  def initialize(&block)
    @html = ""
    instance_eval(&block) if block_given?
  end

  def div(content, **attrs)
    attr_str = attrs.map { |k,v| " #{k}=\"#{v}\"" }.join
    @html += "<div#{attr_str}>#{content}</div>"
  end

  def to_s
    @html
  end
end

builder = HtmlBuilder.new do
  div("Hello", class: "greeting")
  div("World", id: "main")
end
puts builder  # => <div class="greeting">Hello</div><div id="main">World</div>
```

---

## Object Model Recap

```ruby
# Everything is an object
1.class          # => Integer
"hello".class    # => String
nil.class        # => NilClass
true.class       # => TrueClass
:sym.class       # => Symbol
[].class         # => Array

# Method lookup chain
class MyClass
  include MyModule
end
# Lookup: MyClass → MyModule → Object → Kernel → BasicObject

# Introspection
MyClass.ancestors         # full lookup chain
MyClass.instance_methods(false)  # only methods defined in MyClass
obj.methods               # all methods available on obj
obj.instance_variables    # list of instance vars

# Check class hierarchy
1.is_a?(Integer)           # => true
1.is_a?(Numeric)           # => true (Integer inherits from Numeric)
1.is_a?(Comparable)        # => true (Numeric includes Comparable)
1.kind_of?(Integer)        # => true (alias)
1.instance_of?(Integer)    # => true (exact class only)
1.instance_of?(Numeric)    # => false!
```

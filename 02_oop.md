# Ruby Fundamentals — 02: Object-Oriented Programming

## Table of Contents
- [Classes & Objects](#classes--objects)
- [Inheritance](#inheritance)
- [Modules & Mixins](#modules--mixins)
- [Access Control](#access-control)
- [attr_accessor / attr_reader / attr_writer](#attr_accessor--attr_reader--attr_writer)
- [Class Methods vs Instance Methods](#class-methods-vs-instance-methods)
- [Duck Typing](#duck-typing)
- [Comparable & Enumerable](#comparable--enumerable)

---

## Classes & Objects

```ruby
class Animal
  # Class variable (shared)
  @@count = 0

  # Constructor
  def initialize(name, sound)
    @name  = name    # instance variable
    @sound = sound
    @@count += 1
  end

  # Instance method
  def speak
    "#{@name} says #{@sound}!"
  end

  # Class method
  def self.count
    @@count
  end

  # to_s defines how object prints
  def to_s
    "Animal(#{@name})"
  end

  # == lets us compare objects
  def ==(other)
    other.is_a?(Animal) && @name == other.name
  end
end

dog = Animal.new("Rex", "Woof")
cat = Animal.new("Whiskers", "Meow")

dog.speak          # => "Rex says Woof!"
Animal.count       # => 2
puts dog           # => "Animal(Rex)"
```

---

## Inheritance

```ruby
class Animal
  def initialize(name)
    @name = name
  end

  def breathe
    "#{@name} breathes air"
  end

  def speak
    raise NotImplementedError, "#{self.class} must implement speak"
  end
end

class Dog < Animal
  def initialize(name, breed)
    super(name)              # calls parent initialize
    @breed = breed
  end

  def speak
    "#{@name} barks: Woof!"
  end

  def fetch(item)
    "#{@name} fetches the #{item}!"
  end
end

class Cat < Animal
  def speak
    "#{@name} meows: Meow!"
  end
end

rex = Dog.new("Rex", "Labrador")
rex.speak      # => "Rex barks: Woof!"
rex.breathe    # => "Rex breathes air"  (inherited)
rex.is_a?(Dog)    # => true
rex.is_a?(Animal) # => true
rex.class         # => Dog
rex.superclass    # NoMethodError — superclass is a class method
Dog.superclass    # => Animal

# Check ancestry
Dog.ancestors  # => [Dog, Animal, Object, Kernel, BasicObject]
```

---

## Modules & Mixins

> Modules solve the "multiple inheritance" problem and provide namespace separation.

```ruby
# Module as a Mixin (shared behavior)
module Swimmable
  def swim
    "#{name} is swimming!"
  end
end

module Flyable
  def fly
    "#{name} is flying!"
  end
end

module Walkable
  def walk
    "#{name} is walking!"
  end
end

class Duck
  include Walkable      # mixin — adds instance methods
  include Swimmable
  include Flyable

  attr_reader :name

  def initialize(name)
    @name = name
  end
end

class Penguin
  include Walkable
  include Swimmable     # penguins can't fly!

  attr_reader :name

  def initialize(name)
    @name = name
  end
end

donald = Duck.new("Donald")
donald.swim  # => "Donald is swimming!"
donald.fly   # => "Donald is flying!"

skipper = Penguin.new("Skipper")
skipper.swim  # => "Skipper is swimming!"
skipper.fly   # NoMethodError!

# Module as Namespace
module Payments
  class Invoice
    def generate
      "Generating invoice..."
    end
  end

  class Receipt
    def generate
      "Generating receipt..."
    end
  end
end

inv = Payments::Invoice.new
inv.generate  # => "Generating invoice..."

# extend adds module methods as CLASS methods
module ClassMethods
  def description
    "I am the #{name} class"
  end
end

class MyClass
  extend ClassMethods
end

MyClass.description  # => "I am the MyClass class"
```

### include vs extend vs prepend

```ruby
module Greetable
  def greet
    "Hello from #{self.class}"
  end
end

class Person
  include Greetable    # adds as instance methods
end

class Robot
  extend Greetable     # adds as class methods
end

class Alien
  prepend Greetable    # inserts BEFORE the class in method lookup
end

Person.new.greet   # instance method ✓
Robot.greet        # class method ✓
Alien.new.greet    # instance method, but Greetable is checked first ✓
```

---

## Access Control

```ruby
class BankAccount
  def initialize(balance)
    @balance = balance
  end

  # PUBLIC — callable from anywhere
  def deposit(amount)
    @balance += validate_amount(amount)
  end

  def balance
    @balance
  end

  # PROTECTED — callable from within class hierarchy
  protected

  def transfer_to(other_account, amount)
    @balance -= amount
    other_account.receive(amount)
  end

  # PRIVATE — only callable within the object itself
  private

  def validate_amount(amount)
    raise ArgumentError, "Amount must be positive" if amount <= 0
    amount
  end

  def receive(amount)
    @balance += amount
  end
end

acc = BankAccount.new(1000)
acc.deposit(500)      # ✓ public
acc.balance           # ✓ public (1500)
acc.validate_amount(100)  # NoMethodError — private!

# private_method_defined? for checking
BankAccount.private_method_defined?(:validate_amount)  # => true
```

---

## attr_accessor / attr_reader / attr_writer

```ruby
class User
  attr_accessor :name, :email    # getter + setter
  attr_reader   :id              # getter only
  attr_writer   :password        # setter only

  def initialize(id, name, email)
    @id    = id
    @name  = name
    @email = email
  end
end

# attr_accessor generates this automatically:
# def name; @name; end
# def name=(val); @name = val; end

user = User.new(1, "Alice", "alice@example.com")
user.name          # => "Alice"
user.name = "Bob"  # ✓
user.id            # => 1
user.id = 2        # NoMethodError — reader only!
user.password = "secret"  # ✓
user.password      # NoMethodError — writer only!
```

---

## Class Methods vs Instance Methods

```ruby
class Product
  @@all = []

  attr_reader :name, :price

  def initialize(name, price)
    @name  = name
    @price = price
    @@all << self
  end

  # INSTANCE method — called on an instance
  def discounted_price(percent)
    @price * (1 - percent / 100.0)
  end

  def to_s
    "#{@name} ($#{@price})"
  end

  # CLASS methods — called on the class itself
  def self.all
    @@all
  end

  def self.find_by_name(name)
    @@all.find { |p| p.name == name }
  end

  def self.most_expensive
    @@all.max_by(&:price)
  end

  # Alternative: define multiple class methods in a block
  class << self
    def count
      @@all.count
    end

    def reset!
      @@all = []
    end
  end
end

laptop = Product.new("Laptop", 999)
phone  = Product.new("Phone", 699)

laptop.discounted_price(10)  # => 899.1 (instance method)
Product.all                   # => [laptop, phone] (class method)
Product.most_expensive        # => laptop
Product.count                 # => 2
```

---

## Duck Typing

> "If it walks like a duck and quacks like a duck, it's a duck."  
> Ruby cares about what an object *can do*, not what it *is*.

```ruby
class Duck
  def quack
    "Quack!"
  end
end

class Person
  def quack
    "I'm quacking like a duck!"
  end
end

class RubberDuck
  def quack
    "Squeak!"
  end
end

def make_it_quack(duck)
  duck.quack     # doesn't check type, just calls the method
end

make_it_quack(Duck.new)       # => "Quack!"
make_it_quack(Person.new)     # => "I'm quacking like a duck!"
make_it_quack(RubberDuck.new) # => "Squeak!"

# respond_to? — check capability without caring about class
def process(obj)
  if obj.respond_to?(:to_str)
    puts "Processing string: #{obj}"
  elsif obj.respond_to?(:each)
    obj.each { |item| puts "Item: #{item}" }
  end
end
```

---

## Comparable & Enumerable

```ruby
# Include Comparable to get <, <=, >, >=, between?, clamp
class Temperature
  include Comparable

  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees
  end

  # Only need to define <=>
  def <=>(other)
    degrees <=> other.degrees
  end
end

temps = [Temperature.new(30), Temperature.new(15), Temperature.new(25)]
temps.min         # Temperature(15)
temps.max         # Temperature(30)
temps.sort        # sorted array
temp = Temperature.new(20)
temp.between?(Temperature.new(15), Temperature.new(25))  # => true

# Include Enumerable to get map, select, reject, find, etc.
class NumberCollection
  include Enumerable

  def initialize(numbers)
    @numbers = numbers
  end

  # Only need to define each
  def each(&block)
    @numbers.each(&block)
  end
end

nc = NumberCollection.new([1, 2, 3, 4, 5])
nc.select(&:odd?)        # => [1, 3, 5]
nc.map { |n| n * 2 }     # => [2, 4, 6, 8, 10]
nc.min                   # => 1
nc.sum                   # => 15
nc.sort                  # => [1, 2, 3, 4, 5]
```

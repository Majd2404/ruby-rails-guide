# Ruby Fundamentals — 04: Collections (Array, Hash, Enumerable)

## Table of Contents
- [Array](#array)
- [Hash](#hash)
- [Enumerable Module](#enumerable-module)
- [Common Patterns](#common-patterns)

---

## Array

```ruby
# Creation
arr = [1, 2, 3, 4, 5]
arr = Array.new(5, 0)       # => [0, 0, 0, 0, 0]
arr = Array.new(5) { |i| i * 2 }  # => [0, 2, 4, 6, 8]
words = %w[apple banana cherry]    # => ["apple", "banana", "cherry"]
syms  = %i[foo bar baz]            # => [:foo, :bar, :baz]

# Accessing
arr = [10, 20, 30, 40, 50]
arr[0]        # => 10
arr[-1]       # => 50 (last)
arr[1..3]     # => [20, 30, 40]
arr[1, 3]     # start at index 1, take 3 items => [20, 30, 40]
arr.first     # => 10
arr.last      # => 50
arr.first(2)  # => [10, 20]
arr.last(2)   # => [40, 50]
arr.sample    # => random element
arr.sample(2) # => 2 random elements

# Modifying
arr.push(60)     # add to end     => [10,20,30,40,50,60]
arr << 70        # same as push   => [10,20,30,40,50,60,70]
arr.pop          # remove from end, returns it
arr.unshift(0)   # add to front
arr.shift        # remove from front, returns it
arr.insert(2, 99)# insert 99 at index 2
arr.delete(30)   # delete by value
arr.delete_at(1) # delete by index
arr.compact      # remove nils (non-destructive)
arr.compact!     # remove nils (destructive)
arr.flatten      # flatten nested arrays
arr.uniq         # remove duplicates
arr.uniq!        # in-place

# Stack (LIFO)
stack = []
stack.push("first")
stack.push("second")
stack.pop   # => "second"

# Queue (FIFO)
queue = []
queue.push("first")
queue.push("second")
queue.shift  # => "first"

# Sorting
[3,1,4,1,5,9].sort                    # => [1,1,3,4,5,9]
[3,1,4,1,5,9].sort { |a,b| b <=> a } # => [9,5,4,3,1,1] (descending)
words.sort_by { |w| w.length }        # sort by length
words.sort_by(&:length)               # same (shorthand)
words.min_by(&:length)                # shortest
words.max_by(&:length)                # longest

# Transformation
[1,2,3].map { |n| n * 2 }      # => [2,4,6]
[1,2,3].map { |n| n.to_s }     # => ["1","2","3"]
[1,2,3].map(&:to_s)             # same shorthand

# Filtering
[1,2,3,4,5].select { |n| n.odd? }   # => [1,3,5]
[1,2,3,4,5].select(&:odd?)           # same
[1,2,3,4,5].reject { |n| n.odd? }   # => [2,4]
[1,2,3,4,5].filter_map { |n| n * 2 if n.odd? }  # => [2,6,10]

# Aggregation
[1,2,3,4,5].sum                          # => 15
[1,2,3,4,5].reduce(:+)                   # => 15
[1,2,3,4,5].inject(0) { |sum, n| sum + n } # => 15
[1,2,3,4,5].reduce { |product, n| product * n }  # => 120

# Group and partition
[1,2,3,4,5,6].partition(&:even?)      # => [[2,4,6], [1,3,5]]
[1,2,3,4,5,6].group_by(&:even?)       # => {true=>[2,4,6], false=>[1,3,5]}
[1,2,3,4,5].each_slice(2).to_a        # => [[1,2],[3,4],[5]]
[1,2,3,4,5].each_cons(3).to_a         # => [[1,2,3],[2,3,4],[3,4,5]]

# Flattening
[[1,2],[3,[4,5]]].flatten     # => [1,2,3,4,5]
[[1,2],[3,[4,5]]].flatten(1)  # => [1,2,3,[4,5]] (one level only)
[1,[2,[3]]].flat_map { |x| x }  # flatten + map

# Zip and product
[1,2,3].zip([4,5,6])          # => [[1,4],[2,5],[3,6]]
[1,2].product([3,4])          # => [[1,3],[1,4],[2,3],[2,4]]

# Combination methods
[1,2,3].combination(2).to_a   # => [[1,2],[1,3],[2,3]]
[1,2,3].permutation(2).to_a   # => [[1,2],[1,3],[2,1],[2,3],[3,1],[3,2]]

# Useful predicates
[1,2,3].include?(2)            # => true
[1,2,3].any? { |n| n > 2 }   # => true
[1,2,3].all? { |n| n > 0 }   # => true
[1,2,3].none? { |n| n > 5 }  # => true
[1,2,3].count { |n| n.odd? } # => 2
[1,2,3].tally                  # => {1=>1,2=>1,3=>1}
[1,1,2,3,3].tally              # => {1=>2,2=>1,3=>2}
```

---

## Hash

```ruby
# Creation
h = { name: "Alice", age: 30 }
h = Hash.new(0)   # default value 0 for missing keys

# Accessing
h[:name]          # => "Alice"
h[:missing]       # => nil
h.fetch(:name)    # => "Alice"
h.fetch(:missing, "N/A")  # => "N/A" (default)
h.fetch(:missing) { |k| "#{k} not found" }  # block default

h.key?(:name)     # => true
h.has_key?(:name) # => true (alias)
h.value?("Alice") # => true

# Modifying
h[:email] = "alice@example.com"  # add or update
h.delete(:age)                    # remove key
h.merge({ city: "Paris" })        # returns new hash
h.merge!({ city: "Paris" })       # modifies in place

# Iterating
h.each { |key, value| puts "#{key}: #{value}" }
h.each_pair { |k, v| puts "#{k}=#{v}" }
h.each_key { |k| puts k }
h.each_value { |v| puts v }

# Transformation
h.map { |k, v| [k, v.to_s] }.to_h
h.transform_values { |v| v.to_s }   # Ruby 2.4+
h.transform_keys { |k| k.to_s }     # Ruby 2.5+

# Filtering
h.select { |k, v| v.is_a?(String) }
h.reject { |k, v| v.nil? }
h.filter { |k, v| v > 10 }          # alias for select

# Useful methods
h.keys          # => [:name, :age]
h.values        # => ["Alice", 30]
h.to_a          # => [[:name, "Alice"], [:age, 30]]
h.length        # => 2
h.size          # => 2 (alias)
h.empty?        # => false
h.any? { |k,v| v.nil? }
h.all? { |k,v| !v.nil? }
h.count { |k,v| v.is_a?(Integer) }
h.min_by { |k, v| v }
h.max_by { |k, v| v }
h.sort_by { |k, v| v }              # => sorted array of pairs

# Merging with conflict resolution
a = { x: 1, y: 2 }
b = { y: 3, z: 4 }
a.merge(b)                          # => {x:1, y:3, z:4} (b wins)
a.merge(b) { |key, old, new_val| old + new_val }  # custom merge

# Default value hash (great for counting)
word_count = Hash.new(0)
"hello world hello ruby".split.each do |word|
  word_count[word] += 1
end
# => {"hello"=>2, "world"=>1, "ruby"=>1}

# Nested hash access
config = { database: { host: "localhost", port: 5432 } }
config[:database][:host]         # => "localhost"
config.dig(:database, :host)     # => "localhost" (safe dig)
config.dig(:cache, :host)        # => nil (no error!)

# Slicing (Ruby 2.5+)
h = { a: 1, b: 2, c: 3, d: 4 }
h.slice(:a, :c)  # => {a: 1, c: 3}
```

---

## Enumerable Module

```ruby
# Key Enumerable methods with examples
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]
words   = ["the", "quick", "brown", "fox"]

# --- Finding ---
numbers.find { |n| n > 4 }          # => 5 (first match)
numbers.detect { |n| n > 4 }        # alias
numbers.find_index { |n| n > 4 }    # => 4 (index of first match)

# --- Sorting ---
words.sort                           # alphabetical
words.sort_by { |w| w.length }       # by length
words.sort_by { |w| [-w.length, w] } # by length desc, then alpha asc
words.min_by { |w| w.length }        # "the"
words.max_by { |w| w.length }        # "quick" or "brown"
words.minmax_by { |w| w.length }     # ["the", "quick"]

# --- Reducing ---
numbers.reduce(:+)                   # sum
numbers.reduce(:*)                   # product
numbers.inject(100) { |memo, n| memo + n }

# --- Grouping ---
numbers.group_by { |n| n % 3 }
# => {0=>[3,9,6,3], 1=>[1,1,4,1], 2=>[5,2,5,5]}

numbers.tally
# => {3=>2, 1=>3, 4=>1, 5=>3, 9=>1, 2=>1, 6=>1}

numbers.chunk { |n| n.even? }.to_a
# groups consecutive elements by block result

numbers.each_with_object([]) { |n, arr| arr << n * 2 if n > 4 }
# => [10, 9, 12, 10, 10] (alternative to select+map)

# --- Flat Structures ---
[[1,2],[3,4]].flat_map { |a| a.map { |n| n * 2 } }  # => [2,4,6,8]

# --- Zipping ---
[1,2,3].zip([4,5,6], [7,8,9])
# => [[1,4,7],[2,5,8],[3,6,9]]

# --- Lazy Enumeration --- (for infinite or large collections)
(1..Float::INFINITY).lazy
  .select { |n| n.odd? }
  .map { |n| n ** 2 }
  .first(5)
# => [1, 9, 25, 49, 81]  — stops after finding 5, doesn't run forever
```

---

## Common Patterns

```ruby
# Count occurrences
votes = ["alice", "bob", "alice", "charlie", "alice", "bob"]
votes.tally                        # => {"alice"=>3, "bob"=>2, "charlie"=>1}
votes.group_by(&:itself).transform_values(&:count)  # same

# Find duplicates
arr = [1, 2, 3, 2, 4, 3]
arr.tally.select { |_, count| count > 1 }.keys  # => [2, 3]

# Flatten nested array of hashes
users = [{ name: "Alice", tags: ["ruby", "rails"] },
         { name: "Bob",   tags: ["python", "ruby"] }]
users.flat_map { |u| u[:tags] }.uniq  # => ["ruby", "rails", "python"]

# Build lookup hash from array
users = [{ id: 1, name: "Alice" }, { id: 2, name: "Bob" }]
lookup = users.index_by { |u| u[:id] }
# => {1=>{id:1, name:"Alice"}, 2=>{id:2, name:"Bob"}}
lookup[1][:name]  # => "Alice"

# Transform array to hash
keys   = [:name, :age, :city]
values = ["Alice", 30, "Paris"]
keys.zip(values).to_h           # => {name:"Alice", age:30, city:"Paris"}

# Chunk consecutive
[1,1,2,3,3,3,4].chunk_while { |a,b| a == b }.to_a
# => [[1,1],[2],[3,3,3],[4]]

# Deep transform
nested = { a: { b: { c: 42 } } }
nested.dig(:a, :b, :c)   # => 42

# Safe Array() conversion
Array(nil)      # => []
Array([1,2,3])  # => [1,2,3]
Array(42)       # => [42]
# Useful when you don't know if value is array or single item
```

---
layout: post
title: "Applicative programming in Ruby: advanced behaviors"
date: 2023-01-10 09-00-00
permalink: 'applicative-ruby-advanced-behavior'
description: 'Various applicative functor implementations in Ruby'
tags: ruby functional-programming
---

**[Part 1](/applicative-ruby-railway) \| Part 2**

In the previous [post](/applicative-ruby-advanced-behavior) we implemented safe currying on applicative functors. We used a container called `Either` to create these interfaces. In this post we will go even further and create applicative lists and parsers.


## Applicative array

Let's take a look how functors can be implemented for lists:

```ruby
class ApplicativeArray < Array
  include Functor
  def fmap(&fn) = map(&fn.curry)

  include Applicative
  class << self
    def pure(x) = ApplicativeArray.new([x])
  end

  def ^(other)
    ApplicativeArray.new(flat_map { |fn| other.fmap(&fn) })
  end
end
```

`Functor#fmap` is almost a regular map, but it uses `curry` call functions inside the list partially. `pure` takes the value and wraps it into the array. Application (`^`) takes function from another array and applies it to each element of the current array. Let's try it:

```ruby
def plus(x, y) = x + y
def mult(x, y) = x * y

array_with_functions = ApplicativeArray.new([method(:plus), method(:mult)])
array_with_args = ApplicativeArray.new([2, 7])
array_with_args_2 = ApplicativeArray.new([3, 5])

array_with_functions ^ array_with_args ^ array_with_args_2
# => [5, 7, 10, 12, 6, 10, 21, 35]
```

Why 8 elements?

1. `plus` is applied to two elements;
2. `mult` is applied to same elements, getting 4 partially applied functions;
3. these functions are applied to each of two elements getting 8 elements in total.

Can we apply functions to the corresponding element only? Sure, in Haskell this type of the list is called `ZipList`:

```ruby
class Applicative::ZipList
  attr_reader :list

  def initialize(list) = @list = list

  include Applicative::Functor
  def fmap(&fn) = @list.map(&fn.curry)

  include Applicative::ApplicativeFunctor

  class << self
    def pure(x) = Applicative::ZipList.new(repeatedly { x })

    private

    def repeatedly(&block)
      Enumerator.new do |y|
        loop { y << block.call }
      end.lazy
    end
  end

  def ^(other)
    eager_list =
      if @list.is_a?(Enumerator::Lazy) && other.list.is_a?(Enumerator::Lazy)
        @list
      else
        @list.take(other.list.length)
      end

    Applicative::ZipList.new(eager_list.zip(other.list).map { |fn, x| fn.curry.(x) })
  end
end
```

Implementation is slightly more complex, because we have to handle _endless_ lists: we need to do that because `pure` returns it. Here is an example:

```ruby
plus = lambda { |x, y| x + y }
mult = lambda { |x, y| x * y }

functions = ZipList.new([plus, mult])
args = ZipList.new([2, 7])
args_2 = ZipList.new([3, 5])

(functions ^ args ^ args_2).list # => [5, 35]
```

Let's try the same thing but using `pure`, that constructs an endless list for us:

```ruby
functions = ZipList::pure(plus) # endless
args = ZipList::pure(2) # endless
args_2 = ZipList.new([4, 6, 8]) # finite

(functions ^ args ^ args_2).list.eager.to_a # => [6, 8, 10]
```

In this case we get only 3 elements (2 + 4, 2 + 6 and 2 + 8).

### Applicative parser

Let's take a look at more complex application of applicatives: an applicative _parser_. We start with the new data structure called `Pair`, which can hold two values:

```ruby
class Pair
  attr_reader :fst, :snd

  def initialize(fst, snd)
    @fst = fst
    @snd = snd
  end

  def deconstruct = [@fst, @snd]

  include Applicative::Functor
  def fmap(&fn) = Pair.new(fst, fn.(snd))
end
```

It will be a functor, where `fmap` applies function to the _second_ element of the pair, without changing the first one. Now let's define a class that presents a generic parser:

```ruby
class Parser
  def initialize(&parse_fn) = @parse_fn = parse_fn

  def parse(s) = @parse_fn.(s)
end
```

`parse_fn` function should accept and parses some string. When parsing fails it should return `Left` with error message, otherwise — `Right` with the `Pair`. The first element of the `Pair` is the remaining string, the second element is the thing we just parsed. For instance, here is a parser that looks for a character `'x'` in the beginning of the string:

```ruby
x_parser = Parser.new do |s|
  if s.empty?
    Left("unexpected end of input") # because there is no 'x' at the beginning of the empty string
  else
    char = s[0]
    char == 'x' ? Right(Pair.new(s[1..], char)) : Left("unexpected #{char}")
  end
end

x_parser.parse("") # => Left("unexpected end of input")
x_parser.parse("ab") # => Left("unexpected a")
x_parser.parse("xab") # => Right("ab", "x")
```

Let's implement `Functor` for the parser. `fmap` accepts some function that can be applied to the result of the parser and constructs a new parser. Note that we do not parse anything, just build a new parser:

```ruby
class Parser
  include Applicative::Functor

  def fmap(&fn)
    Parser.new do |s|
      parse(s).fmap { |pair| pair.fmap(&fn) }
    end
  end
end

x_parser.fmap(&:upcase).parse("xab") # => Right("ab", "X")
```

Let's add a couple of helpers to build parsers easily:

```ruby
class Parser
  class << self
    def satisfy(&predicate)
      Parser.new do |s|
        if s.empty?
          Left("unexpected end of input")
        else
          char = s[0]
          predicate.(char) ? Right(Pair.new(s[1..], char)) : Left("unexpected #{char}")
        end
      end
    end

    def char(c)
      satisfy { |current| c == current }
    end

    def string(str)
      Parser.new do |s|
        if s.start_with?(str)
          Right(Pair.new(s[str.length..], str))
        else
          Left("unexpected #{s}, expected #{str}")
        end
      end
    end
  end
end
```

Now let's make our parser `Applicative` to make it possible to combine parsers:

```ruby
class Parser
  include Applicative::ApplicativeFunctor

  class << self
    def pure(value)
      Parser.new { |s| Right(Pair.new(s, value)) }
    end
  end

  def ^(other)
    Parser.new do |s|
      case parse(s)
      in Applicative::Either::Right(Pair(s1, g))
        case other.parse(s1)
        in Applicative::Either::Right(Pair(s2, a)) then Right(Pair.new(s2, g.(a)))
        in left then left
        end
      in left then left
      end
    end
  end
end
```

`pure` creates a new parser that always "parses" any value we pass, returning the remaining string as is. Application creates a new parser using following steps:

- if current parser cannot parse a string — we return `Left`;
- otherwise it returns us the remaining string `s1` and the result `a`, we tries to parse `s1` with the second parser:
  - if second parser fails — we return `Left`;
  - otherwise second pair returns us a function `g` (because we're in the `Applicative`), so we return `Right`:
    - the first element is a string that remained from the second parser;
    - the second element is a result of calling `g` with `a`.

This is how it works:

```ruby
parser = (
  Parser.pure(lambda { |a, b| a + b }.curry) ^
    Parser.char('A') ^
    Parser.string("BC")
)

parser.parse("ABC") # => Right ("", "ABC")
```

As you see, `^` operates as the `AND` operator: in this example we ask parser to find us character `A` followed by string `BC`. What about `OR`?

```ruby
module Alternative
  def |(other) = raise NotImplementedError
end

class Parser
  include Alternative

  def |(other)
    Parser.new do |s|
      case parse(s)
      in Applicative::Either::Right(r) then Right(r)
      in _
        case other.parse(s)
        in Applicative::Either::Right(r) then Right(r)
        in left then left
        end
      end
    end
  end
end
```

The new method `|` tries to parse with the first parser, if it fails — with the second, and returns the result.

Let's combine everything together:

```ruby
parser = (
  Parser.pure(lambda { |a, b, c| a + b + c }.curry) ^
    (Parser.char('A') | Parser.char('B')) ^ # we want A or B ...
    Parser.char('C') ^ # ... then C ...
    Parser.string("42") # ... then 42
).fmap(&:downcase) # parser is functor, so `fmap` can be used to transform the result

parser.parse("AC42D") # => #<Either::Right:… @value=#<Pair:… @fst="D", @snd="ac">>
parser.parse("BC42D") # => #<Either::Right:… @value=#<Pair:… @fst="D", @snd="bc">>
parser.parse("DCB") # => #<Either::Left:… @error="unexpected D">
```

Impressive, right?

### Traversable

Let's try to build something _on top_ of applicatives.

![Applicative functor](/assets/traversable.png)

Imagine that you have some applicative container `f` which has a function from `a` to `b` inside. For instance, this function accepts `Either Int` and tries to increment it returning `Either Int`:

```ruby
increment = lambda { |either_value| either_value.fmap { |value| value + 1 } }

increment.(Right(42)) # => Right(43)
increment.(Left("error")) # => Left("error")
```

Then, we have a new module called `Traversable`. It has a single method called `traverse`:

```ruby
module Traversable
  def traverse(applicative_class, &_fn) =
    raise NotImplementedError
end
```

This function accepts some `applicative_class` (which includes `Applicative`) and a function we described above. The result of the call should be the instance of _the same_ `Applicative` that contains the instance of the _the same_ `Traversable`. `Traversable` should contain the value, that was returned by `fn` applied to the data inside the initial traversable.

That sounds smart, so let's just write it and run. Here is the implementation for array:

```ruby
class ApplicativeArray < Array
  include Traversable

  def traverse(applicative_class, &fn)
    return applicative_class::pure([]) if empty?

    x, *xs = self

    applicative_class::pure(lambda { |ta, rest| [ta] + rest }) ^
      fn.(x) ^
      ApplicativeArray.new(xs).traverse(applicative_class, &fn)
  end
end
```

And the example of usage:

```ruby
increment = lambda { |maybe_value| maybe_value.fmap { |value| value + 1 } }

ApplicativeArray.new([Right(1), Right(3), Right(5)])
  .traverse(Either, &increment) # => #<Either::Right:... @value=[2, 4, 6]>

ApplicativeArray.new([Right(1), Left("error")])
  .traverse(Either, &increment) # => #<Either::Left:... @error=“error">
```

Now we can describe the behavior in the easier way: `traverse` goes through some data structure containing values in containers, applies the function to the value in container and _extracts_ container to the top. In our example, list of `Either Int` becomes `Either` list of `Int`:

- if initial list has only `Right` then `Right` will be outside;
- if there are any lefts — the first one will be extracted.

## Outro

In this post we implemented two different applicative functors for lists, created a simple (but yet powerful!) applicative parser and made our lists _traversable_. As you see, applicative functor is a powerful abstraction that helps to describe complex behavior and hide it inside the container.

Hopefully one day we will make something out of it!

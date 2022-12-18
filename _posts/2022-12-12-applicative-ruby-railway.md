---
layout: post
title: "Applicative programming in Ruby: railway reimagined"
date: 2022-12-12 12-00-00
permalink: 'applicative-ruby-railway'
description: 'How to write Railway–style code in Ruby with the Applicative functors'
tags: ruby functional-programming
---

In this post we will see how applicative programming can be used for implementing code in the Railway style using a gem [applicative-rb](https://github.com/DmitryTsepelev/applicative-rb).

---

Railway programming is a common patten for scenarios when you have a series of steps, which are executed in a given order. When everything is good—you get some kind of successful result, if any step fails—you don't do the rest and return the failure to the caller.

For instance, imagine a service that process an order:

- we deduct money from user account;
- we make sure that we have a particular item in stock;
- we update order status.

This is how we can do it with a DSL similar to well–known [dry-monads](https://github.com/dry-rb/dry-monads):

```ruby
class ProcessOrder
  def initialize(order) = @order = order

  def perform
    result = ApplicationRecord.transaction do # start the transaction
      deduct_from_user_account.bind {
        prepare_shipment.bind { # we get here only when previous step returned success
          update_order_status
        }
      }.tap do |result|
        # rollback in case of failure
        raise ActiveRecord::Rollback.new(result.failure) if result.failure?
      end
    end
  end

  private

  def deduct_from_user_account
    if @order.user.balance > @order.amount
      @order.user.deduct_amount(@order.amount)
      Right() # success
    else
      Left("cannot deduct #{@order.amount}, user has #{@order.user.balance}") # failure
    end
  end

  def prepare_shipment
    @order.item_id == 42 ? Success() : Failure("not enough items in warehouse")
  end

  def update_order_status
    @order.processed!
    Left()
  end
end
```

As you can see, each _step_ should return either success (`Right`) or failure (`Left`). The behaviour itself is depend on the container. The container itself is called [Either](https://github.com/DmitryTsepelev/applicative-rb/blob/master/lib/applicative/either.rb):

```ruby
class Either
  class Left < Either
    attr_reader :error

    def initialize(error) = @error = error
    def deconstruct = [@error]
  end

  class Right < Either
    attr_reader :value

    def initialize(value) = @value = value
    def deconstruct = [@value]
  end
end
```

`Right` means a successful result and might hold container some value, `Left` means some failure with the error inside. As you see, it's very similar to well–known `Maybe`, but can hold a value inside.

Let's try it with some real world example, where `fetch_email` tries to get the email by user ID:

```ruby
def fetch_email(user_id)
  if user_id == 42
    Either::Right.new("john.doe@example.com")
  else
    Either::Left.new("User #{user_id} not found")
  end
end

def format_email(either_email)
  case either_email
  in Either::Right(email) then Either::Right.new("Email: #{email}")
  in left then left
  end
end

format_email(fetch_email(42)) # => #<Either::Right:… @value="Email: john.doe@example.com">
format_email(fetch_email(1)) # => #<Either::Left:… @error="User 1 not found">
```

How to work with the value inside the container? You have to unpack it if possible (there's no need to change the error value), work with value and pack back. Looks like there will be a lot of code where we pack and unpack things! Can we avoid it?

## Functors

There is a nice abstraction for it called _functor_:

```ruby
module Functor
  # (a -> b) -> f a -> f b
  def fmap(&_fn) = raise NotImplementedError
end
```

Functor interface has one function, we will call it `fmap`, like it's called in Haskell. Ruby does not have types, so we cannot add a signature to the code, so let's do here: `(a -> b) -> f a -> f b`.

![Functor](/assets/functor.png)

In other words, it should take the function from type `a` to type `b`, a value `a` packed to functor `f`, call the function and return `b` packed to `f`.

> Read more about Functors in my [Haskell post](https://dmitrytsepelev.dev/haskell-adventures-functors)

Here is the example implementation for Either:

```ruby
class Either
  # ...

  include Functor
  def fmap(&fn)
    case self
    in Either::Right(value) then Either::Right.new(fn.(value))
    in left then left
    end
  end
end
```

With this function `format_email` can be simplified:

```ruby
def fetch_email(user_id)
  if user_id == 42
    Either::Right.new("john.doe@example.com")
  else
    Either::Left.new("User #{user_id} not found")
  end
end

def format_email(either_email) = either_email.fmap { |email| "Email: #{email}" }
```

Proper functor should follow some rules:

1. If `identity` (`def identity(value) = value`) is passed, than it should return value without changes;
2. `fmap (f . g) == fmap f . fmap g`, where `.` is a [composition](https://en.wikipedia.org/wiki/Function_composition_(computer_science)) of functions.

## Applicative functors

It works great when there is a value in the container and a function with one argument, but what if there will be a function with more arguments?

Thanks to [curry](https://www.rubydoc.info/stdlib/core/Method:curry), functions can take less arguments and return another function:

```ruby
def sum(x, y) = x + y

Either::Right.new(42).fmap(&method(:sum))
# => #<Either::Right:... @value=#<Proc:... (lambda)>>
```

We can call this function with another argument and get the result:

```ruby
Either::Right.new(42).fmap(&method(:sum)).value.(1) # => 43
```

How to make it more readable?

Applicative functor adds two more methods:

- `pure` that returns _most simple_ container with value;
- `^` takes the function from the first container and applies it to the value stored in the second container.

This is how interface looks like:

```ruby
module Applicative
  include Functor

  def self.included(klass)
    # a -> f a
    klass.extend(Module.new do
      def pure(_value) = raise NotImplementedError
    end)
  end

  # a -> f a
  def pure(value) = self.class.pure(value)

  # f (a -> b) -> f a -> f b
  def ^(_other) = raise NotImplementedError
end
```

Let's try to use it for `Either`:

```ruby
class Either
  # ...
  include Applicative
  def self.pure(value) = Right.new(value)

  def ^(other)
    case self
    in Right(fn) then other.fmap(&fn)
    in left then left
    end
  end
end
```

Note that things will happen only if both containers are `Right`, otherwise it will just keep the error.

Let's rewrite `format_email` one more time:

```ruby
def format_email(either_email)
  add_label = lambda { |label, email| "#{label}: #{email}" }
  Either.pure(add_label) ^ Either::Right.new("Email") ^ either_email
end
```

Why is it useful? It makes `curry` "safe", because we can describe a "golden path". Error will be propagated because of the container _semantics_: we agreed that `Left` should be kept as is without changes. In other words, if you have some steps connected with `^` and one of them returns `Left`, all steps to the right won't even happen and the `Left` will be returned.

> If it's stilly blurry—check out this [post](https://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html) with pictures

However, there is a small downside: each argument for this safe currying should be calculated independently, because these calculations cannot see each other.

Applicative functors also have some laws, but they are a bit more complex so we will not discuss them here. Let's assume that all the implementations in the post are valid.

## Service object in applicative style

Let's get back to the service objects. This is what we had in the beginning of the post:

```ruby
class ProcessOrder
  include Dry::Monads[:result]
  def initialize(order) = @order = order

  def perform
    ApplicationRecord.transaction do
      deduct_from_user_account.bind {
        prepare_shipment.bind {
          update_order_status
        }
      }.tap { |result|
        raise ActiveRecord::Rollback.new(result.failure) if result.failure?
      }
    end
  end

  private

  def deduct_from_user_account; … end
  def prepare_shipment; … end
  def update_order_status; … end
end
```

Note that all three actions are completely independent, which makes it the ideal candidate for the applicative approach! Let's update the service object itself:


```ruby
class ProcessOrder < MultiStepService
  def initialize(order) = @order = order

  add_step :deduct_from_user_account
  add_step :prepare_shipment
  add_step :update_order_status

  def deduct_from_user_account; … end
  def prepare_shipment; … end
  def update_order_status; … end
end
```

Now we need to add the base class:

```ruby
def identity(value) = value # the most helpful function ever
def Right(value = method(:identity)) = Either::Right.new(value) # but we need it to make application work
def Left(error = method(:identity)) = Either::Left.new(error)

class MultiStepService
  class << self
    def add_step(step) = steps << step

    def steps = @steps ||= []
  end

  def perform
    ApplicationRecord.transaction do
      self.class.steps.reduce(Right()) {
        |result, step| result ^ send(step)
      }.on_error { |error| raise ActiveRecord::Rollback.new(error) }
    end
  end
end
```

`#add_step` collects a list of methods to be called. `#perform` reduces steps using the `Right()` as the initial value. `Right()` holds the function that just returns a value passed to it (`identity`), which makes the application work: `^` will run steps until we execute them all or get the first error.

You can find the full example [here](https://github.com/DmitryTsepelev/applicative-rb/blob/master/examples/service.rb).

## Monads

We discussed monads briefly in the beginning, and I could not stop myself from implementing monads from scratch here. We are not going to dig dip into the theory, and jump right to the practice. The difference between applicative functors and monads is that monad has access to the data calculated in the previoues steps.

In order to make something monad you need to implement only two methods `return` and `bind`:

```ruby
module Monad
  include Applicative

  def self.included(klass)
    klass.extend(Module.new do
      # a -> m a
      def returnM(value) = pure(value)
    end)
  end

  # m a -> (a -> m b) -> m b
  def bind(&fn) = raise NotImplementedError
end
```

Check out the source [here](https://github.com/DmitryTsepelev/applicative-rb/blob/master/lib/applicative/monad.rb). As you see, `returnM` does the same thing as we did in `Applicative#pure`, while `bind` is more interesting: it accepts the block, calls it with the current value in the container (if it makes sense, as usual), and returns the result.

> We will go with `returnM` cause `return` is a reserved word in Ruby

Note the difference: in the `Applicative#^` we returned the value in the container! However, the block itself should return the value in the container. This sounds complex, so let's see how to do it for `Either`:

```ruby
class Either
  include Monad
  def bind(&fn)
    case self
    in Right(value) then fn.(value)
    in left then left
    end
  end
end
```

Because of that, we can rewrite `fetch_email`. Now it's a bit more complex, because it can also return the invalid email, so we need to validate it before formatting:

```ruby
def fetch_email(user_id)
  case user_id
  when 42 then Right("john.doe@example.com")
  when 666 then Right("invalid")
  else Left("User #{user_id} not found")
  end
end

def validate(email) = email.include?("@") ? Either::returnM(email) : Left("invalid email")
def format_email(email) = Right("Email: #{email}")
```

Now we can write a function that fetches and validates the email:

```ruby
def fetch_validate_and_format(user_id)
  fetch_email(user_id).bind { |email|
    # we get here only if `fetch_email` returned Right, but email is value, not Either!
    validate(email).bind { |validated_email|
      format_email(validated_email)
    }
  }
end

fetch_validate_and_format(42) # => #<Either::Right:… @value="Email: john.doe@example.com">
fetch_validate_and_format(666) # => #<Either::Left:… @error="invalid email">
fetch_validate_and_format(1) # => #<Either::Left:… @error="User 1 not found">
```

## Outro

Monads are kind of extension for Applicatives, and give us more features. Can we use them always? Not really, because it's possible to create more applicatives than monads (check out this [article](https://www.staff.city.ac.uk/~ross/papers/Applicative.pdf) to learn more on that). Also, if the implementation of the applicative functors follow the rules—we can use them to combine more complex and interesting calculations. At the next post we will focus on these features of Applicatives.

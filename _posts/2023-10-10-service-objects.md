---
title: "Service objects in Rails: how to find a mess"
date: 2023-10-10 09:00:00 +0300
permalink: 'service-objects-anti–patterns'
description: 'Examine your service objects and try to find possible issues caused by wrong composition and missing contracts'
tags: ruby rails
---

Service objects are hard. Back in days, the goal was to extract business logic from controllers and models, and, in some cases, turned out to a black hole inside `app/services` folder.

In this post we will discuss two things that are often missing: contracts and composability. After that I will share a list of  anti–patterns I found while working on dozens of Rails applications. We will examine more and less popular gems to see if they help us to avoid these bad practices. The final part is all about diagnosing your code for these problems and fixing them quickly.

Bear in mind that we will talk about pretty standard Rails monoliths with the relational database, key–value storage, background jobs, mailers etc. If you have something complex (e.g., message queues or search engines)—you will be probably able to extend these cases to handle these interactions as well. Also, sometimes service objects are also called interactors or (business) actions—all issues are applicable to anything in this list.

---

## Navigation

- [Contracts](./service-objects-anti–patterns#contracts)
- [Service composition](./service-objects-anti–patterns#service-composition)
  - [Object–relational impedance mismatch](./service-objects-anti–patterns#objectrelational-impedance-mismatch)
  - [Non–atomic actions](./service-objects-anti–patterns#nonatomic-actions)
  - [What's wrong with composition](./service-objects-anti–patterns#whats-wrong-with-composition)
- [Service implementation anti–patterns](./service-objects-anti–patterns#service-implementation-antipatterns)
  - [Nested transactions](./service-objects-anti–patterns#nested-transactions)
  - [Behavior is different depending on how service was called](./service-objects-anti–patterns#behavior-is-different-depending-on-how-service-was-called)
  - [Incompatible transaction levels](./service-objects-anti–patterns#incompatible-transaction-levels)
  - [Non–atomic action inside transactions](./service-objects-anti–patterns#nonatomic-action-inside-transactions)
  - [IO actions inside transaction](./service-objects-anti–patterns#io-actions-inside-transaction)
  - [Two transactions for single controller action](./service-objects-anti–patterns#two-transactions-for-single-controller-action)
- [I'm using a gem for service objects!](./service-objects-anti–patterns#im-using-a-gem-for-service-objects)
  - [interactor](./service-objects-anti–patterns#interactor)
  - [active_interaction](./service-objects-anti–patterns#active_interaction)
  - [mutations](./service-objects-anti–patterns#mutations)
  - [LightService](./service-objects-anti–patterns#lightservice)
  - [Granite](./service-objects-anti–patterns#granite)
  - [dry-*](./service-objects-anti–patterns#dry-)
- [Anyway, are my services fine or not?](./service-objects-anti–patterns#anyway-are-my-services-fine-or-not)

## Contracts

Ruby does not have explicit type annotations out of the box. In simple cases it's easy to guess: what class will be used if the instance called `user`?

However, what if you want to _compose_ your services? In that case you will have to dig up some code or tests to understand types and make sure you handled all possible cases.

> You might be using [rbs](https://github.com/ruby/rbs) or [Sorbet](https://sorbet.org) to get some types. If you use them—let me know if missing contracts are still an issue in your project.

Ideally service objects should somehow specify their input and output types explicitly. Also, it would be really helpful to have the validation of the _returned value_ to make sure that there are no hidden scenarios that return something different.

There is a number of ways to add validation for input parameters, for instance using [Rails Attributes API](https://api.rubyonrails.org/classes/ActiveRecord/Attributes/ClassMethods.html):

```ruby
class SignUp
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :birthday, :date
  validates :birthday, presence: true
end
```

You can also take a look at [dry-initializer](https://github.com/dry-rb/dry-initializer).

Output validation is harder, and it's more dangerous. Take a look at this code:

```ruby
result = PlaceOrder.call(user:)
```

What is `result`? Can this method raise exceptions? We can open up the source code and check, but what if it calls other classes? We will have to reach the maximum depth to get the list of possible results and exceptions. And we will do it every time for every class, because this knowledge is not written anywhere.

> As a homework exercise, try to estimate how much time you team could waste while figuring out these missing contracts.

## Service composition

In functional programming composition is _an act or mechanism to combine simple functions to build more complicated ones_. In other words, you can compose two functions into a new one that runs the first function on the input and passes the value to the second one. You get the same thing as you would get if you call two functions manually.

In object–oriented programming composition means the ability to use one object from another. They say that there is a _has–a assoctiation_ between them.

> Do not confuse it with `has_one` from ActiveRecord.

You can call one service object from another, i.e., _compose_ them like this:

```ruby
class AddItemToCart
  def initialize(user:, item:)
    @user = user
    @item = item
  end

  def call
    user.with_lock do
      # this is composition of services:
      cart = FindOrCreateCart.call(user: @user)

      cart_item = cart.cart_items
        .create_with(quantity: 0)
        .find_or_initialize_by(item: @item)
      cart_item.quantity += 1
      cart_item.save!
    end
  end
end
```

### Object–relational impedance mismatch

[Object–relational impedance mismatch](https://en.wikipedia.org/wiki/Object–relational_impedance_mismatch) is a set of concepts that are similar between object–oriented languages and relational databases, but they work in completely different ways. For instance, objects and tuples both can hold data, but databases do not have encapsulation.

The most important thing for us is the difference in error handling. In the program you can write some logic, raise exceptions and handle them, but in the database there is an additional level—_transactions_. When you start transaction, you can make some changes in the database, and commit or rollback them altogether. This is called _atomicity_, which is represented by **A** in [ACID](https://en.wikipedia.org/wiki/ACID)).

Depending on the [isolation level](https://www.postgresql.org/docs/current/transaction-iso.html), database can behave differently when you make queries. For instance, default isolation level `READ COMMITTED` can return different data when you make the same query two times if something had changed in between, but `READ COMMITTED` will make queries return the same data until the commit.

### Non–atomic actions

Database is not the only thing we use when we write logic: we can also change other data stores (e.g., Redis or Elasticsearch), perform HTTP requests, work with file system and so on. These actions are not managed by transaction—after the rollback, these changes will stay unless we handle it in some way.

### What's wrong with composition

Take a look again at the example above. Do you think it works fine? Maybe, depends on the implementation of the `FindOrCreateCart`. For instance, this is how it could look like:

```ruby
class FindOrCreateCart
  def initialize(user:)
    @user = user
  end

  def call
    user.with_lock do
      cart = user.orders.find_or_create_by(status: :cart)
    end

    Redis.instance.incr("carts", 1)
  end
end
```

Imagine that we use this service directly as well. We need the lock and transaction to make sure that user has only one cart, and we need to increment counter (just for the demonstration purposes).

This service object looks fine as well, this is how it works:

- transaction opens;
- cart item is looked up:
    - if it's found—transaction commits—Redis counter is incremented;
    - if it's not found—insert is attempted:
        - if insert succeeds then transaction commits—Redis counter is incremented;
        - otherwise—rollback error is thrown, transaction is rolled back and exception prevents counter from the incrementation.

However, when it's called as a part of `AddItemToCart`, two bad things can happen.

First one appears when `FindOrCreateCart` goes to one of scenarios when counter is incremented, but `FindOrCreateCart` raises a `Rollback` error after: in this case cart creation will be rolled back, but _counter increment will still happen_.

The second one happens when something causes `user.orders.find_or_create_by(status: :cart)` to raise `Rollback`: in this case the nested `with_lock` block will catch the exception, but _won't rollback the transaction_ because it's not the place where it was open. As a result, everything will be commited!

> Read more about nested transactions issue [here](https://dev.to/gerardosandoval/understanding-activerecord-nested-transactions-3hee)

How could we avoid these issues? Well, for the second one we could add `requires_new: true` to make sure a nested transaction will use the `SAVEPOINT`. The first issue is not trivial: nested service has no way to know that parent transaction was rolled back; parent service has no way to know if nested action needs a specific rollback.

This is the perfect example of services that cannot be composed. One way to fix this issue is to not reuse the code at all: for instance, you can keep everything in controller actions, which will make transaction usage clearer, but I believe that it will lead us to the unmaintainable mess.

Another way is to split service objects into groups: first is used only on top level (opens up transactions, sets locks etc.) and cannot be called from other services; second contains only database actions; third contains only non–atomic actions.

As a summary of this section we can conclude that _services should either be fully composable or composition should be prohibited at all_. In the next section we will discuss more service object anti–patterns, some of them cause actions to be non—composable.

## Service implementation anti–patterns

### Nested transactions

Nested transactions were already illustrated above: this problem happens when one service object opens up the transaction and calls another service that tries to open up transaction as well. This can lead to the situation when nested rollback is just thrown away.

### Behavior is different depending on how service was called

This one was also illustrated above. The symptom of the problem is that action behaves differently depending on whether it was called directly or from another service object.

### Incompatible transaction levels

This is kinda a sub–problem of nested transactions: if some service needs a higher isolation level that a default one and it's called from another action that did not request this isolation level, it will either behave in a wrong way or database error will happen.

I might be wrong, but all modern databases I know does not allow it. However, even if it was possible, imagine the following situation:

```ruby
class ParentService
  def call
    ApplicationRecord.transaction do
      ChildService.call
    end
  end
end

class ChildService
  def call
    ApplicationRecord.transaction(isolation: :repeatable_read) do
      # logic
    end
  end
end
```

`ChildService` works fine when called directly, but when it's called inside `REPEATABLE READ`—it will behave in a wrong way.

### Non–atomic action inside transactions

Non–atomic action is literally anything that changes a state of anything except the database that runs the transaction. When something fails, transaction will be rolled back, but change made by non–atomic action will stay.

For example, take a look at the following service:

```ruby
class RegisterUser
  def initialize(email:)
    @email = email
  end

  def call
    ApplicationRecord.transaction do
      user = User.create!(email: @email)
      UserMailer.welcome(user).deliver_now
      IssueDiscount.perform_later(user:)

      CreateCart.call(user:)
    end
  rescue ActiveRecord::RecordNotUnique
    # user already exists
  end
end
```

This looks kinda fine: if user already exists than we won't sent mail and issue discount. However, what if `CreateCart` fails? In this case user record will not be committed, but email will be sent and job will be enqueued.

After that, later job will raise an error because it won't be able to load (or _deserialize_) user from the database. Moreover, even if transaction will commit, there is a huge chance that background job processor will pick up the job _before_ the commit, leading to the same error. Of course, it will restart and load the user, so job will succeed but there will be a reported error. Pretty nasty bug to investigate later, right?

It worth mentioning that this issue can be obscure when transaction is opened on the upper level. In this case you won't notice the issue unless you examine the whole call stack. One possible sign is that you have atomic and non–atomic action in the same class: unless there's an implicit transaction—there is something going wrong here.

### IO actions inside transaction

This section is dedicated to another similar pattern, but causing different sympthoms is IO actions inside transactions. Imagine a following service:

```ruby
class CheckPayment
  def initialize(order:)
    @order = order
  end

  def call
    ApplicationRecord.transaction do
      response = PaymentProviderClient.check_payment(order_id: order.external_id)

      order.update!(payment_status: :processed) if response[:status] == :processed
    end
  end
end
```

From the business logic perspective it looks fine: we fetch the status from the external API and update our database based on the response. However, what if that API is down?

> Make sure to not make ANY IO actions in the main (i.e., web server) thread cause it can consume all workers and application will be down. This is the anti–pattern itself, not related to the place where you keep the business logic. Prefer background jobs for such things instead.

That will make our transaction longer. Moreover, locks will be held while application waits from the response, preventing other transactions from finishing.

### Two transactions for single controller action

I know there might be exceptions, but most of the time when you have two transactions (not nested) inside the single action it means that something is wrong. What if the second one fails? Should we revert a first one? How?

For instance, imagine the situation, when you need to do 3 things:

1. change database state (transaction 1);
2. fetch some data from HTTP (we already know it should be outside the transaction);
3. change database state one more time depending on the response (transaction 2).

The best way to do that is to run background job after the first transaction and keep steps 2 and 3 there—it will either complete or we will get the exception to our error tracker, fix the problem and re–run the job.

## I'm using a gem for service objects!

In this section we will discuss ways how to implement service objects using existing popular gems and approaches. As you might have noticed, all these anti–patterns can be more or less easily refactored, but can we prevent them by design? Does gem design help to achieve that?

Linters seem to be helpless here because usually they can check only a single file in isolation, there is no way to see if transaction was opened around. As mentioned earlier, we can mitigate these patterns if we do not have any composition, but that might be a bad solution in terms of maintenance.

### interactor

One of most popular solutions is [interactor](https://github.com/collectiveidea/interactor).

> Do not confuse it with Interactor pattern coming from the Clean architecture. By definition from the internet it means "little, reusable chunks of code that abstract logic from presenters while simplifying your app and making future changes effortless". Not related to our topic at all.

A very first thing that's mentioned in the README at the moment of writing is _context_. Effectively it's just a hash that contains all passed arguments, and you can add more if you want. When service runs successfully you get the whole context back, otherwise you get `Interactor::Failure`. You can also use `context.fail!` to halt and cause service to return a failure object.

```ruby
class AuthenticateUser
  include Interactor

  def call
    if user = User.authenticate(context.email, context.password)
      # adding two more variables to the context
      context.user = user
      context.token = user.secret_token
    else
      context.fail!(message: "authenticate_user.failure")
    end
  end
end

# args will go to context
AuthenticateUser.call(email:, password:)
```

The context itself feels a bit dangerous, because it can be used as a replacement of instance variables, which sounds like a breach of encapsulation. Are we sure someone even wants to read these new variables?

Frankly speaking, it took me a second to understand that we need to pass `email` and `password` to make this call. However, just to make things even more hard, let's take a look at [organizers](https://github.com/collectiveidea/interactor#organizers):

```ruby
class PlaceOrder
  include Interactor::Organizer

  organize CreateOrder, ChargeCard, SendThankYou
end
```

What does it accept and return? You will have to go and read specs or code of all three services. Understanding the _contract_ becomes a really hard job.

Let's see what we got for transactions and non–atomic actions. There is an around hook you can use:

```ruby
class PlaceOrder
  include Interactor::Organizer

  organize CreateOrder, ChargeCard, SendThankYou

  around do |interactor|
    ApplicationRecord.transaction { interactor.call }
  end
end
```

I bet `SendThankYou` contains some non–atomic logic. Can we pull it away from the transaction?

```ruby
class PlaceOrder
  include Interactor::Organizer

  organize CreateOrder, ChargeCard

  around do |interactor|
    ApplicationRecord.transaction { interactor.call }

    SendThankYou.call(interactor.context)
  end
end
```

That kinda works, but what if we need to do something non–atomic inside the service in the middle of the chain? That would be really hard because you will have to not forget to pull it away as well. Moreover, in this case `CreateOrder` and `ChargeCard` cannot be used directly without wrapping them into the transaction.

> It worth mentioning, that there is a `rollback` method that can help us to do some cleanup at least.

As a result, _composition_ does not play well here too: we have to prohibit a direct usage of services that not open the transaction for themselves. Also, we should always remember to not add anything non–transactional to these services and have separate services for that.

### active_interaction

The next stop is [active_interaction](https://github.com/AaronLasseigne/active_interaction). Let's rewrite our previous example:

```ruby
class AuthenticateUser < ActiveInteraction::Base
  string :email
  string :password

  def execute
    if user = User.authenticate(email, password)
      { user:, token: user.secret_token }
    else
      fail AuthenticationFailed
    end
  end
end
```

This looks a bit better in terms of contract: we can see that there are two strings expected. The returned value (`{ user:, token: user.secret_token }`) will be placed to the `result` object:

```ruby
outcome = AuthenticateUser.run(email:, password:)
outcome.valid? # => true
outcome.result # => { user:, token: user.secret_token }
```

We still need to read the code to understand, what is returned. Note that there is no validation to make sure that result is always the same. For instance, in some cases we might return only `user`, and the code that uses this service should be aware of it.

What about the composition? According to the README, we can call another service using `compose` and pass arguments explicitly:

```ruby
class PlaceOrder < ActiveInteraction::Base
  object :user

  def execute
    ApplicationRecord.tranasction do
      order = compose(CreateOrder, user:)
      compose(ChargeCard, user:, order:)
    end

    compose(SendThankYou, user:)
    order
  end
end
```

It's better than implicit context and organizer we saw in `interactor`! However, all other problems are still here: we have to split services into ones that can be directly used and manage transactions by ourselves.

### mutations

Let's take a quick look at [mutations](https://github.com/cypriss/mutations):

```ruby
class AuthenticateUser < Mutations::Command
  required do
    string :email, matches: EMAIL_REGEX
    string :password
  end

  def execute
    if user = User.authenticate(email, password)
      { user:, token: user.secret_token }
    else
      :auth_failed
    end
  end
end
```

There is an input validation. The same as `active_interaction` there is an implicit result, but the difference is that there are no other validations except data types, README suggests to do everything manually in the `execute` method.

Composition is done using plain old method calls:

```ruby
class PlaceOrder < Mutations::Command
  required do
    object :user
  end

  def execute
    ApplicationRecord.tranasction do
      order = CreateOrder.execute(user:)
      ChargeCard.execute(user:, order:)
    end

    SendThankYou.execute(user:)

    order
  end
end
```

This approach has same issues as previous solutions we inspected.

### LightService

Another popular gem I found is [LightService](https://github.com/adomokos/light-service):

```ruby
class Transactional
  def self.call(context)
    ApplicationRecord.transaction { yield }
  end
end

class CalculatesTax
  extend LightService::Organizer

  def self.call(order)
    with(order:).with(Transactional).reduce(
        LooksUpTaxPercentageAction,
        CalculatesOrderTaxAction,
        ProvidesFreeShippingAction
      )
  end
end

class LooksUpTaxPercentageAction
  extend LightService::Action
  expects :order
  promises :tax_percentage

  executed do |context|
    tax_ranges = TaxRange.for_region(context.order.region)
    context.tax_percentage = 0

    if object_is_nil?(
      tax_ranges,
      context,
      'The tax ranges were not found'
    )
      next context
    end

    context.tax_percentage = tax_ranges.for_total(context.order.total)

    if object_is_nil?(
      context.tax_percentage,
      context,
      'The tax percentage was not found'
    )
      next context
    end
  end

  def self.object_is_nil?(object, context, message)
    if object.nil?
      context.fail!(message)
      return true
    end

    false
  end
end
```

What we can notice here:

- interactor–like organizers;
- interactor–like around hooks for transactions (or run them manually);
- `promises` to specify the output.

Contract is defined better for separate services, but it's still a challenge to read it for the organizer. Composition has all the issues we saw before.

### Granite

You might be curious how [gem](https://github.com/toptal/granite) with <200 stars appeared here. Take a look:

```ruby
class PlaceOrder < Granite::Action
  subject :user

  private def execute_perform!(*)
    # no need to open transaction — gem will do that for us when needed and use same
    # transaction for other services
    order = create_order_service.perform!
    charge_card_service.perform! # not sure how to pass order here, see below
  end

  after_commit do
    UserMailer.thank_you(user: subject).perform_later
  end

  memoize def create_order_service
    CreateOrder.new(subject)
  end

  memoize def charge_card_service
    ChargeCard.new(subject)
  end
end
```

The thing that's done in a different way is that Granite supports service composition out of the box. As you see, we instantiate other services and call them, the gem handles transactions.

Here comes a problem: gem _requires_ all services to be defined as memoized instances, making it impossible to pass anything that was created by another service. I guess it's possible to implement anything one might need using some code jiggling.

There are many more things in the gem: various hooks, validations, data representers, policies, context, associations, exceptions, I18n and many more. What's missing is the documentation—it's fairly short so you will have to dig into gem source to learn it better. I like this gem way more than others because it has a lot of cool ideas, but writing services this way feels hard: you need to keep in mind too many rules.

### dry-*

Often times people mention [dry stack](https://dry-rb.org) when they are asked about the way they implement service objects. This is a set of gems that can do different things, so the way you cook them really matters. There are a couple of thins to remember if you decide to go that way.

[dry-validation](https://dry-rb.org/gems/dry-validation/1.10/) can be used for the contract of input data, output is not covered by anything if I'm not mistaken.

[dry-transaction](https://dry-rb.org/gems/dry-transaction/0.15/) is about _business_ transactions and does not help with database transactions at all, so you have to manage that manually as discussed in previous solutions.

[dry-monads](https://dry-rb.org/gems/dry-monads/1.6/) are often used as result objects, but there are some common pitfalls. One of the most popular features is do notation that implements a popular [railway pattern](https://blog.logrocket.com/what-is-railway-oriented-programming/):

```ruby
class CreateAccount
  include Dry::Monads[:result]
  include Dry::Monads::Do.for(:call)

  def call(params)
    values = yield validate(params)
    account = yield create_account(values[:account])
    owner = yield create_owner(account, values[:owner])

    Success([account, owner])
  end

  def validate(params)
    # returns Success(values) or Failure(:invalid_data)
  end

  def create_account(account_values)
    # returns Success(account) or Failure(:account_not_created)
  end

  def create_owner(account, owner_values)
    # returns Success(owner) or Failure(:owner_not_created)
  end
end
```

This looks good, but be aware that when `yield Failure` happens the database transaction will be [rolled back](https://dry-rb.org/gems/dry-monads/1.6/do-notation/#transaction-safety). This brings us to the problem of "all or nothing". Imagine that you want to create an order for user from his cart, but also cleanup the cart from items that are out of stock. As a result we have 3 possible scenarios:

- Success when order is created;
- Failure when cart is empty;
- something when cart _became_ empty.

I'd really like to use `Failure(:empty_cart)` but I cannot, because it will roll back my transaction! As a result we have to do a following thing:

```ruby
def call
  ApplicationRecord.transaction do
    yield clear_cart
    yield create_order
  end
end

def clear_cart
  cart.cart_items.filter(&:out_of_stock?).delete_all
end

def create_order
  if cart.cart_items.any?
    order = user.orders.create!(cart)
    Success(:order_created, order:)
  else
    Success(:empty_cart)
  end
end
```

Alternatively, we can stop using do notation or `yield`, but that's what we came for.

One last thing to mention that `Failure` rolls back transaction only when used with `yield`, this [won't work](https://mikey.bike/j/2023/04/dry-rb-monad-laws.html):

```ruby
def call
  ApplicationRecord.transaction do
    yield clear_cart
    create_order
  end
end
```

## Anyway, are my services fine or not?

After reading this bunch of text you might be asking yourself if your services are fine or they need some changes. I prepared a checklist for you.

First of all, if you are not reusing the code at all or have separate classes for actions that happen before and after transactions and never mix them—probably you can stop thinking about composition issues. Otherwise—your services might be affected by some of the related anti–patterns.

The fastest way to check this is to install [isolator](https://github.com/palkan/isolator). It will report cases when someone performs non–atomic action, schedules job or makes HTTP request from inside the transaction. It might find some issues caused by composition. As soon as you get some offences, you can quickly fix some of them by [after_commit_everywhere](https://github.com/Envek/after_commit_everywhere):

```ruby
class SomeService
  include AfterCommitEverywhere

  def call
    ApplicationRecord.transaction do
      # db operations

      after_commit do
        # jobs, mails and etc go here
      end

      # you can continue db operations if you want
    end
  end
end
```

This is not ideal and you might still need to redesign your service object, but at least it will help to fix some nasty bugs right away by _separating flows_.

HTTP queries can be moved away from transaction by putting it before the transaction block:

```ruby
class CheckPayment
  def initialize(order:)
    @order = order
  end

  def call
    response = PaymentProviderClient.check_payment(order_id: order.external_id)

    ApplicationRecord.transaction do
      order.update!(payment_status: :processed) if response[:status] == :processed
    end
  end
end
```

However, it's not always easy, because this service might be called from another one which opened the transaction already. As you might have guessed, the hardest part is nested transactions. I did not found an existing solution, so I prepared a quick and dirty script for you:

```ruby
ApplicationRecord.singleton_class.prepend(Module.new do
  def transaction(**kwargs)
    Thread.current[:transaction_stack] ||= []

    unless kwargs[:requires_new]
      location = caller_locations(1, 1).first

      if Thread.current[:transaction_stack].any?
        puts <<~TEXT
          Found nested transaction in #{location}.

          Opened up in:
          #{Thread.current[:transaction_stack].join("\n")}
        TEXT
      end
    end

    if Thread.current[:transaction_stack].empty? || !kwargs[:requires_new]
      Thread.current[:transaction_stack] << location
    end

    super(**kwargs).tap do
      Thread.current[:transaction_stack].pop
    end
  rescue e
    Thread.current[:transaction_stack] = []
    raise e
  end
end)
```

Put it to the initializer and run your specs—maybe there will be something.

As mentioned, input validation can be handled by [Rails Attributes API](https://api.rubyonrails.org/classes/ActiveRecord/Attributes/ClassMethods.html), [dry-initializer](https://github.com/dry-rb/dry-initializer) or something similar.

---

You might be wondering how to to completely avoid nested transactions, write fully reusable and composable service objects without _any way_ to make it wrong. I'm curious too, stay tuned and I'll share something in the next post!

> If you want to take a look at the thing I'm experimenting with—here [it is](https://github.com/DmitryTsepelev/clean_actions).

At the meantime, take a look at your services, see if you can spot some anti–patterns using your eyes and tools. Check if that causes some bugs or performance issues in your system. Think if you are using composition, try to remember bugs caused by unhandled scenarios after the service call.

Maybe you will be able to solve it by redesigning the class layout. Probably making some new team agreements will help. It might turn out that your services are fine and you don't face any issues at all.



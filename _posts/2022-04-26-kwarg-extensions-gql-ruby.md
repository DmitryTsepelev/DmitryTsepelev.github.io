---
layout: post
title: "How to configure field extensions using keyword arguments in GraphQL Ruby"
date: 2022-04-26 9:00:00 +0300
permalink: 'kwarg-extensions-gql-ruby'
description: 'Learn how to configure field extensions for graphql-ruby using keyword arguments in 3 minutes'
tags: ruby graphql
---

[graphql-ruby](https://graphql-ruby.org) comes with a built–in support for [field extensions](https://graphql-ruby.org/type_definitions/field_extensions.html). You can think of it as it were middlewares: a field that has an extension will be resolved only after passing the extension and only if extension explicitly allows that. An ability to halt the resolution can help a lot, for instance when you want to cache something (when cache is hit there is no need to resolve the field) or check permissions (when check fails—we do not need to continue resolving).

> Have no idea what GraphQL is and why people love it? Check out my [tutorial](https://evilmartians.com/chronicles/graphql-on-rails-1-from-zero-to-the-first-query) about GraphQL in Rails first

Imagine that we have the following schema with a single field we want to cache:

```ruby
class BaseObject < GraphQL::Schema::Object
end

class QueryType < BaseObject
  field :cached_val, String, null: false

  def cached_val
    "I'm cached at #{Time.now}"
  end
end

class GraphqlSchema < GraphQL::Schema
  query QueryType
end
```

Obviously nothing is currently cached, so when we call this field multiple times we will get different responses:

```ruby
query = <<-GQL
  query {
    cachedVal
  }
GQL

puts GraphqlSchema.execute(query).dig("data", "cachedVal") # => I'm cached at 2022-04-07 12:47:22 +0300
sleep 3
puts GraphqlSchema.execute(query).dig("data", "cachedVal") # => I'm cached at 2022-04-07 12:47:26 +0300
```

Here is the plan: when field is called for the first time, we will resolve it and remember the value, and use the cached one next time field is called.

> Our cache implementation will be very naive, please do not do it in your production app 🙂 I cache GraphQL responses using [graphql-ruby-fragment_cache](https://github.com/DmitryTsepelev/graphql-ruby-fragment_cache)

Here is the implementation:

```ruby
class CacheExtension < GraphQL::Schema::FieldExtension
  def resolve(object:, arguments:, **rest)
    key = cache_key(object, arguments)
    store[key] ||= yield(object, arguments)
  end

  private

  def store
    Thread.current[:field_cache] ||= {}
  end

  def cache_key(object, arguments)
    "#{object.class.name}-#{@field}-#{arguments.hash}"
  end
end

class QueryType < BaseObject
  field :cached_val, String, null: false, extensions: [CacheExtension]

  def cached_val
    "I'm cached at #{Time.now}"
  end
end
```

Let's make sure it works (note that response is the same after 3 seconds):

```ruby
puts GraphqlSchema.execute(query).dig("data", "cachedVal") # => I'm cached at 2022-04-07 15:54:57 +0300
sleep 3
puts GraphqlSchema.execute(query).dig("data", "cachedVal") # => I'm cached at 2022-04-07 15:54:57 +0300
```

Looking good, but passing the array to the `:extensions` looks a bit cumbersome, and it will be even less readable when we decide to _configure_ our extensions. For instance, let's change our extension to support TTL (time to keep value cached):

```ruby
class CacheExtension < GraphQL::Schema::FieldExtension
  def resolve(object:, arguments:, **rest)
    key = cache_key(object, arguments)
    cached_value = store[key]

    if cached_value.nil? || ttl_expired?(cached_value)
      resolved_value = yield(object, arguments)
      store[key] = { value: resolved_value, cached_at: Time.now }
      return resolved_value
    end

    cached_value[:value]
  end

  private

  def ttl_expired?(cached_value)
    options[:ttl] && (Time.now - cached_value[:cached_at]) > options[:ttl]
  end

  def store
    Thread.current[:field_cache] ||= {}
  end

  def cache_key(object, arguments)
    "#{object.class.name}-#{@field}-#{arguments.hash}"
  end
end
```

This is how we'll have to configure the extension:

```ruby
class QueryType < BaseObject
  field :cached_val,
        String,
        null: false,
        extensions: [{ CacheExtension => { ttl: 3 } }]

  def cached_val
    "I'm cached at #{Time.now}"
  end
end
```

..and check that it works:

```ruby
puts GraphqlSchema.execute(query).dig("data", "cachedVal") # => I'm cached at 2022-04-07 16:29:50 +0300
sleep 1
puts GraphqlSchema.execute(query).dig("data", "cachedVal") # => I'm cached at 2022-04-07 16:29:50 +0300
sleep 3
puts GraphqlSchema.execute(query).dig("data", "cachedVal") # => I'm cached at 2022-04-07 16:29:54 +0300
```

In my opinion, passing kwargs for each extension would be a bit more readable:

```ruby
field :cached_val, String, null: false, cached: { ttl: 3 }
```

Is it possible? Yes, but we'll need to patch the initializer in our `BaseObject`:

```ruby
class BaseObject < GraphQL::Schema::Object
  def initialize(*args, cached: false, **kwargs, &block)
    if cached.is_a?(Hash)
      extension(CachedExtension, **cached)
    elsif cached
      extension(CachedExtension)
    end

    super(*args, **kwargs, &block)
  end
end
```

It's important to remove our custom keyword argument from `kwargs` to avoid passing it to the `super` call, because parent initializer does not expect it.

Here is a full listing that you can run and play with:

```ruby
require "bundler/inline"

gemfile do
  source "https://rubygems.org"

  gem "graphql"
end

class CacheExtension < GraphQL::Schema::FieldExtension
  def resolve(object:, arguments:, **rest)
    key = cache_key(object, arguments)
    cached_value = store[key]

    if cached_value.nil? || ttl_expired?(cached_value)
      resolved_value = yield(object, arguments)
      store[key] = { value: resolved_value, cached_at: Time.now }
      return resolved_value
    end

    cached_value[:value]
  end

  private

  def ttl_expired?(cached_value)
    options[:ttl] && (Time.now - cached_value[:cached_at]) > options[:ttl]
  end

  def store
    Thread.current[:field_cache] ||= {}
  end

  def cache_key(object, arguments)
    "#{object.class.name}-#{@field}-#{arguments.hash}"
  end
end

class BaseObject < GraphQL::Schema::Object
  def initialize(*args, cached: false, **kwargs, &block)
    if cached.is_a?(Hash)
      extension(CachedExtension, **cached)
    elsif cached
      extension(CachedExtension)
    end

    super(*args, **kwargs, &block)
  end
end

class QueryType < BaseObject
  field :cached_val, String, null: false, cached: { ttl: 3 }

  def cached_val
    "I'm cached at #{Time.now}"
  end
end

class GraphqlSchema < GraphQL::Schema
  query QueryType
end

query = <<-GQL
  query {
    cachedVal
  }
GQL

puts GraphqlSchema.execute(query).dig("data", "cachedVal") # => I'm cached at 2022-04-07 16:29:50 +0300
sleep 1
puts GraphqlSchema.execute(query).dig("data", "cachedVal") # => I'm cached at 2022-04-07 16:29:50 +0300
sleep 3
puts GraphqlSchema.execute(query).dig("data", "cachedVal") # => I'm cached at 2022-04-07 16:29:54 +0300
```

---

That's all for today! Hopefully this approach will help you to write more idiomatic GraphQL code in Ruby.

> Looking for a real world example of this trick? I have [one](https://github.com/DmitryTsepelev/graphql-ruby-fragment_cache/blob/master/lib/graphql/fragment_cache/field_extension.rb#L8) for you

Fun fact: this post is based on my [gist](https://gist.github.com/DmitryTsepelev/065bb6bc796898f5745c4209d1b4bb21) created 3 years ago. At the middle of writing this post I decided to check docs and realized that this trick was added there two months ago, but I decided to keep going, cause my example is different anyway 🙂

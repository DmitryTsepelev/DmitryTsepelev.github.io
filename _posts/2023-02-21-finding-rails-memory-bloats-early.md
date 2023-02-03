---
layout: post
title: "How to find a memory bloat in your Rails app before it happens"
date: 2023-02-21 09-00-00
permalink: 'finding-rails-memory-bloats-early'
description: 'Where Ruby methods keeps methods and variables defined on the top–level scope'
tags: ruby performance
---

Memory bloat in Ruby happens when someone loads a lot of data to the memory. Ruby Virtual Machine does not return most of the allocated memory to the operating system even after data is collected as a garbage. It's not a big deal for local development or console programs, but if you have a bloat in the big Rails app it might cost you a lot of money.

It is quite easy to find a fix memory bloat when it happens (you have a monitoring tool set up, right?), but can we _prevent_ it? In this article I will present a new approach for that, using a new gem [io_monitor](https://github.com/DmitryTsepelev/io_monitor) as a reference implementation.

## Understanding how bloat happens

Imagine a pretty regular Rails app that sometimes goes to the database for data. When we load some data from the database we put it to the memory, and, when we do not have enough memory, Ruby VM goes to the operating system and asks for more. All Ruby objects in memory are organized into _pages_, and Ruby VM can return memory back to the operating system only from _last free pages_. As a result, if your app loaded a lot of data, the pages it was stored at can be returned to the system only if there is nothing that's still in use after them.

I recommend this [post](https://www.joyfulbikeshedding.com/blog/2019-03-14-what-causes-ruby-memory-bloat.html) with lots of pictures to get a better undestanding of this process.

## Why bloat is a bad thing?

Your app will need more resources to run; moreover, if you live in the Kubernetes cluster you'll be able to run less pods on the same machine and can see some restarts when quota is reached.

The memory bloat is easy to find when it happens: most of APMs will show you the place where application performed a lot of allocations, so you just need to go it fix this. Can we do it _before_ it starts being a problem?

## Early bloat detection

In most of cases (well, I do not have statistics except my own experience) memory bloat is caused by I/O operations. You fetch data from the database or Redis, read file from the disk, perform the network request, and unexpectedly big amount of data is about to be allocated in the application memory.

> Of course there is a chance that your app itself creates a lot of objects from the code, but it's a pretty rare thing to happen

The fix is usually trivial: you need to either load data in batches (so memory consumption will be the same as the batch size) or do not load it at all and perform calculations _somewhere else_. For instance, if you need a `sum` of transactions — do `Transaction.sum(:amount)` (which is `SELECT SUM(amount) FROM transactions`) instead of `Transaction.all.sum(&:amount)` (which is `SELECT * FROM transactions` and send it over the wire).

Here is an idea: what if we try to measure the size of data that was loaded from the I/O and compare it to the response size? Even in case if you have 100 transactions to get the sum and your APM keeps silence, this ratio will still be quite big!

For instance, for ActiveRecord you can do something like this:

```ruby
ActiveRecord::ConnectionAdapters::AbstractAdapter.prepend(Module.new do
  def build_result(*args, **kwargs, &block)
    io_bytesize = kwargs[:rows].sum(0) do |row|
      row.sum(0) do |val|
        ((String === val) ? val : val.to_s).bytesize
      end
    end

    Aggregator.instance.increment(io_bytesize)

    super
  end
end)
```

Now we need to subscribe to notifications from controllers for action processing:


```ruby
ActiveSupport::Notifications.subscribe("process_action.action_controller") do |*args|
  io_bytesize = Aggregator.instance.io_bytesize
  body_bytesize = args.last[:response].body.bytesize

  ratio = io_bytesize / body_bytesize.to_f

  Rails.logger.info "Loaded from I/O #{io_bytesize}, response bytesize #{body_bytesize}, I/O to response ratio #{ratio}"
end
```

As a result, we can see messages about actions that have a high ratio in our logs!

> You can find the whole example [here](https://github.com/DmitryTsepelev/io-monitor-demo).

Let's add a very simple controller with two actions — fast and slow:

```ruby
class App < Rails::Application
  routes.append do
    get '/slow', to: 'app#slow'
    get '/fast', to: 'app#fast'
  end
end

class AppController < ActionController::Base
  def slow
    render json: {sum: Transaction.all.sum(&:amount)}
  end

  def fast
    render json: {sum: Transaction.sum(:amount)}
  end
end
```

If you run it — you'll notice the difference in logs:

```
Started GET "/slow" for 127.0.0.1 at 2023-02-04 23:01:27 +0300
Processing by AppController#slow as */*
  Transaction Load (4.5ms)  SELECT "transactions".* FROM "transactions"
Completed 200 OK in 43ms (Views: 0.1ms | ActiveRecord: 11.7ms | Allocations: 95836)
Loaded from I/O 69899, response bytesize 15, I/O to response ratio 4659.933333333333

Started GET "/fast" for 127.0.0.1 at 2023-02-04 23:01:35 +0300
Processing by AppController#fast as */*
  Transaction Sum (3.1ms)  SELECT SUM("transactions"."amount") FROM "transactions"
Completed 200 OK in 4ms (Views: 0.1ms | ActiveRecord: 3.1ms | Allocations: 336)
Loaded from I/O 7, response bytesize 15, I/O to response ratio 0.4666666666666667
```

## io_monitor

If you want to try this approach in your application, you don't need to build this from scratch. Use [io_monitor](https://github.com/DmitryTsepelev/io_monitor)! Right now it supports Redis and HTTP along with ActiveRecord, and can publish to logs, Rails notifications and Prometheus.

Setup is fairly simple. After installation you need to configure what to monitor and where to publish and you're good to go:

```ruby
IoMonitor.configure do |config|
  config.publish = [:logs, :notifications, :prometheus] # defaults to :logs
  config.warn_threshold = 0.8 # defaults to 0
  config.adapters = [:active_record, :net_http, :redis] # defaults to [:active_record]
end
```

After that you need to include it controllers you want to check (or all of them):

```ruby
class MyController < ApplicationController
  include IoMonitor::Controller
end
```

If something is not right you'll see something in logs:

```
ActiveRecord I/O to response payload ratio is 0.1, while threshold is 0.8
```

I never tested it in the real app yet, so please let me know if you notice something weird!

---

In this post we discussed memory bloats: now you can not only notice and fix them, but also try to _prevent_. The main outcome is a bit different though: if you work with I/O — always consider the amount of data you might get and try to minimize this amount.

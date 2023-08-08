---
title: "Ruby goes to the movie theater: directing the refactoring of your application"
date: 2023-09-12 09:00:00 +0300
permalink: 'directing-refactoring'
description: 'How to plan and execute the refactoring of you Ruby application'
tags: ruby rails engineering
---

This post introduces a method I use to refactor big applications. I want the process to happen in a predictable manner and make sure that important things are addressed before others. One day I realized that there is a missing tool in my workflow, so I'll introduce my new gem called [rubocop_director](https://github.com/DmitryTsepelev/rubocop_director).

## Refactoring as a process

Refactoring is a technique of transforming the code without making changes in its functionality. Why people do that?

Sometimes the way that code written makes it harder for to engineers to improve the performance or add a new feature. Another example is a scenario when application is big and there are many ways to do the same thing or same _responsibility_ is implemented in different _layers_ of application.

This is how refactoring looks like in books:

1. the problem is spotted and a better solution is found;
2. add some tests if coverage is not enough;
2. code is rewritten.

> Please, please, do not start refactoring before covering the code with specs!

Eventually it will be done, but a number of issues that might cause issues:

1. there are _too many_ places to fix, so refactoring will take a lot of time to complete;
2. most of the time there are more than one problem that's being refactored at the given moment;
3. people might accidentally write code in the "old style", so amount of work will keep growing.

Clearly, we need a tool to somehow track all occurrences of the given issue with the code, build a list of them to track what should be done, and another tool to prevent writing the code in the old style.

## Unexpected tool to solve our problems

Fortunately, Ruby ecosystem already has such tool. [Rubocop](https://github.com/rubocop/rubocop) is a linter that has a lot of plugins to enforce various best practices for your codebase. When particular rule is _offended_—Rubocop can highlight it in the code editor. Most of Ruby applications have it executed on the CI as well.

If you cannot fix the problem right now, but want to do it later—you can add the exception to the `.rubocop_todo.yml`:

```yml
Style/BracesAroundHashParameters:
  Exclude:
    - 'app/models/user.rb'
```

But how can it help us to refactor _our_ application to _our_ best practices? Easy, you just need to write your own cops! That might sound complex, but you can get used to it right after you write one or two of them.

### Writing custom cops for Rubocop

For instance, imagine that you want to restrict opening DB transactions anywhere except `app/business_actions/`. Here's a cop:

```ruby
module Isolation
  class IllegalTransactionBlock < Cop
    MSG = "Transaction shoud only be opened from business actions".freeze

    def_node_matcher :transaction?, "(send ... :transaction ...)"

    def on_send(node)
      add_offense(node, message: MSG) if transaction?(node)
    end
  end
end
```

Now you can configure Rubocop using `.rubocop.yml` to run it for a specific folder:

```yml
Isolation/IllegalTransactionBlock:
  Exclude:
    - app/business_actions/**/*.rb
```

Looks like we reached our goals:

1. now we can regenerate a TODO list for Rubocop to find all problems to fix;
2. with new TODO we can see where we are in terms of overall refactoring process;
3. no one can add new transaction in the illegal place without breaking the CI.

> Read more about writing cops in my [post](https://evilmartians.com/chronicles/custom-cops-for-rubocop-an-emergency-service-for-your-codebase), old but gold!

## Dealing with larger codebases

If you ever worked on a huge application, you can imagine what your `.rubocop_todo.yml` will look like, especially after installing new plugin, upgrading old ones and writing a number of custom cops. You're looking at this N–thousand–lines file and asking yourself where to start. You clearly understand that you cannot just fix everything right away, it will take _weeks_ of human–hours. Where to start?

I'd suggest to take a look at two things.

Firstly, some offences are a bit better than others. For instance, if you have nested transactions (why is it bad? [read this](https://pragtob.wordpress.com/2017/12/12/surprises-with-nested-transactions-rollbacks-and-activerecord/))—you might want to fix that before, say, replacing `=>` with `:` in hash literals. Also, some issues might block others, so they should go first. Looks like we need to add some _weights_ to our cops.

Secondly, files are updated with a different frequency. I'd prefer "hot" files to satisfy all team best practices, cause people will learn them faster just by looking at this code. Also, files that were not updated for years likely gonna have no issues or can eventually be sunset, so we can postpone the refactoring. Sounds like we need to consider a frequency of updates as well. Where do we have it? In the git history!

Now we need to combine these two things—meet [rubocop_director](https://github.com/DmitryTsepelev/rubocop_director)!

### Setting up rubocop-director 🎬

To get the ball rolling, install the gem and generate the default config:

```bash
$ bundle add rubocop_director
$ bundle exec rubocop-director --generate-config
```

You will get a file with weights of all cops that have offences, default cop weight and file frequency update weight:

```yml
update_weight: 1
default_cop_weight: 1
weights:
  Isolation/IllegalTransactionBlock: 1
  Layout/HashAlignment: 1
```

Now you can grab some pop corn and let `rubocop_director` to make a refactoring plan for you:

```bash
bundle exec rubocop-director
```

When it's done (it might take a while)—you'll see the report:

```bash
💡 Checking git history since 1995-01-01 to find hot files...
💡🎥 Running rubocop to get the list of offences to fix...
💡🎥🎬 Calculating a list of files to refactor...

Path: app/controllers/user_controller.rb
Updated 99 times since 1995-01-01
Offenses:
  🚓 Isolation/IllegalTransactionBlock - 2
Refactoring value: 1.5431217598108933 (54.79575%)

Path: app/models/user.rb
Updated 136 times since 1995-01-01
Offenses:
  🚓 Isolation/IllegalTransactionBlock - 1
  🚓 Layout/HashAlignment - 1
Refactoring value: 1.2730122208719792 (45.20425%)
```

Files are sorted by the _refactoring value_, which is calculated using a formula: `sum of value from each cop (<count of offences> * <cop weight> * (<count of file updates> / <total count of updates>) ** <update weight>)`. As you see, file update frequency is here too!

What can you tune here?

1. change cop weights to increase the value of ones you want to go first;
2. change the update frequency weight (you might want to increase or reduce it);
3. change the date, since when updates are counted (e.g., `--since 2023-01-01`).

When you're done—rebuild the report and you'll get a new prioritized list. Now you can pick up some files from the top and start working on them!

---

In this post we discussed a method of an observable refactoring process of Ruby applications. Here are key takeaways:

1. start with detecting issues in the code and finding solutions for them;
2. if you can describe an issue in terms of Rubocop—do that and get all the benefits (CI, linters etc.);
3. otherwise—describe it in docs and track the progress manually;
4. if you cannot fix issues all at once—build a list of files to fix, prioritize them and work on them from the top;
5. if you were able to do everything using Rubocop—you can grab [rubocop_director](https://github.com/DmitryTsepelev/rubocop_director) for this job.

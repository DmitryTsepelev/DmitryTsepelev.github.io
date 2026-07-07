---
title: "Surviving rolling deploys when Sidekiq meets a class it doesn't know yet"
date: 2026-07-07 09:00:00 +0300
permalink: 'sidekiq-requeue-missing-class'
description: 'A tiny Sidekiq middleware that requeues jobs whose class isn''t loaded yet, so rolling deploys stop throwing NameError—plus a look at how it works under the hood.'
tags: ruby rails sidekiq
---

You ship a new background job—let's call it `SendShinyNewThingJob`—run the deploy, and go get coffee. By the time you're back, the error tracker is on fire:

```
NameError: uninitialized constant SendShinyNewThingJob
```

But the class is _right there_ in the codebase. It's deployed. You can call `SendShinyNewThingJob.perform_later` in a console on production without a hitch. So what happened?

# Why a class you just deployed is "missing"

The culprit is the rolling deploy. When you roll out a new version, old and new processes run side by side for a while—that's the whole point, zero downtime. During that window you have two kinds of processes in play:

- something that _enqueues_ jobs: a freshly started pod, a scheduler (sidekiq-cron, `sidekiq-scheduler`, a clock process), or a web process already running the new code;
- something that _executes_ jobs: a Sidekiq worker that might still be running the old code, because it hasn't been restarted yet.

The new code enqueues `SendShinyNewThingJob`. An old worker picks it up, tries to materialize the class from the queue payload, and—since that constant simply doesn't exist in its process—blows up with a `NameError`. ActiveJob is no different: its Sidekiq wrapper calls `job_class.constantize` while deserializing the payload, so you get the very same error, just raised from inside the wrapper:

```
NameError: uninitialized constant SendShinyNewThingJob
```

It's a race condition, pure and simple. The job isn't wrong, the code isn't wrong—the enqueue just won the race against the worker restart.

# The workarounds, and why they get old

The textbook answer is "deploy discipline": always ship the code that _defines_ a job before you ship the code that _enqueues_ it.

The most rigorous version of this is a **two-phase deploy** (also called _expand and contract_, or the _parallel change_ pattern). Instead of shipping the whole change at once, you split it in two:

1. **Expand.** Ship a first deploy that only _adds_ `SendShinyNewThingJob`—the class exists, but nothing enqueues it yet. Wait for the rollout to fully finish, so every worker has restarted and now knows the class.
2. **Contract.** Ship a second deploy that actually _enqueues_ it. By now there's no worker left that can't load the class, so the race is gone.

The same idea shows up in lighter forms: guard the enqueue behind a feature flag you flip only after every worker has restarted, or orchestrate the rollout so workers cycle before schedulers do.

All of this works, but it adds overhead to every deploy. In practice the "define" and "enqueue" code usually lands in the same pull request, so you're manually splitting and sequencing it each time, and it's easy to forget. I wanted something that just handles the common case automatically, so I wrote [sidekiq-requeue_missing_class](https://github.com/DmitryTsepelev/sidekiq-requeue_missing_class).

# The fix

Add the gem and require it:

```ruby
# Gemfile
gem "sidekiq-requeue_missing_class"
```

```ruby
require "sidekiq/requeue_missing_class"
```

That's it. The middleware self-registers on the server side. When a worker pulls a job whose class it can't load yet, instead of raising, the job is quietly pushed back onto the queue with a short delay. A moment later—once the rollout finishes and workers are running the new code—the job gets picked up and runs normally. The `NameError` storm never happens.

If you want to tune it:

```ruby
Sidekiq::RequeueMissingClass.configure do |c|
  c.delay = 30               # seconds before the requeued job runs again (default: 30)
  c.max_requeues = 10        # how many times to bounce it before giving up (default: 10)
  c.logger = Sidekiq.logger  # or nil to stay quiet
end
```

The one number to think about is `delay × max_requeues`. Make it comfortably longer than a typical rollout—with the defaults that's five minutes of patience, which covers most deploys. If a job is _still_ unresolvable after `max_requeues`, the middleware stops babysitting it and lets it run (and fail) normally, so a genuinely missing class still surfaces as a real error instead of bouncing around your queue forever.

# How it works under the hood

The whole thing is a [Sidekiq server middleware](https://github.com/sidekiq/sidekiq/wiki/Middleware)—the layer that wraps every job's execution on the worker side. That's the natural spot to intervene: it sits _before_ your job runs, sees the job payload, and can decide whether to proceed or reschedule.

**Checking whether the class is loadable.** The middleware asks the same question Sidekiq itself would: can this constant be resolved right now? It uses `Object.const_get(name)`, which is exactly the path Sidekiq uses to turn a queue payload back into a class. That detail matters—because `const_get` triggers Rails autoloading, a class that _can_ be autoloaded counts as present, not missing. We only reschedule when the constant is genuinely absent from this process, not merely un-referenced-so-far.

**ActiveJob vs. native Sidekiq.** These carry the class name differently. For a native Sidekiq worker, the class in the payload _is_ the job. An ActiveJob job is wrapped in an adapter's job wrapper, with the real class name tucked away under `job_class`. The middleware handles both, so it doesn't matter which style your app uses—or, as is common in older codebases, both at once.

**Rescheduling instead of raising.** When the class isn't there, the job is put back with the configured delay rather than executed. To enforce `max_requeues`, it keeps track of how many times a given job has already bounced, so a job that will _never_ resolve doesn't loop endlessly—it eventually falls through to normal execution and fails loudly, which is what you want for an actually-broken class name.

The overhead for the happy path is negligible: for the overwhelming majority of jobs, whose class loads fine, it's one constant lookup and then straight through to your code.

# When you don't need it

To be fair about the boundaries:

- If your deploy pipeline already guarantees code-before-enqueue ordering and you trust it, you may not need this at all.
- It solves _missing classes_, and only that. It does **not** rescue you from incompatible changes to a job's arguments—renaming a keyword, changing arity, changing the shape of a serialized argument. An old worker can happily load a class and still choke on a payload shaped for the new version. That's exactly what the two-phase deploy above is for: expand the argument list first, wait for the rollout, then contract. The gem and expand/contract aren't rivals—reach for two-phase on genuinely breaking changes, and let the middleware be the safety net for the everyday missing-class race you'd otherwise have to babysit by hand.

---

Quick recap:

1. rolling deploys create a window where new code enqueues a job class that old workers haven't loaded yet, producing an `uninitialized constant` `NameError`;
2. the usual fix is manual deploy discipline, which works but is a tax you pay on every deploy;
3. [sidekiq-requeue_missing_class](https://github.com/DmitryTsepelev/sidekiq-requeue_missing_class) automates the common case—a server middleware requeues the job with a delay until a worker can load its class;
4. after `max_requeues` it gives up and lets the job fail normally, so genuinely broken class names still surface;
5. it handles both ActiveJob and native Sidekiq jobs, and uses the same `const_get` path Sidekiq does, so autoloadable classes aren't treated as missing;
6. it won't save you from incompatible _argument_ changes—that's a separate versioning concern.

If your deploys have ever coughed up a mysterious `uninitialized constant` for a class that's clearly deployed, give the gem a spin—install, require, done. And if you hit an edge case or have an idea, [issues and PRs](https://github.com/DmitryTsepelev/sidekiq-requeue_missing_class) are always welcome!

---

Wrangling Sidekiq, rolling deploys, and background-job reliability at scale? [I offer Rails architecture consulting.](/consulting/)

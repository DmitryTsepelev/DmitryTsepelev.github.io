---
layout: post
title: "On chosing the ideal architecture of the web application"
date: 2022-08-30 10:00:00 +0300
permalink: 'on-ideal-architecture'
description: 'An essay about my criteria for the ideal architecture'
tags: engineering ruby rails
---

When you need to _make a change_ (or fix a bug) in your web application and _you know the class and method_ you need to change then your architecture is _perfect_.

---

Many people—many minds. If you ask people about the best architecture they worked with—you'll get a lot of conflicting opinions. What do they argue about?

They prefer to use different abstractions for different purposes (which means there is no golden standard). From the other hand, some people do not think about the architecture at all, because they use some "framework" and they just follow the "best practices". This approach is more dangerous, because when there is no convention—everyone will assume that he follows these practices in a way he understands them.

I spent almost a decade working with [Ruby on Rails](https://rubyonrails.org), so I'm going to use it as a reference. In case of Rails, the "best practice from framework" is to follow the MVC–ish architecture:

- `models` contain classes that represent entities using an [active record](https://en.wikipedia.org/wiki/Active_record_pattern) pattern: they encapsulate both the logic of working with the database and business logic of the entity itself;
- `controllers` receive HTTP requests, perform actions by manipulating models and return the HTTP response to clients;
- `views` contain templates to render complex responses (e.g., HTML or JSON) and _might be_ used in controllers.

The approach above is called "fat models". If business logic is moved to controllers than the approach is called "fat controllers" (which does not fix anything).

|Responsibilities|Model|Controller|View|
|-|:-:|:-:|:-:|
|Performing DB queries|+|||
|Business Logic|+|||
|Serving HTTP request||+||
|Presenting the data|||+|

Is it everything? Nope, we also have to deal with authorization and authentication. Also, most of applications have background jobs that also contain some logic that:

|Responsibilities|Controller|Background Job|Model|View|
|-|:-:|:-:|:-:|
|Performing DB queries|||+||
|Business Logic||+|+||
|Serving HTTP request|+||||
|Authorization|+||||
|Authentication|+||||
|Presenting the data||||+|

After a while developers realized that this is not enough. The mature application starts struggling because classes know how to do to many things: they do not follow the [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single-responsibility_principle), which tells us that each class has to be responsible for a single thing.

However, some models are often used in different flows and grow up to thousands of lines. When you need to add new feature—you have to look through all the methods to understand if there is something similar already. Also, when there are a lot of methods—it's hard to track down dependencies between them, so we get the _high coupling_ between groups of methods.

Developers invented service objects.

![service object](/assets/service_object.png)

We got it covered! Business logic is extracted to service objects!

|Responsibilities|Controller|Background Job|Service Object|Model|View|
|-|:-:|:-:|:-:|
|Authentication|+|||||
|Serving HTTP request|+|||||
|Business Logic||calls service object|+|||
|Authorization|||+|||
|Performing DB queries||||+||
|Presenting the data|||||+|

Wait, is there an international standard of creating service objects? Unfortunatelly no, so after a while you get an endless folder with various classes doing different things. Each attempt to fix it ends up in either new gem (there are [a lot](https://www.ruby-toolbox.com/categories/Service_Objects)) or just a new layer for architecture archeology.

![standards](/assets/standards.png)

What kind of business logic people tend to put to service objects? Everything usually starts with CRUD operations and complex query builders. Also, many applications have third–party integrations, so these API clients become services too.

|Responsibilities|Controller|Background Job|Service Object|Model|View|
|-|:-:|:-:|:-:|
|Authentication|+|||||
|Serving HTTP request|+|||||
|CRUD||calls service object|+|||
|Query building|||+|||
|External API clients|||+|||
|Authorization|||+|||
|Performing DB queries||||+||
|Presenting the data|||||+|

What to do when something gets too many responsibility? ~~Call ghostbusters~~ Split! Let's move CRUDs to "actions", and move query builders and API clients to separate folders as well:

|Responsibilities|Controller|Background Job|Action|Query Object|API Client|Model|View|
|-|:-:|:-:|:-:|
|Authentication|+|||||||
|Serving HTTP request|+|||||||
|Authorization|||+|+||||
|CRUD||calls service object|+|||||
|Query building||||+||||
|External API clients|||||+|||
|Performing DB queries||||||+||
|Presenting the data|||||||+|

Okay, now we have authorization messing up with query building and logic, so let's introduce another laye... Let's stop here.

---

Which approach is correct? Well, all of them are fine! Depending on the application size, it might be enough to go with Rails–way, but make sure to define responsibilities for each layer and do not go over boards. See the HTTP code or current user in the model? The border was crossed.

How to understand that architecture does not fit your needs? For instance, when you are not sure what class is responsible for the thing you want to do. Another sign is when you have a _specific case_ that does not fit to your folder structure (e.g., when you need to respond with JSON for the first time and you put formatting logic just to the controller).

In opposite, when responsibilities are clear and consistent—you always know where to find the code you want to change. Keep the architecture easy to understand. Mind the SRP.


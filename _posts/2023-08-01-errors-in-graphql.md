---
title: "Errors in GraphQL"
date: 2023-08-01 12:00:00 +0300
permalink: 'errors-in-graphql'
description: 'GraphQL errors best practices'
tags: graphql
---

One of the main GraphQL benefits is a type system. It allows both server and client applications have some _guarantees_ of what they are expecting to receive. However, according to the standard, errors are not typed: it's just an array of valid JSON objects. This is why we should design errors more thoughtfully. In this post we will discuss some good practices and patterns I saw while working on a number of GraphQL applications.

## Errors as a part of the GraphQL response

According to the GraphQL specification, errors are the part of the response (along with the `data` and `extensions`), for instance:

```json
{
  "data": {
    "viewer": nil
  },
  "errors": ["access denied"],
  "extensions": {
    "complexity": 12
  }
}
```

The specification does not define a type for errors, like it does for the `data`. The array element inside the `errors` key should be a valid JSON structure (e.g., strings or objects).

## Expected versus unexpected errors

Instead of putting everything under the `errors` key, let's try to divide errors into groups:

1. errors that are completely unexpected and cannot be fixed by user in any way (e.g., database is down);
2. errors that affect a part of the user workflow (e.g., user is not authorized to access a certain part of the graph, and the error will look the same for _any_ field);
3. errors in the submitted data (i.e., validation errors).

We can spot a couple of interesting patterns here.

First and second type of errors can happen both in queries and mutations. They are not depending on the submitted data at all. Morevover, user cannot fix this error by submitting different arguments with the same query string.

Third group can only happen in mutations, because this kind of errors can represent either invalid state of the data in the database (e.g., processed order cannot be cancelled) or invalid mutation arguments (e.g., percent cannot be greater than 100).

## Taming top–level errors

GraphQL errors are not typed, so we have to add some types on our own. Let's make errors objects (not just strings) with code and details fields:

```json
{
  "data": {
    "processOrder": nil
  },
  "errors": [
    {
      "code": "permission_denied",
      "details": { "message": "you cannot perform this action" }
    },
    {
      "code": "server_error",
      "details": { "message": "DB is down" }
    }
  ]
}
```

Codes should go to the special list and existing codes should never change. It will help client applications to handle such errors in a common manner. Details might contain anything you want, but it might be helpful to have the message field everywhere.

### Permission errors

If application has a permission/role system and it is possible for user to try to do some action without success guarantee — we might want to have this as a specific error. Permission errors can happen both in mutations and queries, so it makes sense to return them inside the top–level `errors` field.

For instance, imagine that there is a mutation for order processing:

```gql
mutation ProcessOrder($id: ID!) {
  processOrder(id: $id) {
    status
    user {
      email
    }
  }
}
```

Where permission error can happen? It can be on the mutation level:

```json
{
  "data": {
    "processOrder": nil
  },
  "errors": [
    {
      "code": "permission_error",
      "details": {
        "field": ["processOrder"],
        "message": "you cannot process this order"
      }
    }
  ]
}
```

...or processing might work fine, but user cannot be accessed:

```json
{
  "data": {
    "processOrder": {
      "status": "processed",
      "user": nil
    }
  },
  "errors": [
    {
      "code": "permission_error",
      "details": {
        "field": ["processOrder", "user"],
        "message": "you cannot access this data"
      }
    }
  ]
}
```

One more thing: one more approach to permissions is to have a special place to ask server if the particular action is possible. It could be a special type in the type or root. If it says you can do it — do it, otherwise you'll get an error in the errors key.

### Application errors

When something goes really bad on the server side (e.g., database is not responding or query format is wrong) we should put it to the `errors`. For instance:

```json
{
  "data": {
    "processOrder": nil
  },
  "errors": [
    {
      "code": "client_error",
      "details": { "message": "request format is wrong" }
    }
  ]
}
```

Note that usually such errors should not be shown to the client as is. Instead, we should tell them that something wrong is happened and optionally ask to contact the support (optionally, because we should already know about this issue from our error tracker).

## Handing validation errors

Validation errors follow a similar pattern as mutation arguments, so it makes sense to keep them as a part of the response, rather than put them to the top–level `errors` array. In this case we keep the whole power of the GraphQL type system. In this case each mutation will have a union as the return type: `Success` will be returned when everything is fine, while `Failure` will represent validation errors.

For instance, imagine the mutation that registers a new user:

```gql
mutation SignUp($email: String!, $password: String!) {
  signUp(email: $email, password: $password) {
    __typename

    ...on SignUpSuccess {
      user {
        id
      }
    }

    ...on SignUpFailure {
      email
      password
    }
  }
}
```

The successful response will look like this:

```json
{
  "data": {
    "signUp": {
      "__typename": "SignUpSuccess",
      "id": 123
    }
  },
  "errors": []
}
```

In case of failure we can access errors for each fields, which can be just lists of strings to show to user:

```json
{
  "data": {
    "signUp": {
      "__typename": "SignUpFailure",
      "email": [],
      "password": ["should be longer than 8"]
    }
  },
  "errors": []
}
```

Sometimes mutation fails not because of the input, but because of the database state, which is related to this particular mutation. For instance, imagine the mutation `ProcessOrder` that checks for order status and prohibits processing orders which are already processed. In this case return type could be `ProcessOrderSuccess | OrderCannotBeProcessed | ProcessOrderFailure`:

```gql
mutation ProcessOrder($id: ID!) {
  processOrder(id: $id) {
    ...on ProcessOrderSuccess {
      order {
        ...OrderFields
      }
    }

    ...on OrderCannotBeProcessed {
      order {
        ...OrderFields
      }
      # no reason cause it's obvious
    }

    ...on ProcessOrderFailure {
      order {
        ...OrderFields
      }
      # we can also grab a reason of failure
      reason
    }
  }
}
```

The alternative to this approach could be a shared `Success` type which always points to the root of the graph, allowing clients to fetch whatever they want:

```gql
mutation ProcessOrder($id: ID!) {
  processOrder(id: $id) {
    ...on ProcessOrderSuccess {
      # we're in the root so we can fetch whatever we need
      orderById($id: ID!) {
        ...OrderFields
      }
      viewer {
        email
      }
    }
  }
}
```

This is more flexible but also more verbose.

## A word about monitoring

One of the common techniques in monitoring is [USE](https://thenewstack.io/?p=22690467):

- **U**tlization represents the ratio of resource used (e.g., how many background workers are in use);
- **S**aturation stands for the lack of the resource (e.g., how many background jobs are waiting while all workers are busy);
- **E**rrors represent the amount of errors in the monitored component.

As you might guess, we care about the last one in this post. In REST it was pretty simple to find errors: you can look at response HTTP codes and count 5xx. It's note so easy with GraphQL, where most of responses are `200 OK`, so we should not forget to configure our monitoring tool to count responses with `errors` filled. Moreover, we should differ real errors from, say, permission errors. This is the moment when error codes come really handy.

---

Let's sum up what we discussed in the post:

- generic errors that user should not see should go to `errors`;
- permissions errors are likely gonna go to `errors` too;
- when if we put something to `errors` — it’s better to have a structure (e.g., code and details)
- validations and mutation specific errors should go to the mutation payload.

---
layout: post
title: "Haskell Adventures: Functors"
date: 2022-03-31 12-00-00
permalink: 'haskell-adventures-functors'
description: 'Ruby engineer is exploring functors in Haskell'
tags: haskell
---

**[Part 1](/haskell-adventures) \| Part 2**

Software design becomes easier when it's possible to _replace parts_ of the program: for instance, user might want to load the same report in the CSV or XLS format. In this case we need to have a shared part that prepares the data, and the replaceable one for the formatting. In the object–oriented world we have classes with methods that can be _polymorphic_: they have the same API but work differently under the hood. Usually such classes might have a common base class/implement interface (e.g., C#, Java) or just have a method with the same signature (e.g., Ruby, Python). How can we do it in the Haskell where we do not have classes in the traditional sense?

## Types are instances of classes. Wait, what?

Believe me or not, but in Haskell this problem is solved using _classes_ 👹 However, Haskell classes have nothing common with OOP ones: class, or _type class_, defines a list of functions that should be implemented in the type that belongs to the class. As soon as we know that we work with a type that implements a class, we can start using all the functions defined in its class type.

> You might notice that type classes are very similar to OOP interfaces

Time to get our hands dirty! Imagine a situation that you have a value that might be blank, let's try to present it using types:

```haskell
data MMaybe a = MJust a | MNothing deriving (Show)

MJust 1 -- MJust 1
```

Welcome the _type constructor_, that can embed the value of any other type (i.e., `a`) and gives it a _context_: it's easy to understand if value is present, or not. As a result, our value goes to the some kind of _box_ (this analogy is not fully correct but we can go with that for now).

Let's try to compare two values:

```haskell
MJust 1 == MJust 2

-- <interactive>:5:1: error:
--     • No instance for (Eq (MMaybe Integer)) arising from a use of ‘==’
--     • In the expression: MJust 1 == MJust 2
--       In an equation for ‘it’: it = MJust 1 == MJust 2
```

Boom! `No instance for (Eq (MMaybe Integer))` means that our type should be an instance of a class `Eq`. No more words, I do want to compare my boxes!

```haskell
instance (Eq m) => Eq (MMaybe m) where
  MJust x == MJust y = x == y
  MNothing == MNothing = True
  _ == _ = False

MJust 1 == MJust 2 -- True
```

There are three possible scenarios: `MNothing` is equal to `MNothing`, `MNothing` and `MJust` can never be equal, `MJust` is equal to `MJust` only when their values are equal. In order to compare values we do `x == y`, but how Haskell knows these two values _can be_ compared? This is why we have `(Eq m) =>`: it tells compiler that `(MMaybe m)` can only implement `Eq` when `m` implements `Eq`. Let's try `MMaybe` with something incomparable:

```haskell
data Incomparable = Something | SomethingElse

MJust Something == MJust SomethingElse

-- <interactive>:8:1: error:
--     • No instance for (Eq (MMaybe Incomparable))
--         arising from a use of ‘==’
--     • In the expression: MJust Something == MJust SomethingElse
--       In an equation for ‘it’:
--           it = MJust Something == MJust SomethingElse
```

Failed as expected! By the way, did you notice little `deriving Show` at the beginning? `Show` is also class type that teaches Haskell how to print values, while `deriving` asks Haskell to use the _default_ implementation.

> – Mom, can we have the implementation of `Show`?<br/>
> – We have the implementation at home!<br/>
> At home: `deriving`

## Using Maybe and understanding Functor

Why did I call my type `MMaybe`? Because the problem we solve is so common that `Maybe` is a part of a standard library! Let's try it out: we're going to write _business logic_, like we do in our daily work 🙂 There is a Jira card assigned to us: we need to read a user's email from the database and format it before printing.

```haskell
type UserID = Int
type Email = String

getEmail :: UserID -> Maybe Email
getEmail 42 = Just "john.doe@example.com"
getEmail _ = Nothing

formatEmail :: Maybe Email -> Maybe Email
formatEmail maybeEmail
  | Just email <- maybeEmail = Just $ "Email: " ++ email
  | otherwise = Nothing

formatEmail $ getEmail 42 -- Just "Email: john.doe@example.com"
```

First of all, we define two new types—`UserId` and `Email` to make our signatures more readable (otherwise our function signature was `getEmail :: Int -> Maybe String`). Technically they are just aliases to built–in types.

The `getEmail` function imitates database call. Since our startup is still small we have only one user. Moreover, previous 41 users deleted their data. 🤷‍♂️ Notice that we switched to the `Maybe` type from the standard library (it's called _Prelude_). Our homemade implementation was kicked out of the house.

The `formatEmail` function accepts the `Maybe Email` type and performs the formatting: if it's not `Nothing` it "unboxes" the value (`Just email <- maybeEmail`), adds the `"Email: "` prefix and packs it back to `Maybe`.

You might notice a new thing in the syntax: there is `$` in the `Just $ "Email: " ++ email`, what is it for? It helps us to change or order of execution without using braces. If we remove `$` from this expression, it will be executed as `(Just "Email: ") ++ email`, which will obviously fail. In opposite, `$` will make sure that the right part will be executed earlier (i.e., as it was `Just ("Email: " ++ email)`).

The pattern when we take value out of the "box", change it and put back sounds like something we might want to use often. Please welcome our new friend—type class `Functor!`:

```haskell
data MMaybe a = MJust a | MNothing deriving (Show)

instance Functor MMaybe where
  fmap f (MJust x) = MJust (f x)
  fmap f MNothing = MNothing

type UserID = Int
type Email = String

getEmail :: UserID -> MMaybe Email
getEmail 42 = MJust "john.doe@example.com"
getEmail _ = MNothing

formatEmail :: MMaybe Email -> MMaybe Email
formatEmail = fmap (\email -> "Email: " ++ email)

formatEmail $ getEmail 42 -- MJust "Email: john.doe@example.com"
```

In order to prepare this example we had to bring back our own implementation of `Maybe` we kicked off earlier. The operation we look for is called `fmap`: it accepts a function and a "box"; if possible, it takes value from the box, applies the function and puts the value back to the box. This is how we can implement it for the `MMaybe`:

```haskell
instance Functor MMaybe where
   fmap f (MJust x) = MJust (f x)
   fmap f MNothing = MNothing
```

Now we can change `formatEmail` to use `fmap` and get rid of the pattern matching:

```haskell
formatEmail :: MMaybe Email -> MMaybe Email
formatEmail = fmap (\email -> "Email: " ++ email)
```

`fmap` is used very often, so there is even an infix form for it—`<$>` (please note that this snippet was switched back to the built–in `Maybe` which is already an instance of `Functor`):

```haskell
type UserID = Int
type Email = String

getEmail :: UserID -> Maybe Email
getEmail 42 = Just "john.doe@example.com"
getEmail _ = Nothing

formatEmail :: Maybe Email -> Maybe Email
formatEmail maybeEmail = ("Email: " ++) <$> maybeEmail

formatEmail $ getEmail 42 -- Just "Email: john.doe@example.com"
formatEmail $ getEmail 4 -- Nothing
```

By the way, a list type (e.g., `[1, 2, 3]`) is an instance of Functor too. How do you think `fmap` is implemented for lists? Right, it's just a `map` you've seen thousands of times before 🙂

## Functors for functions

In the previous example we used `fmap` with the function that accepts only one argument: `++`, which is used to concatenate lists, accepts two arguments, but we partially applied one argument using _currying_ (`("Email: " ++)`) so it became a function of one argument.

> If you forgot what currying is—check out the [first part](/haskell-adventures) of the article

What if we pass a function that accepts two arguments (e.g., `++`) to the `fmap`? As we know, `fmap` applies the function to the value in the box. We also know, that when we call a multi–parameter function with a single argument, it returns a new function without the first argument. Combine these two facts and you'll get the right answer: `fmap` will take the value from the box, partially apply it to the function, and put that curried function back to the box! Congratulations, you just discovered _applicative functors_.

There is a special type class called `Applicative` that adds a support of applicative functors to types. In order to make things work, we need to implement two functions: `pure` and `<*>`.

```haskell
data MMaybe a = MJust a | MNothing deriving (Show)

instance Functor MMaybe where
   fmap f (MJust x) = MJust (f x)
   fmap f MNothing = MNothing

instance Applicative MMaybe where
  pure = MJust
  MNothing <*> _ = MNothing
  (MJust f) <*> something = fmap f something
```

The `pure` function accepts a value and puts it to the box. For instance, in case of `MMaybe` it wraps the value with `MJust` and calls it a day; if it was a list—it would create a list with a single element.

The `<*>` is a little bit tricky: it has an applicative functor on the left and some value on the right. It tries to apply the function from the applicative functor to the value when it's possible, otherwise, when there is the `MNothing` on the left—returns `MNothing`.

You might wonder why one might want to use it, so here is an example from our do much business application:

```haskell
type UserID = Int
type Email = String

getEmail :: UserID -> MMaybe Email
getEmail 42 = MJust "john.doe@example.com"
getEmail _ = MNothing

formatEmail :: MMaybe Email -> MMaybe Email
formatEmail maybeEmail = (++) <$> maybeEmail <*> MJust "Email: "

formatEmail $ getEmail 42 -- MJust "Email: john.doe@example.com"
formatEmail $ getEmail 4 -- MNothing
```

Take a closer look at the updated `formatEmail` function:

```haskell
formatEmail :: MMaybe Email -> MMaybe Email
formatEmail maybeEmail = (++) <$> maybeEmail <*> MJust "Email: "
```

First of all we build an applicative functor using `(++) <$> maybeEmail`, there are two scenarios possible:

1. if `maybeEmail` contains a value then `++` will be applied to that value (e.g., `(++) <$> MJust "value"` will become `MJust ("value ++)`);
2. if `maybeEmail` is `MNothing` then it will just stay `MNothing`.

After that `<*>` is used to apply functor to the value. Again, there are two scenarios possible:

1. if there is an `MNothing` on the left then it will just stay `MNothing`;
2. if there is an applicative functor on the left then the function inside it will be applied to the value inside `MJust` (e.g. `MJust ("value" ++) <*> MJust "!"` becomes `MJust ("value" ++ "!")` and `MJust ("value!")`).

Why so complex? The **main benefit** here is that we only care about the "good" scenario when email exists, because when `MNothing` appears in the middle of the chain—it's just _traversed_ to the top of the calculation.

## ISBN validation problem

If you read my previous article about Haskell you might notice that I'm bad at examples, so I steal them from [Codewars](https://www.codewars.com). Things did not change, and here comes another one!

Imagine that you need to validate an [ISBN](https://en.wikipedia.org/wiki/International_Standard_Book_Number) number and here are the rules:

1. number has exactly 10 symbols;
2. every position except the last one can contain only digits;
3. last position can be either digit or "X" (which means 10);
4. given that positions start with 1, a sum of positions multiplied by the corresponding digit should be dividable by 11 without a reminder.

Let me illustrate the last part: `048665088X` is valid because `(0*1+4*2+8*3+6*4+6*5+5*6+0*7+8*8+8*9+10*10) / 11` equals to `32`.

Right to the solution:

```haskell
import Data.List
import Data.Char ( digitToInt, isDigit )
import Data.Maybe

validISBN10 :: String -> Bool
validISBN10 num
  | length num /= 10 = False
  | Nothing <- sumDigitsMod11 = False
  | Just value <- sumDigitsMod11 = value == 0
  where
    sumDigitsMod11 = (`mod` 11) . sum <$> sequence components
    components = [(*) <$> pure pos <*> getDigit char pos | (pos, char) <- zip [1..] num]

getDigit :: Char -> Int -> Maybe Int
getDigit 'X' 10 = pure 10
getDigit 'X' _ = Nothing
getDigit d _
  | isDigit d = pure $ digitToInt d
  | otherwise = Nothing

validISBN10 "048665088X" -- True
validISBN10 "048665188X" -- False
```

As before, let's read from the bottom to the top. A function `getDigit` accepts a character and its position and returns `Maybe Int`. Why `Maybe`? Well, it would be helpful to stop calculations when we see something invalid. First of all, we try to parse `X` on the 10 position:

```haskell
getDigit 'X' 10 = pure 10
```

Then, we make sure that `X` does not appear somewhere else:

```haskell
getDigit 'X' _ = Nothing
```

Finally, we try to parse a digit and return it, otherwise—return `Nothing`:

```haskell
getDigit d _
  | isDigit d = pure $ digitToInt d
  | otherwise = Nothing
```

Before we get to that huge function that solves our problem, let's come up with the plan: we need to take all the chars along with their positions, parse them, sum and make sure that the result can be divided by 11. Since we operate functors we can stop thinking about bad scenarios and just return Nothing.

Let's start with generating pairs of positions and parsed digits:

```haskell
components = [(*) <$> pure pos <*> getDigit char pos | (pos, char) <- zip [1..] num]
```

You might guess that [1..] is the endless list. `zip` takes two lists and generates a list of pairs, the length of the list will match the length of the shortest one. Given that `num` equals to `"048665088X"`, `zip [1..] num` will produce `[(1,'0'),(2,'4'),(3,'8'),(4,'6'),(5,'6'),(6,'5'),(7,'0'),(8,'8'),(9,'8'),(10,'X')]`.

We use a list generation syntax to get a list of positions multiplied by digits:

```haskell
(*) <$> pure pos <*> getDigit char pos
```

`(*) <$> pure pos` takes a position `pos` (a first element of the pair), uses `pure` to put it to the `Maybe` (we could write `Just pos`, but we want be _fancy_) and makes it an applicative functor. For instance, if our `pos` equals 2, the result of `(*) <$> pure pos` will be `Just (* 2)`. After that, we apply this functor to the result of the `getDigit` call, which can be either `Just` or `Nothing`. As a result, the whole expression will be either `Just Int` or `Nothing`, and `components` will contain the array with multiplication results.

Huh, next line:

```haskell
sumDigitsMod11 = (`mod` 11) . sum <$> sequence components
```

Let's start with `sequence`: this function takes a list of "boxes", takes all values, puts them into the list and wraps the list with the box again. As usual, a single `Nothing` element makes the whole thing `Nothing`:

```haskell
sequence [Just 1, Just 2, Just 3] -- Just [1,2,3]
sequence [Just 1, Nothing, Just 3] -- Nothing
```

``(`mod` 11) . sum`` is a composition of two functions: the result function will accept some list, sum values in it and call `mod` on it.

> By the way, `.` is a function too! It has an interesting type `(b -> c) -> (a -> b) -> a -> c`, which means that when it gets two functions it combines them into a new one. Also, since `.` is a function, we can use a prefix form: ``(.) (`mod` 11) sum [1,2,3]``. Not sure why you might need it, just wanted to let you know 🙂

Finally, we use `<$>` to perform the operation on our `Maybe` value on the right. When it's `Just`—we sum numbers and get the reminder. Now we only need to check our conditions:

```haskell
validISBN10 :: String -> Bool
validISBN10 num
  | length num /= 10 = False
  | Nothing <- sumDigitsMod11 = False
  | Just value <- sumDigitsMod11 = value == 0
  where
    sumDigitsMod11 = (`mod` 11) . sum <$> sequence components
    components = [(*) <$> pure pos <*> getDigit char pos | (pos, char) <- zip [1..] num]
```

We need to check three things:

- `length num /= 10 = False` checks the expected length of the input;
- `Nothing <- sumDigitsMod11 = False` handles the case when something wrong happened during parsing (i.e., `Nothing` appeared somewhere);
- `Just value <- sumDigitsMod11 = value == 0` checks that reminder is zero.

🎉 Problem is solved!

---

That's all for today! Let's recap what we learned:

1. Haskell has type classes to define list of functions and, optionally, default implementations for them;
2. types can be instances of classes, which means that have all these functions implemented;
3. types can be polymorphic using type classses;
4. types can be "boxes" for the values they hold, box gives value a "context";
5. `Functor` is a class that can change value in the box;
6. `Applicative Functors` represent functions inside boxes and their usage along with regular values.

Reminding that I have no production experience in Haskell, so things I write might be inaccurate or even completely wrong. Bear with me 🐻

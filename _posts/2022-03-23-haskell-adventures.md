---
layout: post
title:  "Haskell Adventures: digging into the declarative approach"
date:   2022-03-23 12-00-00
permalink: 'haskell-adventures'
description: 'Rubyist is travelling to the unexplored Haskelland'
---

My Haskell journey started with a [Stepic course](https://stepik.org/course/75) a couple of years ago: I learned a lot of cool things back in the days but there were no room for a practice. This year I started to solve [Advent of Code](https://github.com/DmitryTsepelev/advent_of_code_2021) using Go but realized that this is a perfect moment to grab Haskell from the shelf and try to use it. Turned out that Haskell is very beautiful!

This article is not going to teach you how to write Haskell. Instead, I'm going show my excitement by sharing a couple of cool code snippents; maybe it will convince you to give Haskell a shot.

> **Heads up!** I'm never a Haskell programmer and I don't have any production experience with it; my solutions are not ideal and my explanations likely gonna be wrong. Feel free to yell at me in any way you can 🙂

## List generators

Haskell is a functional language that convinces us to use a declarative approach to programming. What does it mean? Instead of writing an _algorithm_ that solves the problem, we specify what we _want to get_.

Please welcome our first guest—a list generator:

```haskell
[(x, y) | x <- [1, 2], y <- ['a', 'b']]
-- [(1,'a'),(1,'b'),(2,'a'),(2,'b')]
```
On the right from the pipe we specify lists we want to use for generation; we can have any number of lists we want and Haskell will create all possible combinations for us. On the left we use elements from the generated combination to build the element we want to see in the result list. In our case we need to have pairs built from elements of `x` and `y` lists. Under the hood it will be just nested loops, but we do not write them by hand!

Cool, but how can we exclude some elements? We can add conditions to the right side, for instance this is how we can exclude `(2, 'a')` element from the result list:

```haskell
[(x, y) | x <- [1, 2], y <- ['a', 'b'], (x, y) /= (2,  'a')]
-- [(1,'a'),(1,'b'),(2,'b')]
```

I bet you wonder why ~~the hell~~ we discuss list generators and where we can use them. Fear not, let's build something real!

## Gamedev 101

Imagine an endless chess board (2D, no surprizes) with a king, which can move to any cell adjacent to his current position. Our goal is to write a function that accepts a current position and returns a list of all possible positions, and here it is:

```haskell
moves (x, y) =
  map (\(dx, dy) -> (x + dx, y + dy)) deltas
  where
    deltas = [(dx, dy) | dx <- [-1..1], dy <- [-1..1], (dx, dy) /= (0, 0)]

moves (4, 5) -- [(3,4),(3,5),(3,6),(4,4),(4,6),(5,4),(5,5),(5,6)]
```

Let's figure out what's going on. First of all, we define a function that accepts a _pair_ of integers (to keep things simple, let's use numbers as coordinates). Note that we use _pattern matching_ to bind the first element of the pair to `x` and a second one to `y`:

```haskell
moves (x, y) =
```

Then, we use `map` to go through `deltas`:

```haskell
map (\(dx, dy) -> (x + dx, y + dy)) deltas
```

`map` is a function that accepts another function `f`, a list `l`, applies `f` to each element of `l` and returns a list of results; `deltas` represents possible coordinate changes from the current position.

Now we are going to _generate_ `deltas`:

```haskell
where deltas = [(dx, dy) | dx <- [-1..1], dy <- [-1..1], (dx, dy) /= (0, 0)]
```

As before, we specify two lists (for `x` and for `y`) and exclude `(0, 0)` because we do want the king to move!

As we remember, `moves` accepts a pair of integers, but how Haskell knows that? It _guesses_, but we can help it and define types explicitly:

```haskell
type Position = (Int, Int)

move :: Position -> Position -> Position
move (x, y) (dx, dy) = (x + dx, y + dy)

moves :: Position -> [Position]
moves current =
  map (\delta -> move current delta) deltas
  where
    deltas = [(dx, dy) | dx <- [-1..1], dy <- [-1..1], (dx, dy) /= (0, 0)]
```

> Well, it's not a "guess" but the "type inference", thanks [@riftdc
](https://www.reddit.com/r/haskell/comments/tkpt4c/comment/i1wnw8q/?utm_source=share&utm_medium=web2x&context=3) for pointing it out ❤️

First of all, we define a new named type `Position` which represents a _pair of integers_:

```haskell
type Position = (Int, Int)
```

The anonymous function we passed to `map` was a bit cumbersome so let's extract it to the separate function called `move` and use pattern matching to make it clearer. This function accepts a current position, a delta and returns the result position:

```haskell
move :: Position -> Position -> Position
move (x, y) (dx, dy) = (x + dx, y + dy)
```

Finally, we add a signature to the `moves` function and change the implementation to use `move`:

```haskell
moves :: Position -> [Position]
moves current =
  map (\delta -> move current delta) deltas
  where
    deltas = [(dx, dy) | dx <- [-1..1], dy <- [-1..1], (dx, dy) /= (0, 0)]
```

If you heard of _currying_ before, you might feel that previous example is a bit dirty. Fear not, if you haven't, here is a very basic explanation from [Wiki](https://en.wikipedia.org/wiki/Currying):

> Currying is the technique of converting a function that takes multiple arguments into a sequence of functions that each takes a single argument

In Haskell _each_ function accepts a single argument and returns another function: if function accepts `n` arguments and returns a value `v` then it can accept `k` first arguments and return a new function that accepts `n - k` arguments. Let's take a look at a `+` function that accepts two numbers and returns a new number (i.e., `Int -> Int -> Int`). Nothing can stop you from calling it _partially_, with a one argument and get `Int -> Int` function:

```haskell
add = (+)
add3 = add 3
add3 2 -- 5
```

In the example above we called `+` function with one argument 3 and got a function that adds 3 to any other you number you pass.

Let's use this trick to cleanup our previous example. We change our `moves` function to partially call new function `move` with the current position, while second argument will be passed by `map`. `move` just sums two positions and uses pattern matching to keep things short:

```haskell
type Position = (Int, Int)

move :: Position -> Position -> Position
move (x, y) (dx, dy) = (x + dx, y + dy)

moves :: Position -> [Position]
moves current =
  map (move current) deltas
  where
    deltas = [(dx, dy) | dx <- [-1..1], dy <- [-1..1], (dx, dy) /= (0, 0)]
```

## Knight Gambit

Our next challenge is replace our endless chess board with a real one (8x8), and use knight instead of a king. Real chess players use special notation to represent field cells: letters from `a` to `h` represent a horizontal coordinate, while numbers from `1` to `8` stand for a vertical one. We're going to change our implementation to work with this coordinate system. The task stays the same, we need to write a function that returns all possible moves from a given position.

> Knight movement trajectory has an L shape: two steps horizontally and one vertically or the opposite.

Here is a whole thing:

```haskell
import Data.Char ( ord, chr )

type Position = (Char, Char)

movesFrom :: Position -> [Position]
movesFrom (h, v) = filter onBoard allMoves
  where
    allMoves = [(move h dh, move v dv) | dh <- deltas, dv <- deltas, abs dh /= abs dv]
    deltas = [-1, -2, 1, 2]
    move pos delta = chr (ord pos + delta)

onBoard :: Position -> Bool
onBoard (h, v) = h `elem` ['a'..'h'] && v `elem` ['1'..'8']

movesFrom ('b', '3') -- [('a','1'),('a','5'),('c','1'),('c','5'),('d','2'),('d','4')]
```

First of all, we need to import two library functions: `ord` returns an integer code for a given character, `chr` performs an opposite operation—returns a character for an integer code.

Then, let's take a look at a simple function `onBoard`, which checks if given position is valid for 8x8 board (`elem` makes sure that given element exists in the array):

```haskell
onBoard :: Position -> Bool
onBoard (h, v) = h `elem` ['a'..'h'] && v `elem` ['1'..'8']
```

Function `movesFrom` returns all possible positions from a given one, the only thing we need to do is to take all possible positions and removes ones outside the board:

```haskell
filter onBoard allMoves
```

What our options are? We generate all possible shifts using `deltas` (`[(dh, dv) | dh <- deltas, dv <- deltas, abs dh /= abs dv]` gives us all valid L–shapes `[(-1,-2),(-1,2),(-2,-1),(-2,1),(1,-2),(1,2),(2,-1),(2,1)]`) and use these deltas to perform movements. Function `move` uses `ord` to make position integer, adds delta and converts value back to the character:

```haskell
allMoves = [(move h dh, move v dv) | dh <- deltas, dv <- deltas, abs dh /= abs dv]
deltas = [-1, -2, 1, 2]
move pos delta = chr (ord pos + delta)
```

Now we can find possible movements for our knights...

## Shortest Knight Path

...and we want to find out a minimal count of movements knight needs to make to reach one given position from another. Here is how we do that:

```haskell
shortestPath :: Position -> Position -> Int
shortestPath from to =
  length
    $ takeWhile (to `notElem`)
    $ iterate generateMoves [from]
  where generateMoves = foldl (\acc pos -> acc ++ movesFrom pos) []

shortestPath ('b', '3') ('g', '5') -- 3
```

Let's read from the bottom to the top. `generateMoves` is a function that takes a list of positions and generate a new list of all possible positions that can be reached:

```haskell
generateMoves = foldl (\acc pos -> acc ++ movesFrom pos) []
```

We use currying again to omit the `generateMoves` argument, Haskell will pass it as the last argument to the `foldl` call. Here's the uncurried version:

```haskell
generateMoves positions = foldl (\acc pos -> acc ++ movesFrom pos) [] positions
```

`foldl` is the generic function to perform maps, reduces and all similar things. It accepts a function `f`, an initial accumulator and a list to iterate on. The function `f` is be called for each element of the list along with the current value of the accumulator and returned value becomes the new accumulator.

For instance, this is how we can replace `map (+1) [1, 2, 3]` with `foldl`:

```haskell
foldl (\acc i -> acc ++ [i + 1]) [] [1, 2, 3] -- [2, 3, 4]
```

And here is the "reduce" example—we want to get a sum of all numbers:

```haskell
foldl (\acc i -> acc + i) 0 [1, 2, 3] -- 6
```

Then, let's figure out what `iterate` does. This function accepts function `f` and an initial value `i` to iterate on; `f` will be called on `i` giving us some value `i'`, `f` will be called on `i'` and so on. All these values (`i`, `i'`, `i''`, ...) will be collected to the endless list.

Wait! Doesn't it mean our program will stop here? It won't, because Haskell is lazy and it won't calculate things until we try to access them, so it won't make any unneeded work. Here's an example that generates a list of squares starting from 2:

```haskell
take 5 $ iterate (^2) 2 -- [2, 4, 16, 256, 65536]
```

Now we're ready to understand `iterate generateMoves [from]` line: it generates _all possible paths_ from the given point; the result of this call is a list of lists, where each nested list contains points that are reachable on the nth step.

Now we need to limit this list somehow! `takeWhile` will keep iterating until the condition we specify is met, our condition is that any list contains the point we look for (```(to `notElem`)``` uses currying again, the uncurried version is ```(\positions -> to `notElem` positions)```). As a result we'll have a list of lists, and we only need to calculate, how many elements this list has, so we use `length`.

Voila! 🎉

---

That's all for today, and I hope that some readers already getting some books to dig into Haskell. Next time we'll talk about functors. Stay tuned!

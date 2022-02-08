+++
title = "Learning Haskell during AOC"
date = 2022-02-08
[taxonomies]
tags = ["Haskell"]
+++
Last december I decided to take part in the Advent of Code. Of course, just programming my way through the AOC wouldn't be as much fun as doing it in a *shiny new language*. This time I decided to give Haskell a try. I had been doing functional programming in Rust and Javascript for some time and I thought it would be great moment to finally try the hard stuff. I was eager to use a pure functional language, so the 1st of December I downloaded the Glasgow Haskell Compiler (GHC for friends) and started my completely functional adventure.

## Where to start
In [Haskell's official page](https://www.haskell.org/) there wasn't any official guide to learning Haskell, but there were some interesting links. I ended up reading [Learn you a Haskell for Great Good](http://www.learnyouahaskell.com/), which did teach me a lot and helped me solved most of the problems I had during the AOC.

## Automatic currying
The first thing that really got my attention was the automatic currying. Functions in Haskell can only accept one argument. Knowing that, it would be fair to ask "how can you write code with functions only taking one argument?". The answer relies in Higher Order functions. Here we can see a simple Haskell function with its corresponding signature.
```haskell
add :: (Num a) => a -> a -> a
add x y = x + y
```
If we take a closer look at the signature, we can see it takes something of a type `a` which must implement the Num typeclass (more on that later), but then it returns another functions `a -> a`, which really calculates the final result. This might seem odd at first, but it allows to do things like this.
```haskell
map (add 1) [1, 2, 3] -- Adds 1 to all the elements of the array
```
In fact, knowing that all operators can be used as functions in Haskell we can do the following:
```haskell
map (+ 1) [1, 2, 3] -- Adds 1 to all the elements of the array
```
Which also works! 
This allows to express some operations in a very concise way.

## Rust similarities
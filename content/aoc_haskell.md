+++
title = "Learning Haskell during AOC"
date = 2022-02-17
[taxonomies]
tags = ["Haskell"]
+++
Last december I decided to take part in the Advent of Code. Of course, just programming my way through the AOC wouldn't be as much fun as doing it in a *shiny new language*. This time I decided to give Haskell a try. I had been doing functional programming in Rust and Javascript for some time and I thought it would be a great moment to finally try the hard stuff. I was eager to use a pure functional language, so the 1st of December I downloaded the Glasgow Haskell Compiler (GHC for friends) and started my completely functional adventure.

## Where to start
In [Haskell's official page](https://www.haskell.org/) there wasn't an official guide to learn Haskell, but there were some interesting links. I ended up reading [Learn you a Haskell for Great Good](http://www.learnyouahaskell.com/), which did teach me a lot and helped me solve most of the problems I had during the AOC. I also read a lot of documentation at [Hackage](https://hackage.haskell.org/) and used [Cabal](https://www.haskell.org/cabal/) to help me run my programs and manage my dependencies.

## Automatic currying
The first thing that really got my attention was the automatic currying. Functions in Haskell can only accept one argument. Knowing that, it would be fair to ask "how can you write code with functions only taking one argument?". The answer relies in Higher Order functions. Here we can see a simple Haskell function with its corresponding signature.
```hs
add :: (Num a) => a -> a -> a
add x y = x + y
```
If we take a closer look at the signature, we can see it takes something of a type `a` which must implement the Num typeclass (more on that later), but then it returns another function `a -> a`, which is the one that really calculates the final result. This might seem odd at first, but it allows us to do things like this.
```hs
map (add 1) [1, 2, 3] -- Adds 1 to each element of the list 
```
In fact, if we know that all operators can be used as functions in Haskell, we can do the following:
```hs
map (+ 1) [1, 2, 3] -- Adds 1 to each element of the list 
```
Which also works! 
This allows to express some operations in a very concise way.

## Rust similarities
Coming from a Rust background, another thing I noticed was that lot of the syntax and names were similar to the Rust equivalents. For example, `derive` means more or less the same in both languages and both use `let` as a keyword to declare new variables. But the similarities go much deeper than just syntax and names. Rust's traits mimic the behavior of Haskell's typeclasses and Rust's enums also remember Haskell's sum types, which also explains why Rust's `Option` is so reminiscent of Haskell's `Maybe`. Furthermore, both languages feature pattern matching and guards. It seems clear to me that Haskell and other functional programming languages really influenced Rust. Of course there are other things that are way different in Haskell. 

## Hard to read symbols
Haskell can really differ from other languages, specially when you encounter symbols like `$`, `<$>`, `!`, `!!` or `<*>`, which are all commonly used in Haskell. To a newcomer they can be rather disorienting, even if they are easier to write when you already know the meaning. This got me thinking. In the end every language does this, it's just that Haskell forces that limit, resulting in more concise code, but harder to read for people who haven't been exposed to Haskell yet. It is difficult to say at which point the heavy usage of symbols in a language results in hard to read code, even for experienced people in that language. It may even vary with each person. I have read some code in [APL](https://tryapl.org/) and I don't think too many people will get angry at me if I say it is too much when you feel the need for [an specific keyboard](http://www.dyalog.com/apl-font-keyboard.htm).

## Function composition
One powerful thing about Haskell is how easy it is to compose new functions using the ones you already have. The `.` operator allows to write this type of code really easily. One example:
```hs
-- New function that adds 1 to each element of a list and then filters the numbers bigger than 2
foo = filter (> 2) . map (+ 1) 
```
Of course, the `.` operator alone wouldn't achieve this if it wasn't for functions like `filter`, `map` or `fmap`. 

## Parser combinators 
Throughout the AOC I found some problems when reading the input for the day. Some days the input would be rather complicated and I would struggle to parse it correctly into the data structures I had chosen. That all went away when I stumbled on [this page explaining parser combinators](https://two-wrongs.com/parser-combinators-parsing-for-haskell-beginners.html). They are really simple to use. For example, imagine we need to parse IPs, we could do the following:
```hs
parseIPv4 :: ReadP IP
parseIPv4 = do
  a <- readOctet
  char '.'
  b <- readOctet
  char '.'
  c <- readOctet
  char '.'
  d <- readOctet
  return $ IPv4 [a, b, c, d]
  where
    readOctet = fmap read $ many1 $ satisfy isDigit
```
It works mostly in a declarative way. We specify our requirements and then the combinators do their best to find matches with what we specify. In this case we are specifying four octets separated by dots. Each octet is composed by at least one digit and the whole number is parsed to an integer, so the final IPv4 representation is a list with four numbers. Keep in mind that `return` here doesn't mean the same that in other languages, but rather means the IPV4 we are returning will be wrapped in the `ReadP` monad (More on monads later).

Of course, you could say that IPv4 can be expressed in different ways and that parsing them is more complex than what appears here... **and you would be right!** This is only an example, but it allows me to illustrate how clean and simple parser combinators are.

If we also wanted to parse IPv6, it would be as simple as defining the correspondent combinator for the IPv6 and then combining both:

```hs
data IP = IPv4 [Int] | IPv6 [String] deriving (Show)

parseIP :: ReadP IP
parseIP = parseIPv4 +++ parseIPv6
```
The `+++` operator parses the input with **both** combinators. We can access both results (if they are succesful) when we use the corresponding `ReadP`. In this case we know an IPv4 will never have the same format as an IPv6.

I hope I have highlighted how useful parser combinators are. As I said this was really a game changer for parsing the different inputs of the AOC.

## Monads
They say it is impossible to talk about Haskell without mentioning monads, so here we are. I had heard about monads, but I never got a clear idea about what they were, so it was nice finally knowing about them. I would lie if I said I know every aspect of them, but I have used a few of them. There are some that are somewhat simple like the `IO` and `Maybe` monads. `Maybe` was the easiest one for me, because of its resemblance with Rust's `Option`. The idea that a value can be there or not is something rather simple also, even if it does have some important implications. 

From the monads I used, the one I had the most difficulties with was the `State` monad. One important thing about Haskell is that normal functions are pure. They have a certain input and produce a certain output, but they don't store values of any kind. Nonetheless, there are certain cases in which such thing is useful and I happened to be in one of those situations during the AOC. My takeaway is that, even if stateful computations are possible in Haskell, they are harder to do, because they don't align well with Haskell philosophy. I also have to say they probably get way easier with practice and with a better understanding of monads.

## Wrapping up
Learning Haskell while doing the AOC challenges was hard, some days I felt like the language itself was designed for the task I had to do, but other days I just wanted to go back to a "normal" language. In the end, it was worth it. Even if I had already programmed in a functional style, Haskell still broke my preconceptions several times and forced me to think in a different manner, which I find really valuable. I really liked Haskell and would like another chance to use it.

And that's it! Hope you liked the post! It was more of a story than anything else, but it's something I wanted to share. If you want to check out my solutions to the AOC, you can do so [here](https://github.com/CastilloDel/AdventOfCode2021).
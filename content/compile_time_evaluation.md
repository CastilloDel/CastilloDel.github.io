+++
title = "Compile time evaluation in Nim, Zig, Rust and C++"
date = 2022-04-13
[taxonomies]
tags = ["Nim", "Zig", "Rust", "C++"]
+++

Evaluating code at compile time is a capability that most, if not all, compiled programming languages possess, even if it can't be directly used by the programmer, but rather applied as an optimization. In this post, we'll explore a bit how to execute code at compile time in Nim, Zig, Rust and C++.

## Nim

In Nim, we can use the word `const` to declare a variable that needs to be evaluated at compile time. If we didn't want or can't evaluate something at compile time we will need to use `var` or `let`. The following code will result in a list of string representing the numbers between 1 and 100 to be present in the binary.

```nim
const numbers = toSeq(1..100).map(x => $x) # $ allows us to transform the numbers to string
for num in numbers:
    echo num
```

It's easy to check that's the case. We just need to open the binary with a text editor and check that there is some place where there is a list with all these numbers. Don't worry if there is a lot of strange symbols there! They just mean the binary code there isn't meant to represent ASCII or even UTF characters, but our list of numbers will be there. I found the following, all in the same line and with strange symbols between each number:

```
@1@2@3@4@5@6@7@8@9@10@11@12@13@14@15@16@17@18 ... @96@97@98@99@100
```

Now, that we know the basics we can try to do something funnier. We will try to implement [Fizzbuzz](https://en.wikipedia.org/wiki/Fizz_buzz) (a really famous, but rather simple programming challenge), so that the result is always evaluated at compile time.

```nim
import std/sequtils
import std/sugar

proc fizzbuzz(number: int): string {.compileTime.} =
  if number mod 3 == 0:
    result = "Fizz"
  if number mod 5 == 0:
    result &= "Buzz"
  if result == "":
    result = $number

const numbers = toSeq(1..100).map(x => fizzbuzz(x))
for num in numbers:
    echo num
```

Notice that with this we have introduced some new elements. There is a function call and some mathematic operations. There is also the `compileTime` pragma, which ensures that this function will only be called during compilation and no code will be generated for it. If we check the resulting binary we will find something like this:

```
@1@2@Fizz@4@Buzz@7@8@11@13@14@FizzBuzz@16@17@19@22@23@26@28@29@31 ...
```

It works! It almost seems like it was too easy, right? Executing arbritrary code at compile time seems like something really complex. We will continue thinking about that further down the line, but now we feel optimistic and happy because everything has come out nicely.

But wait... Why do "Fizz", "Buzz" and "FizzBuzz" only appear once? And why are some numbers missing like 6, 9, 20 and 30???

Every number that is missing should be "Fizz", "Buzz" or "FizzBuzz" and these ones only need to appear once, because the same string can be reused. But now we have a new problem, if these strings we have been seeing don't contain "Fizz" more than once, then they can't represent the final array, right?. That's true. What we have been seeing are only the preprocessed strings, the array will be made of pointers to the directions of this strings, that way it's possible to reuse the "Fizz" string.

## Zig

In zig we can evaluate code at compile time using the `comptime` keyword. It should be pretty easy based on our experience with Nim. Something like the following should be enough:

```zig
fn get_fizzbuzz(n: usize) []const u8 {
    if (n % 15 == 0) {
        return "FizzBuzz";
    } else if (n % 3 == 0) {
        return "Fizz";
    } else if (n % 5 == 0) {
        return "Buzz";
    }
    return n // missing: convert n to string
}

pub fn main() anyerror!void {
    const fizzbuzz_numbers = comptime init: {
        var numbers: [100][]const u8 = undefined;
        for (numbers) |*val, i| {
            val.* = get_fizzbuzz(i + 1);
        }
        break :init numbers;
    };
    for (fizzbuzz_numbers) |n| {
        std.debug.print("{s}\n", .{n});
    }
}
```

Notice how we wrap the `fizzbuzz_numbers` initialization in a `comptime` block. This should mostly work. We only need to to convert `n` to a string and we're all set! In Nim it was so easy, surely we can just convert it. Let me search how to do it.

```zig
    return std.fmt.allocPrint(allocator, "{d}", .{n});
```

We can just do something like this! But wait, we need an allocator for this. Why? Well, if we think it through it's clear why. Different numbers need different string sizes to be represented. 10 can be represented in two ASCII characters, but 100 needs three. So this is something that tipycally needs heap allocated memory to work. We don't need to get desperate though! Luckily for us, Zig has our backs!

```zig
    return std.fmt.comptimePrint("{}", .{n});
```

The `comptimePrint` function allows us to do exactly what we need. It performs all the necessary checks at compile time, so the compiled code knows the exact size of the string it needs to handle. This way it can be allocated in the stack. For this to work, we need to mark `n` as a `comptime` value, which means it must be known at compile time. The final code would be the following:

```zig
const std = @import("std");

fn get_fizzbuzz(comptime n: usize) []const u8 {
    if (n % 15 == 0) {
        return "FizzBuzz";
    } else if (n % 3 == 0) {
        return "Fizz";
    } else if (n % 5 == 0) {
        return "Buzz";
    }
    return std.fmt.comptimePrint("{}", .{n});
}

pub fn main() anyerror!void {
    const fizzbuzz_numbers = comptime init: {
        var numbers: [100][]const u8 = undefined;
        for (numbers) |*val, i| {
            val.* = get_fizzbuzz(i + 1);
        }
        break :init numbers;
    };
    for (fizzbuzz_numbers) |n| {
        std.debug.print("{s}\n", .{n});
    }
}
```

If we repeat what we did with the Nim code, we can also see the corresponding strings in the binary. We can also see it clearly using [Compiler Explorer](<https://gcc.godbolt.org/#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:zig,selection:(endColumn:26,endLineNumber:3,positionColumn:26,positionLineNumber:3,selectionStartColumn:26,selectionStartLineNumber:3,startColumn:26,startLineNumber:3),source:'const+std+%3D+@import(%22std%22)%3B%0A%0Afn+get_fizzbuzz(comptime+n:+usize)+%5B%5Dconst+u8+%7B%0A++++if+(n+%25+15+%3D%3D+0)+%7B%0A++++++++return+%22FizzBuzz%22%3B%0A++++%7D+else+if+(n+%25+3+%3D%3D+0)+%7B%0A++++++++return+%22Fizz%22%3B%0A++++%7D+else+if+(n+%25+5+%3D%3D+0)+%7B%0A++++++++return+%22Buzz%22%3B%0A++++%7D%0A++++return+std.fmt.comptimePrint(%22%7B%7D%22,+.%7Bn%7D)%3B%0A%7D%0A%0Apub+fn+main()+anyerror!!void+%7B%0A++++const+fizzbuzz_numbers+%3D+comptime+init:+%7B%0A++++++++var+numbers:+%5B100%5D%5B%5Dconst+u8+%3D+undefined%3B%0A++++++++for+(numbers)+%7C*val,+i%7C+%7B%0A++++++++++++val.*+%3D+get_fizzbuzz(i+%2B+1)%3B%0A++++++++%7D%0A++++++++break+:init+numbers%3B%0A++++%7D%3B%0A++++for+(fizzbuzz_numbers)+%7Cn%7C+%7B%0A++++++++std.debug.print(%22%7Bs%7D%5Cn%22,+.%7Bn%7D)%3B%0A++++%7D%0A%7D'),l:'5',n:'0',o:'Zig+source+%231',t:'0')),k:55.38833031495245,l:'4',m:100,n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:z090,filters:(b:'1',binary:'0',commentOnly:'1',demangle:'0',directives:'1',execute:'1',intel:'0',libraryCode:'0',trim:'1'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:zig,libs:!(),options:'',selection:(endColumn:1,endLineNumber:1,positionColumn:1,positionLineNumber:1,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:1,tree:'1'),l:'5',n:'0',o:'zig+0.9.0+(Zig,+Editor+%231,+Compiler+%231)',t:'0')),k:44.61166968504757,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4>).

I liked Zig approach, it made the different problems be clear, but also gave us the appropiate tools to overcome them. Nim, on the other hand, just handled everything for us which was really nice. It is also really interesting to look at [the source code of the `comptimePrint` function](https://github.com/ziglang/zig/blob/f8d2b87fa122a948e2c8e1056e2ec4cfd4cf01bf/lib/std/fmt.zig#L1935). It is implemented outside the language, in the standard library, and really takes advantage of Zig's compile time execution capabilities.

## Rust

We can also try to do the same in Rust! In Rust we use the keyword `const` to declare a function that can be evaluated at compile time. This restricts the operations we can do. For example, we can't use the `String` type, because it uses heap allocations. So our function should return a `&'static str`, which means a reference to a string that lives throughout the entire program (usually because the data is embedded in the binary).

```rust
const fn get_fizzbuzz_equivalent(number: usize) -> &'static str {
    if number % 15 == 0 {
        "FizzBuzz"
    } else if number % 3 == 0 {
        "Fizz"
    } else if number % 5 == 0 {
        "Buzz"
    } else {
        &number.to_string() // Obviously this doesn't work for the reasons we discussed previously
    }
}
```

This is a nice first approach, but we have the same problem we had with Zig. Notice that Rust also makes the problem really clear, we can't use a String inside a `const` function. We could use something like [const_format](https://docs.rs/crate/const_format/latest) to get what we need. We would have the following code:

```rust
use const_format::formatcp;

const fn get_fizzbuzz_equivalent<const number: usize>() -> &'static str {
    if number % 15 == 0 {
        "FizzBuzz"
    } else if number % 3 == 0 {
        "Fizz"
    } else if number % 5 == 0 {
        "Buzz"
    } else {
        formatcp!("{}", number)
    }
}
```

We need to declare number as a const generic, to indicate it must be known at compile time(the arguments of a `const fn` don't need to be known at compile time, if they aren't the function will just run normally at runtime, but we can't call `formatcp!` at runtime). But even with that, `formatcp!` doesn't work nicely with const generics. I think it should be possible to do it, but I couldn't find a simple way to do it.

## C++

C++'s `constexpr` is similar to Rust's `const`, so we can do something like this:

```c++
constexpr char* get_fizzbuzz(int number) {
    if (number % 15 == 0) {
        return "FizzBuzz";
    } else if (number % 3 == 0) {
        return "Fizz";
    } else if (number % 5 == 0) {
        return "Buzz";
    }
    return number; // convert to string
}
```

Again, we have the same problem as before. In C++, we can solve it with the [`to_chars` function](https://en.cppreference.com/w/cpp/utility/to_chars) from the standard library. The downside is that to use `to_chars` we need to first declare an array with a fixed length. This is, of course, a valid solution, but we could be using some unnecessary bytes. In the Zig and Nim solution we don't need to that because the compiler calculated the actual necessary length.

## Conclusion

I wanted to bring attention to the fact that a lot of operations that we think are really simple, like converting a number to string, aren't actually that simple.

Of course, there is a lot more to talk about this topic, but I think this rather short post can be interesting to some people. I did learn a lot writing and experimenting about the topic. If you reached the end, thank you! I hope you have learned something.

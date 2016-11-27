---
layout: post
title: Prototyping with Runtime Errors in&nbsp;Elm
hidden: true
sitemap: false
---
While I was listening to [The Changelog podcast episode #198](https://changelog.com/podcast/198), I caught Chris Allen saying something along these lines:

> Contrary to popular belief, the Haskell type system has some escape hatches. For example, you can put an error value in the definition of a function if you wanna say it has a type but you don't have time to implement it right now.
>
> <!--more-->
> This turns out to be a really powerful technique, because it means that I can work in terms of types alone without having written the code or implemented any of the functions and then test combining the functions together without executing them, just seeing, just asking my REPL what type would it be if I composed functions `f`, `g`, and `h` – given that I've only defined their types, I haven't implemented them, evaluating them would produce a runtime error – tell me what happens. And then I can figure out if I'm figuring out the right design or a combination of functions that achieves what I want before I've done any real work.
>
> (…)
>
> Basically the idea is that the types are this kind of a way to step away from the specifics so that
> you could make certain that you're thinking the right thoughts before you do all the work upfront.

Chris most probably meant something like this:

```haskell
foo :: String -> [String]
foo = error "Oh hai!"

bar :: [String] -> [String]
bar = error "Nope."
```

`foo` and `bar` have their type signatures written out, but instead of having an actual implementation, they raise a runtime error. These functions can be then used with other functions, for example in the repl. I'll show it in a second, but first let's see how we would implement something like that in Elm.

As for the first approach, we're going to directly translate Haskell to Elm. Instead of `error`, we're going to use [`Debug.crash`](http://package.elm-lang.org/packages/elm-lang/core/5.0.0/Debug#crash).

```haskell
foo : String -> List String
foo = Debug.crash "Oh hai!"

bar : List String -> List String
bar = Debug.crash "Nope."
```

This code does compile, but if we try to include it in the repl…[^1]

[^1]: If you wish to load this snippet into the repl, you need to put it into a module first. The easiest way to do that is to put `module Main exposing (..)` as the first line of the file, add the function definitions and save the file as `Main.elm`.

```haskell
> import Main exposing (..)
/Users/rav/Projects/prototyping-with-runtime-errors/repl-temp-000.js:556
		throw new Error(
		^

Error: Ran into a `Debug.crash` in module `Main` on line 9
The message provided by the code author is:

    Nope.
```

Well, this is not good. To fix this, we need to add an argument to the definitions of the functions. This will stop Elm from immediately evaluating the right side of the function definition (and crashing our program).[^2]

[^2]: To get a better grasp of it, write down two versions of that function—one with an argument on the left side of the equation and one without it—and compile them with `elm make Main.elm --output main.js`. Then open the JS file and see how their implementations differ.

```haskell
foo : String -> List String
foo x = Debug.crash "Oh hai!"

bar : List String -> List String
bar x = Debug.crash "Nope."
```

Now we should be able to import the module, compose some functions and get some feedback from the repl and the compiler.

```haskell
> import Main exposing (..)
> baz = foo >> bar
<function:_user$project$Repl$baz> : String -> List String
```

This works without a problem, since we're composing `foo : String -> List String` with `bar : List String -> List String`. After doing the composition with [`>>`](http://package.elm-lang.org/packages/elm-lang/core/4.0.5/Basics#), Elm interprets `baz` as `String -> List String`.

`baz = foo >> bar` is equivalent to `baz x = bar (foo x)`. When we call `baz`, Elm first calls `foo` with the argument `x` and then `bar` with the result of `foo x`.

And now let's try the other way around:

```haskell
> bux = bar >> foo
-- TYPE MISMATCH ----------------------------------- repl-temp-000.elm

The right argument of (>>) is causing a type mismatch.

4|       bar >> foo
                ^^^
(>>) is expecting the right side to be a:

    List String -> c

But the right side is:

    String -> List String

Hint: With operators like (>>) I always check the left side first. If it seems fine, I assume it is correct and check the right side. So the problem may be in how the left and right arguments interact.
```

The second definition, `bux`, causes some problems. That's because we're trying to compose `bar : List String -> List String` with `foo : String -> List String` and the type of the return value of `bar` (`List String`) doesn't match the type of the only argument of `foo` (`String`).

Here lies the power of this technique. **Without writing the implementation of `foo` and `bar`, we were able to get some feedback from the compiler and see if our ideas make sense in the grand scheme of things.** To give you a "real world example", you could have a big `update` function in your Elm app and a plan to split it into smaller helpers. You could then prototype with runtime errors and check if the plan is going to pan out or if you forgot about something.

In general, this approach is a handy tool to have in your toolbox.

If you enjoyed this article, take a look at [_Type Bombs in Elm_ by Kris Jenkins](http://blog.jenkster.com/2016/11/type-bombs-in-elm.html). It describes an approach which is useful in an opposite situation – when you don't know the exact type and would like the compiler to tell you what it expects there.

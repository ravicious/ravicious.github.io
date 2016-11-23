---
layout: post
title: Prototyping with Runtime Errors in&nbsp;Elm
hidden: true
sitemap: false
---
While I was listening to [The Changelog podcast episode #198](https://changelog.com/podcast/198), I caught Chris Allen saying something along these lines:

> Contrary to popular belief, the Haskell type system has some escape hatches. For example, you can put an error value in the definition of a function if you wanna say it has a type but you don't have time to implement it right now.
>
<!--more-->
> This turns out to be a really powerful technique, because it means that I can work in terms of types alone without having written the code or implemented any of the functions and then test combining the functions together without executing them, just seeing, just asking my REPL what type would it be if I composed functions `f`, `g`, and `h` – given that I've only defined their types, I haven't implemented them, evaluating them would produce a runtime error – tell me what happens. And then I can figure out if I'm figuring out the right design or a combination of functions that achieves what I want before I've done any real work.
>
> (…)
>
> Basically the idea is that the types are this kind of a way to step away from the specifics so that
> you could make certain that you're thinking the right thoughts before you do all the work upfront.

Chris most probably meant something like this:

```haskell
foo :: Bool -> a
foo = error "Oh hai!"

bar :: [a] -> [a]
bar = error "Nope."
```

`foo` and `bar` have their type signatures written out, but instead of having an actual implementation, they raise a runtime error. Then these functions can be used with other functions, for example in the repl. I'll show it in a second, but first let's see how we would implement something like that in Elm.

As for the first approach, we're going to directly translate Haskell to Elm. Instead of `error`, we're going to use [`Debug.crash`](http://package.elm-lang.org/packages/elm-lang/core/5.0.0/Debug#crash).

```haskell
foo : Bool -> a
foo = Debug.crash "Oh hai!"

bar : List a -> List a
bar = Debug.crash "Nope."
```

This code does compile, but if we try to include it in the repl…[^1]

[^1]: If you wish to load this snippet into the repl, you need to put it into a module first. The easiest way to do that is to put `module Main exposing (..)` as the first line of the file, add the snippet and save the file as `Main.elm`.

```haskell
> import Main exposing (..)
/Users/rav/Projects/prototyping-with-runtime-errors/repl-temp-000.js:556
		throw new Error(
		^

Error: Ran into a `Debug.crash` in module `Main` on line 9
The message provided by the code author is:

    Nope.
```

Well, this is not good. To fix this, we need to add an argument to the definitions of the functions. I'll explain why it helps in a second.

```haskell
foo : Bool -> a
foo x = Debug.crash "Oh hai!"

bar : List a -> List a
bar x = Debug.crash "Nope."
```

Now we should be able to import the module, try to compose some functions and get some feedback from the repl and the compiler.

```haskell
> import Main exposing (..)
> baz = foo >> bar
<function:_user$project$Repl$baz> : Bool -> List a
```

This works without a problem, since we're trying to compose `foo : Bool -> a` with `bar : List a -> List a`. After doing the composition with [`>>`](http://package.elm-lang.org/packages/elm-lang/core/4.0.5/Basics#), Elm interprets `baz` as `baz : Bool -> List a`. The composition with `>>` is the equivalent of writing `baz x = bar (foo x)`. First we call `foo` with the argument `x` and then `bar` with the result of `foo x`.

And now the other way around:

```haskell
> bux = bar >> foo
-- TYPE MISMATCH ----------------------------------- repl-temp-000.elm

The right argument of (>>) is causing a type mismatch.

4|       bar >> foo
                ^^^
(>>) is expecting the right argument to be a:

    List a -> b

But the right argument is:

    Bool -> a

Hint: I always figure out the type of the left argument first and if it is acceptable on its own, I assume it is "correct" in subsequent checks. So the problem may actually be in how the left and right arguments interact.
```

The second definition, `bux`, causes some problems. That's because we're trying to compose `bar : List a -> List a` with `foo : Bool -> a` and the type of the return value of `bar` (`List a`) doesn't match the type of the only argument of `foo` (`Bool`).

Here lies the power of the technique described by Chris. Without writing the implementation for `foo` or `bar`, we were able to get some feedback from the Elm compiler and see if our ideas make sense in the grand scheme of things. To give you a "real world example", you could have a big `update` function in your Elm app and a plan to split it into smaller helpers. With this technique, you could check if the plan is going to pan out or if you forgot to take care of something.

In general, this approach is a handy tool to have in your toolbox.

## Elm is not lazy (but we can be)

Why did it work in Haskell, but in Elm we had to add an argument? It has to do with [Haskell's laziness](https://www.explainxkcd.com/wiki/index.php/1312:_Haskell).

Haskell is lazily evaluated, Elm is strictly evaluated. You may want to read [how Evan explains why Elm is not lazy](https://groups.google.com/forum/#!topic/elm-discuss/9XxV9L0zoA0). But you don't have to grok the whole laziness thing to understand why we had to add that extra argument. Let's look at the compiled source of these two functions: one that takes an argument and one that doesn't.

```haskell
withArgument : Bool -> a
withArgument bool =
    Debug.crash "Oh hai!"

withoutArgument : Bool -> a
withoutArgument =
    Debug.crash "Oh hai!"
```

Putting this snippet into `Main.elm` and executing `elm make Main.elm --output main.js` in the console is going to output something like this in `main.js`:

```javascript
var _user$project$Main$withArgument = function (bool) {
	return _elm_lang$core$Native_Utils.crash(
		'Main',
		{
			start: {line: 6, column: 5},
			end: {line: 6, column: 16}
		})('Oh hai!');
};

var _user$project$Main$withoutArgument = _elm_lang$core$Native_Utils.crash(
	'Main',
	{
		start: {line: 11, column: 5},
		end: {line: 11, column: 16}
	})('Oh hai!');
```

`withArgument` is a function which expects to receive a bool and then calls `Debug.crash`, while the definition of `withoutArgument` calls `Debug.crash` immediately.

This becomes even more clear after we compile Main.elm as an HTML file (`elm make Main.elm`) and try to open it. The compiled JavaScript code is going to crash as soon as it encounters the definition of `withoutArgument`.

---

If you enjoyed this article, take a look at [_Type Bombs in Elm_ by Kris Jenkins](http://blog.jenkster.com/2016/11/type-bombs-in-elm.html). It described an approach which is useful in an opposite situation – when you don't know the exact type and would like the compiler to tell you what it expects there.

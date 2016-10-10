---
layout: post
title: The Polyglot Approach to&nbsp;Getting Better at&nbsp;Modeling the&nbsp;State and Writing Property Tests in Elm
---
When learning a&nbsp;new programming language, there's usually this problem of "What should I do now?". Nowadays, many websites offer a&nbsp;clear path for beginners to follow (shout out to [exercism.io](http://exercism.io)!). However, with relatively new languages like Elm, it still may be hard to find materials on a&nbsp;given topic.

But what if we could… use resources written for other languages?
<!--more-->

I had the luck of stumbling upon [Mark Seemann's talk _Types + Properties = Software_](https://vimeo.com/162036084). It's about modeling the state with types in such a&nbsp;way that illegal states become unrepresentable. This makes writing property-based tests and applications themselves much easier.

I liked the talk so much that I decided to dive deeper into this topic. Turns out [Mark wrote a&nbsp;whole article series](http://blog.ploeh.dk/2016/02/10/types-properties-software/) about that and on _F# for fun an profit_ there's [another wonderful series of posts on designing with types](https://fsharpforfunandprofit.com/series/designing-with-types.html).

But yeah, these two series are written with F# in mind. However, I'm going to show you that Elm has all the required tools and a&nbsp;type system powerful enough to implement all the concepts laid out in those articles!

# What does it mean to make illegal states unrepresentable?

Modeling the state is hard. That's especially true if you're coming from a&nbsp;language where the only data structures at hand are objects and objects with methods.[^1] You may not know how to harness the type system to help you with this task.

[^1]: [`https://twitter.com/moonpolysoft/status/769629482923597825`](https://twitter.com/moonpolysoft/status/769629482923597825)

If you're following the Elm community, you may have heard just recently about making illegal states unrepresentable – [Richard Feldman's _Making Impossible States Impossible_](https://www.youtube.com/watch?v=IcgmSRJHu_8) is an excellent talk with many practical examples.

But what does it really mean to make impossible states impossible?

In short, **the type system enables you to describe the program in such a&nbsp;way that it's impossible to get it into a&nbsp;nonsensical state**. You get compiler errors if you ever try to. It tells you something you wrote doesn't make much sense even before you run the program.

For example, let's say in our app we need a&nbsp;way to check if the pie is being prepared in the kitchen or if it's ready for everybody to eat. A&nbsp;naive approach would be to try to represent this state as a&nbsp;string:

```haskell
type alias Model = {
  pieState: String
}

-- Later in the code:

model = {
  pieState = "inKitchen"
}
-- …
{ model | pieState = "ready" }
```

This could sort of work. However, there's nothing stopping us or other contributors from making a&nbsp;mistake and changing `pieState` to an empty string or [`"should've invented the universe first!"`](https://www.youtube.com/watch?v=5_vVGPy4-rc).

A better approach is to use [a union type](https://guide.elm-lang.org/types/union_types.html):

```haskell
type PieState = InKitchen | Ready

type alias Model = {
  pieState: PieState
}

-- Later in the code:

model = {
  pieState = InKitchen
}
-- …
{ model | pieState = Ready }
```

If somebody ever tries to set `pieState` to anything else than `InKitchen` or `Ready`, they're going to get a&nbsp;compiler error!

# What is property-based testing?

A bit contrived, but nonetheless good example to explain property-based tests is a&nbsp;function which reverses a&nbsp;list. Traditional tests consist of checking whether the thing under the tests returns the right output given a&nbsp;specific input.  We might say that given `["foo", "bar", "baz"]`, our function returns `["baz", "bar", "foo"]`. But that's just one case and we'd like to be sure the function works with anything people throw at it!

We could write a&nbsp;bunch of other tests, but this would not exhaust all the possible inputs. Also, we definitely don't want to write dozens of tests by hand. Instead, we can generate them!

**So what are property-based tests? These are tests which say that for all possible inputs, the given property holds true.** The testing library takes those tests, generates a&nbsp;configurable number of random inputs and checks if the property is true for all of the generated inputs.

It's worth noting that different communities give different names for these tests. From [elm-test docs](http://package.elm-lang.org/packages/elm-community/elm-test/2.1.0/Test#fuzz):

> These are called "fuzz tests" because of the randomness. You may find them elsewhere called property-based tests, generative tests, or QuickCheck-style tests.

While it may sound scary and "mathy" at first, looking at concrete examples should help to clear things up.

The biggest challenge with property-based testing is… Well, coming up with the right set of properties. What are the properties of the mentioned reversing function? One of them would be that sorting the list and then sorting the result again gives us the list we began with. How would we express that using just words? Let's take the bolded sentence from two paragraphs above and rephrase it:

_for all instances of some kind of inputs…_ : for all lists…

_…the given property holds true_ : …reversing the list twice returns the original list

In terms of code, this property looks like this:

```haskell
reverse (reverse list) == list
```

The only problem left is generating random inputs. This is the issue the libraries for fuzz testing solve. If we look at [the docs for the `fuzz` function](http://package.elm-lang.org/packages/elm-community/elm-test/2.1.0/Test#fuzz) of the brilliant elm-test library, we can see that the `fuzz` function takes three arguments:

1. `Fuzzer a` – The so-called fuzzer which is going to generate random values of type `a`.
2. `String` – A&nbsp;string describing the test.
3. `(a -> Expectation)` – A&nbsp;function representing a&nbsp;property. The function takes a&nbsp;value of type `a` and returns [an expectation](http://package.elm-lang.org/packages/elm-community/elm-test/2.1.0/Expect) – which is just a&nbsp;fancy way of saying that the given value should be equal to the expected value, for example:

```haskell
Expect.equal [4, 3, 2, 1] (reverse [1, 2, 3, 4])
```

From this function signature we can figure out that our role in writing property-based tests is to come up with a&nbsp;property expressed as a&nbsp;standard Elm function and a&nbsp;fuzzer—which is also a&nbsp;function—that's going to generate random values for the property function. elm-test is then going to take the property function, generate dozens or hundreds or thousands of random values (the number is up to us) and check if all the generated values satisfy the property.

Just as traditional tests, this method also doesn't exhaust all possible inputs, but it's able to come up with test cases which you'd never think of when testing in the traditional way. For example, [fuzz tests helped to find bugs in Clojure](https://www.youtube.com/watch?v=JMhNINPo__g).[^2]

[^2]: It's [CLJ-1285](http://dev.clojure.org/jira/browse/CLJ-1285) if you don't have the time to watch the whole talk.

That is not to say traditional tests are in opposition to property-based tests – they are quite complimentary and you'll often find yourself using both in your projects.

There's a&nbsp;little more to property-based tests than that. You can find out about it either by watching [the mentioned Mark Seemanns's talk](https://vimeo.com/162036084) or [the intro to property testing](https://www.youtube.com/watch?v=zi0rHwfiX1Q) done by John Hughes – one of the people behind QuickCheck, the original Haskell library for generative tests released in 1999.

# How do unrepresentable states and property tests mesh together?

If you design the program in such a&nbsp;way that invalid states are unrepresentable, it's much easier to define fuzzers for those states. As Mark Seemann put it:

> With the algebraic data types available in F# or Haskell, you can design your types so that illegal states are unrepresentable. (…) This makes it much easier to test the behaviour of your system with Property-Based Testing, because you can declaratively state that you want your Property-Based Testing framework to provide random values of a&nbsp;given type. The framework will give you random values, but it can only give you valid values.

Isn't it nice? If you take the example with the pie state, writing property tests for it may seem like overengineering, as there are only two states to choose from. As you progress through Mark's posts, you'll see he was right in what he said.

# What are the Elm libraries for property tests?

From the available options, I can recommend the mentioned [elm-test library](http://package.elm-lang.org/packages/elm-community/elm-test/latest). It has a&nbsp;built in support for running fuzz tests. It comes with [a bunch of predefined fuzzers](http://package.elm-lang.org/packages/elm-community/elm-test/2.1.0/Fuzz), but it also allows you to write [custom fuzzers](http://package.elm-lang.org/packages/elm-community/elm-test/2.1.0/Fuzz#custom) by combining functions delivered by two other libraries.

And that's it! There's another tool required to actually run the tests, but you'll learn that from the elm-test readme.

# Alright, unrepresentable states, property testing, the F# articles and Elm tools. How do I make all of it work together?

So, it has come to this. You know something about modeling the state, about property-based tests and you know the tools you can use in Elm. Now is the time to read [the _Types + Properties = Software_ series](http://blog.ploeh.dk/2016/02/10/types-properties-software/).

The way to approach the series I can recommend is to read an article, understand its contents (don't worry, Mark is a&nbsp;good writer) and try to implement what has been shown in the article, but in Elm instead of F#.

You may be terrified that there's another new thing to learn while you were just trying to practice Elm. Don't be scared, you can do this! Although I have a&nbsp;bit of a&nbsp;prior experience with languages similar to F#, I never did F# myself. Code in F# is very similar to Elm code in terms of how it looks and how it works.

If you ever get lost, I prepared [a repo with my solution to KataTennis in Elm](https://github.com/ravicious/kata-tennis). I tried to make one commit for each article from the series, so it shouldn't be too hard to find out how I solved a&nbsp;particular challenge. Don't be afraid to ask questions – I'm new to Elm too, so I'd love to get some feedback and discuss alternative solutions!

# What are the biggest differences between F# and Elm implementations?

While I was going through KataTennis, it hit me how explicit Elm is. There's no magic. I was able to figure out everything by looking at the type signatures, then maybe reading the docs for a&nbsp;particular function and then doing the same with another function.

In case of F#, it's a&nbsp;slightly different story. As you get to Mark explaining the first property test, you'll find out that he didn't have to create a&nbsp;custom fuzzer for his union type. That's because [FsCheck (the F# library) can do that for you](http://fscheck.github.io/FsCheck/TestData.html#Default-Generators-and-Shrinkers-based-on-type).

Does it make either language better than the other? I don't think so. For me, it's a&nbsp;matter of tradeoffs the language designers make and whether those tradeoffs fit my preferred way of working. It boils down to what features the designers chose to offer and to the design of particular libraries.

While Mark was happily strolling with the default FsCheck fuzzers, I had to implement a&nbsp;new fuzzer for each of the custom union types. Again, it's hard for me to make a&nbsp;case _for_ or _against_ a&nbsp;given language out of this comparison. I liked how Elm made me pick the right semantics for each fuzzer, but I bet there are people who would label that as "boilerplate".

---

I hope you're going to enjoy KataTennis as much as I did! If you wish to comment, here's a link to [discussion on /r/elm](https://www.reddit.com/r/elm/comments/56tyrk/the_polyglot_approach_to_getting_better_at/).

The resources which were mentioned throughout this post are as follows:

* [Types + Properties = Software – Mark Seemann @ NDC London 2016](https://vimeo.com/162036084)
* [Mark Seemann's _Types + Properties = Software_ series](http://blog.ploeh.dk/2016/02/10/types-properties-software/)
* [The _"Designing with types"_ series on F# for fun and profit](https://fsharpforfunandprofit.com/series/designing-with-types.html)
* [Making Impossible States Impossible – Richard Feldman @ elm-conf 2016](https://www.youtube.com/watch?v=IcgmSRJHu_8)
* [elm-test](http://package.elm-lang.org/packages/elm-community/elm-test/2.1.0)
* [Testing the Hard Stuff and Staying Sane – John Hughes @ Clojure/West 2014](https://www.youtube.com/watch?v=zi0rHwfiX1Q)
* [My solution to KataTennis written in Elm](https://github.com/ravicious/kata-tennis)

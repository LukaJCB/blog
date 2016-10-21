---
layout: post
title:  "What makes a programming language functional"
date:   2016-07-31 18:31:25 +0100
categories: Functional
tags: Scala F# Haskell Elm JavaScript C# Rust Swift Kotlin Ruby Python Java 
---

What makes a language functional? Well, that's a good question!
Some people say that lambdas make a language a functional language. 
Others say it's the ability to bind functions to variables. But that in itself isn't really all it takes right?
After all, we could do this with function pointers in C! And I don't think many would claim C is a functional language, right?

In this article I'll try to go over some specific features that I personally think make or break a functional programming language.
Now pay in mind, that a lot of this is going to be fairly subjective, but I'd love to hear your thoughts in the comments!

So without further ado here's what we will take a look at throughout this article (in order of most to last important to functional programming):

* First Class Functions 
* Immutability
* Recursion
* Expression-Oriented Programming
* Currying
* Lazy Evaluation
* Algebraic Data Types
* Other topics (Higher Kinded Types, Existential Types)


## First class functions
Most languages support this nowadays, 
Java 8 brought First Class functions to the language and C++ also included them in C++11.
Having functions be first class basically only really means that you can use them the same way you would use any other value,
 like being able to bind functions to variables and pass them around your application.

Now there have been a lot of different names for what is basically just a couple of concepts, so we'll try and go through them shortly.

### Anonymous Functions AKA Lambda Expressions
Lambdas and anonymous functions are essentially the same thing, it's just a function that doesn't have a binding to an identifier.
So they're just easy ways for creating functions, in the same sense that some languages allow literals for Arrays or dictionaries.
Anonymous functions originate in Alonzo Church's Lambda Calculus, hence the name.
Here's an example in python:
{% gist 75fe403cf5433ae98f0799b5a0e57bde %}
With that we've defined a variable holding a function that will square a given number when applied. Pretty straight forward!

### Lexical Closures
Now closures are often confused with lambdas and it's not difficult to understand why. 
All closures are anonymous functions, but it doesn't work vice versa. 
Closures are special functions, that close over the environment where it was created, meaning it can gain access to values not in its parameter list.
Very similar to how methods can access instance variables. You've probably already heard of the saying "Closures are the poor man's objects".
Let's take a look at an example in Kotlin:
{% gist 1ef4e596d8909077d1723ba909fe842a %}
In this snippet we can see that the function passed to `forEach` gets access to the `sum` variable and can even modify it.
So closures and lambdas go hand in hand, then what's the deal with Higher order functions?

### Higher Order Functions
Most of you have probably already used Higher Order Functions. HOFs are just functions that take another function as a parameter.
So any function that takes a callback function could be considered a HOF. 
Other very notable example include functions like `map`, `filter` `reduce`. 
Here's an example for a `filter`-function in Swift:
{% gist eaf3cef5bee0d1bffde86b25f1ab089c %}
Here predicate is a function that takes a parameter of type `T` and returns a `Bool` depending on if the element matches the predicate or not.
You'll find many of these HOFs in most functional languages and it's almost an absolut must for doing any kind of functional programming.


## Immutability
Immutability plays a big part in FP, a small subset of functional languages are so called "pure" functional languages.
Purity means, that absolutely everything is immutable.
Examples for this are Haskell, Elm and (as the name suggests) PureScript.

So on the one side we've got languages where everything is immutable,
 but on the other side we also have languages where immutability is unavailable.
JavaScript has often been touted as a functional language, butit didn't have any way to make variables immutable until ES6
and even then it's only limited to local variables and there's still no way to declare instance variables to be immutable
 (unless you're using Object.freeze).
Other languages where immutability is also lacking are Python and Ruby.

Now this is not the only metric for how much a given language supports immutability.
Another is of course, how much the language encourages you to write immutable code.
In Java and C# for example you need to add an extra keyword to make a value immutable 
(C# also doesn't allow Type Inference on immutable values).
{% gist 2801f404fe983fe6981f7a7e4567a192 %}

Put this in contrast to Rust: 
{% gist e6d04f0cb82a63b7dce5a0c56b1872c9 %}

It's quite clear which programming language wants to encourage you to use immutable values.
Some compilers (like Rust and Swift) also emit a warning if you use a mutable variable, but don't mutate it.

Lots of functional languages also offer a way to copy an Object or Record, but with one or multiple values modified.
This makes creating a new value almost exactly as easy as modifying the original. 
Here's an example of what that might look like in Elm:
{% gist af689d4cf7409a773dcfcf001b6aa13a %}

Another thing to consider is something traditionally called "const correctness",
 which means that you can't mutate parts of a value if it's declared immutable.
For example this is legal in Java:
{% gist d16354ae4b4a261ae34b0c7e994d54a5 %}
Where as the equivalent would throw a compiler error in C++.

Now the last thing to consider is whether or not your language supports performant immutable data structures.
Languages like Kotlin and Swift support simple readonly data structures,
 that are the same as their mutual counterpart, but not being able to modify it.

Other more functional languages offer us special collections that are optimized for immutability.
These immutable data structures have operators that can modify and return a copy of the original. 
Most of the time, however we don't even need to copy most of the collection and can instead reuse it,
 because we don't need to worry about subsequent code making changes to our data.
So there's no need to defensively copy the whole structure, saving both memory and time!
This is called data sharing and is a huge benefit of immutable data structures.

The coolest collections come from the late Scala community leader Phil Bagwell (R.I.P.),
 essentially they offer amortized O(1) lookups, insertions and deletions on Vectors and HashMaps.
You can find these data structures in Clojures and Scalas standard library as well as in libraries such as Immutable.js. 
There's a lot of super interesting stuff to talk about,
 so if you're interested check out [this great series][persistent cols] on how these data structures actually work.

## Recursion
Most if not all of modern programming languages support the notion of recursion
However it plays a much bigger part in functional programming than in imperative programming.
That is because when programming in a functional way,
 iterating over data structure recursively is not just much more elegant,
 but also the only way to iterate without invoking side-effects.

In order for this to be efficient, most functional languages offer an optimization technique called "Tail Call Optimization".
With this technique it's possible for recursive functions to not increase the size of the call stack.
In other words: the compiler more or less replaces the original function with the equivalent of an imperative loop.

Going into too much detail here would break the scope of the article, so here's the gist of it:
 if the last thing you do (i.e. the "tail" position) in a function is a recursive call to itself,
 the compiler can optimize this to act like iteration instead of recursion.

So if you want to do functional programming in your language without having to worry about stack overflows,
 your language should probably provide some form of TCO.

Some languages make writing tail recursive functions a lot easier by giving you a way to mark them as such.
For example Scala supports a way explicitly annotate a function as tail recursive
 and let the compiler throw an error when it isn't.

{% gist fbbc28f52b44e20753d947affa50dff7 %}
This guarantees that an error is issued whenever tail call optimization cannot be performed by the compiler.

## Expression-Oriented Programming
To understand Expression-Oriented programming we first need to define the difference between expressions and statements.
This is best explained by contrasting return types of different functions.
For example a `void` type means the method is probably a statement, since it doesn't a result.
Everything else is an expression and yields a value when computed. 
Typically the former looks something like `list.sort()` while expressions look like `sorted = sort(list)`.

To put it simply, in Expression-Oriented programming languages everything is an expression!
Which means that everything has a return value.
This is something we aspire to because statements always have side effects and should be avoided as much as possible in functional programming.

Rust, Ruby, Kotlin and more functional languages like Scala, Elm, Haskell or F# all use this paradigm.
This means that for example the `if`-construct always returns a result:

{% gist f561df4429488ff0b67094ac3bc3efee %}

Another neat example are Scala's "for-expressions" which involve some syntactical sugar that looks very similar to for-loops found in imperative languages.

## Currying

Currying is when you break down a function that takes multiple parameters into a series of functions each taking a parameter and returning a new function.
A simple example would be this:

I've written in depth about the workings and advantages of Currying and the most often confused technique of partial application [in an earlier article][currying].
Both Currying and partial application are very useful tools in the functional programming world and most functional languages make it very easy to do so.
For example in Haskell or one of the different ML-derived languages, it's mostly impossible to even write an uncurried version of functions. 
Functions are just curried by default! Here's some Haskell code:
{% gist 4f38aa3e1c0b995b637ab2e2e67c08b2 %}
The type signature of the `add` function is `Integer -> Integer -> Integer`, meaning it takes an Int and returns a function that takes an Int and returns an Int.
This allows us to just pass one argument to the `add` function and get another generic function that adds the passed argument.
Different languages handle this differently, but it's a pretty important tool in the functional programmers toolkit! 

Scala for example lets you partially apply functions easily by passing an underscore `_` as a function parameter.
Swift is a rather curious example, because it supported a very way of writing curried functions, but has since deprecated them without any real replacement.


## Lazy evaluation
Lazy evaluation is a technique to defer the computation of expressions to when they are really needed.
This is in contrast to eager evaluation, where every expression is evaluated immediately. 
Lazy evaluation can be very beneficial when programming in a functional way. 
Let's look at an example of eager evaluation in JavaScript:
{% gist a9cef6a02adbb0f0efca42af6abbb397 %}

In this example each operation returns a new copy immediately once called. 
What we'd like to do is use higher-order functions like `map` and `filter` instead of manually fusing passes,
 but without having to create intermediate data structures and having to iterate the structure multiple times.
This can be solved quite handily using lazy evaluation.

So now let's have a look at equivalent code using lazy evaluation:
{% gist de1db796f69c53accffecc1dec8e2a5d %}
Wait a minute... it's the same basic code! 
Well yeah it is, the truly interesting stuff is happening behind the scenes, but this code can demonstrate a few things. 

Firstly, we no longer create intermediate copys of the list,
 in fact nothing even gets computed untill we access the first element by calling the `head` method.
Secondly, since we're only accessing the first element of the list, all the operations are only applied once and we can save a lot of execution time. 
We do not need to evaluate the whole list, when all we want to do is print out the first element.

In Haskell lazy evaluation is the default, but in most other functional languages it's opt in.
Examples for these are the various Lisps, Scala and F#.


## Algebraic Data Types

Algebraic data types? Aw man, what's this fancy maths stuff? I just want to program cool stuff!
Alright! I won't go into too much detail here, so bear with me for a moment!
Okay, so most functional languages allow you to define simple data types.
These ADTs are simple data cointainers that can be defined recursively.
They can be easily constructed and deconstructed and usually come with built in structural equality checking.

All of this allows us to utilize a technique called "Pattern matching".
Pattern matching is a kind of `switch-case` on steroids, it can do type-testing, it can check exhaustiveness and it can destructure it's arguments.
Let's have a look at an example written in Scala:

{% gist 43b9e1cfde0d5893037d5a08a3c7fa2a %}

This is just a rather simple example, but I'm sure you can imagine how powerful the `match` expression can be.
An ADT can be anything by the way, from Tuples to Lists, to Records.
So Pattern matching is extremely useful because we can decompose any kind of data structure by its shape instead of its contents.

With pattern matching navigating and decomposing data structures becomes very convenient, with a compact syntax.


## Other advanced features

There's two other things I'd like to atleast give a mention, they're both fairly complex and probably warrant a whole article just to get a good understanding.
Furthermore they're both features of a type system, which might be interesting in staticly typed languages, but no so much for dynamic ones.

### Higher kinded Types
The first feature is the ability to create "Higher Kinded Types",
 which can be seen as providing a way to is the ability to generically abstract over things that take type parameters
Here's an example with a Functor in Scala:
{% gist 79b15bd06c689ff9750da111ff75a63a %}
Here `F[_]` could be anything that takes a generic parameter, so `Option[T]` or `List[T]` would both be fine.

### Existential Types

The other feature is called "Existential Types" can be used for several different purposes,
 but what they do is to 'hide' a type parameter for outside use.
Sometimes you don't care about the actual type but only that it exists. 
Existential types can make this a reality without making the type parameter covariant.


## Conclusion

Now I'd like to conclude without telling you which language is a functional language or which one isn't. 
The line is probably more blurred than not and it's impossible to find some objective criteria for a functional language.
We could argue for ages about what or what doesn't constitute one and how we should weigh these features on a scale from 1 to 100,
 instead I think we've got a fair overview of features functional programmers use everyday.

My hope is that after reading this article,
 you understand that lambdas aren't the only criteria and what else might play a role in programming in a functional way.
Yes we can do functional programming in almost any language,
 but in most that would be more cumbersome than we'd like and we should probably strive to use the right tool for the right job.
Once you try out a language that has a lot of these "functional" features, you'll probably find programming with pure functions a lot more pleasant.
And I hope you guys can also enjoy functional programming more once you've got a hold on some of these cool features. 

[persistent cols]: http://hypirion.com/musings/understanding-persistent-vector-pt-1
[currying]: http://lukajcb.github.io/blog/scala/2016/03/08/a-real-world-currying-example.html

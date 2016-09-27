---
layout: post
title:  "Handling null in different languages"
date:   2016-03-27 23:00:25 +0100
categories: Scala
tags: Scala null Option Maybe Java Swift F# Ruby C# Groovy Kotlin
---

Tony Hoare called them his [billion dollar mistake][tony], but we all know them, null references.

We can probably all agree that we've all had our fair share of annoyances with these infamous little things,
so today we're going to explore how different programming languages choose to handle (or not handle) `null` (or `nil`) values.
This has been a topic for some time now and I always wanted to have some kind of comparison between different solutions,
 with their respective advantages and disadvantages.
 
First let's look at an example how NOT to handle `null` in Java 7. 
You've probably seen something like this before and probably hoped never having to do it again.

{% gist 3c904567ee9b16d95caf5679bddcf2cf %}

The first solution I want to look at is the "Safe navigation operator" `?.` in Groovy, which I believe was the first to implement it.
With the Safe navigation operator we can chain the navigation without risking a NullPointerException. 
The above can be written in Groovy like this:

{% gist 679c89a36b0ff6688cd8ef8f65de0273 %}

Last year in 2015, Ruby and C# added this feature as well. 
C# uses the same syntax as Groovy:
{% gist 03de289359bd9a636c84a5f44e1fde90 %}
Ruby went with `&.` to avoid confusion, since Ruby allows questions marks in their method names to signal a returned boolean value:
{% gist f6d9f523c34bd3da71266ec9a782e51e %}

This is a great addition to the respective languages, however it does have one flaw. 
These null-checks are never enforced, so you can forget about them quite easily and introduce subtle bugs into your code base.

Other functional languages, such as Scala, F# and Haskell, take a different approach.
They strive to completely eliminate the null value from their Type System. 

Instead Scala and F# provide an `Option` type, while Haskell provides a very similar `Maybe` type.
A value of type `Option` or `Maybe` can either hold a value of a given type or nothing. 

This way you have to do an explicit check everytime you want to access a value that might not be there. 
It's impossible for you to "forget", because the type system enforces it. 

Java adopted this kind of system in Java 8, which includes other functional programming features. 
We'll see why that's important later. First let's look at an example in F#.

{% gist 6ec04932a20ef9cb191a00e5e5b310d2 %}

And here's the same thing in Java 8:
{% gist 07d5b46b642bca626595447cc1d4ca80 %}

`map` is a higher order function that transforms the value inside the `Option` if it exists and does nothing if it's absent.
`flatmap` does the same thing, however it flattens the result, so you don't get nested Options like this: `Optional<Optional<Optional<String>>>`
The same thing can also be applied in Scala:

{% gist ab52ea5cacdc353aceddde170042cc1a %}

However Scala also has a special syntax using for-comprehensions, which are much more readable if you ask me.
Here's how that would look:


{% gist f0aeadefd123a914c76fa1df9a8ac56c %}

This is a very good way to handle optional values and forching the user to handle null explicitly is a big step forward if you ask me.
However one might argue that the Groovy, C# and Ruby solutions are much shorter and more concise.
That's why in the last few years languages have tried to combine the two concepts.
We're going to have a look at how Swift and Kotlin thrive to combine the two, for maximum readability.

First let's look at an example in Swift:
{% gist 2401adcf6b048e80ae851bfdb2d06eb3 %}

So this is the exact same way Java, Scala and F# do it, but in Swift you can convert it to this:

{% gist 82effd50906ab03df1d0870940602755 %}

So Swift invokes some syntactic sugar similar to Scala to improve readability. 
In the same sense you can write `Optional<String>` as `String?`. 

Kotlin does this the same way:
{% gist 4e852aea3b263495f46e7aab583b6815 %}

So in these examples `firm` would either be a valid string, or `null` in kotlin and `nil` in Swift.
In both languages, if you don't explicitly make your variables optional by adding a `?` after its type, it has to be initialized.
This makes dealing with `null` values much more reasonable. 

So that's the rundown on the different strategies of dealing with `null`. 
I personally really like the syntactic sugar Kotlin, Swift and Scala provide and the absence of a real `null` in Scala, F# and other functional programming are even better.
What do you guys think about this? What's your favorite solution? Let me know!


References:

[Groovy: http://www.groovy-lang.org/operators.html#_safe_navigation_operator](http://www.groovy-lang.org/operators.html#_safe_navigation_operator)

[Kotlin: https://kotlinlang.org/docs/reference/null-safety.html](https://kotlinlang.org/docs/reference/null-safety.html)

[Java 8: http://www.oracle.com/technetwork/articles/java/java8-optional-2175753.html](http://www.oracle.com/technetwork/articles/java/java8-optional-2175753.html)

[Ruby 2.3: https://www.ruby-lang.org/en/news/2015/12/25/ruby-2-3-0-released/](https://www.ruby-lang.org/en/news/2015/12/25/ruby-2-3-0-released/)

[Swift: https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/OptionalChaining.html](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/OptionalChaining.html)

[C# 6: https://msdn.microsoft.com/en-us/library/dn986595.aspx](https://msdn.microsoft.com/en-us/library/dn986595.aspx)

[tony]: http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare

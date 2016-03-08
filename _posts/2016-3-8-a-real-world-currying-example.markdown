---
layout: post
title:  "A real world Currying example"
date:   2016-03-08 14:00:25 +0100
categories: Scala
tags: Scala Currying Partial-Application 
---

I recently came across a situtation where I tried to explain to a colleague the concept of `Currying`, `partial function application` and its benefits.
I proceeded to show him a quick example in `Swift` and tried to refer him to the internet for more complex, but I wasn't able to find a single example that goes beyond the basics.

So in this post, I'm going to try to explain why `Currying` is fun and how it's done in `Scala`.

Firstly I'd like to specify what exactly Currying and partial application is, aswell as the difference between the two.
Currying is the process of transforming a function that takes multiple arguments into a sequence of functions that each have only a single parameter. 
Partial application is different in that it takes an arguement, applies it partially and returns a new function without the passed parameter.

First the prime example for partial application that is found all around the net:
{% highlight scala %}

def add(a: Int)(b: Int) = a + b

val onePlusFive = add(1)(5) // 6

val addFour = add(4)_ // (Int => Int)

val twoPlusFour = addFour(2) // 6

assert(onePlusFive == twoPlusFour) // true

{% endhighlight %}

In this snippet we used partial application to create a function that adds 4 to an `Int`. 
We can do this, by seperating our arguments into different sets of parentheses and calling the function with only one parameter followed by an underscore `_`.
With this multiple parameter lists, we've prepared our function for partial application and, as we'll see later, already curried our function.

Now currying is a little bit more complex, but it explains how we get from a "normal" function to a curried function.
A "normal" `add-function` is of the type `(Int, Int) => Int`. A curried `add-function` would be of type `Int => (Int => Int)`.

Let's take a look at how we can curry a function like this in Scala:
{% highlight scala %}
def curryBinaryOperator[A](operator: (A, A) => A): A => (A => A) = {
		
    def curry(a: A): A => A = {
        (b: A) => operator(a,b)
    }
	
    curry	
}
{% endhighlight %}

We've defined a generic function that will turn any binary operator into a curried function. We can test this out with an add or a multiplication function:

{% highlight scala %}
def add(a: Int, b: Int) = a + b // (Int, Int) => Int
def multiply(a: Int, b: Int) = a * b // (Int, Int) => Int
      
val addCurried = curryBinaryOperator[Int](add) // Int => (Int => Int)
val multiplyCurried = curryBinaryOperator[Int](multiply) // Int => (Int => Int)
{% endhighlight %}

Note here, that our addCurried function is the same as our add function above with the multiple parameter lists. 
So `Scala` makes it very very easy to create curried functions. 
Other functional programming languages like `Haskell` or `F#` also provide you with a very easy way to write curried functions.

Now all of this is pretty basic and is great to get a grasp of the Scala syntax. However it's really not very useful in portraying the benefit of Currying or partial application.
So I've prepared a scenario where it really comes in handy. Let's imagine we wanted a program that deals with premiums for credit card usage.
What we have is a list of credit cards and we'd like to calculate the premiums for all those cards that the credit card company has to pay out.
The premiums themselves depend on the total number of credit cards, so that the company adjust them accordingly.

We already have a function that calculates the premium for a single credit card and takes into account the total cards the company has issued:
{% highlight scala %}

case class CreditCard(creditInfo: CreditCardInfo, issuer: Person, account: Account)

object CreditCard {
    
    def getPremium(totalCards: Int, creditCard: CreditCard): Double = { ... }
    
}

{% endhighlight %}

Now a reasonable approach to this problem would be to map each credit card to a premium and reduce it to a sum.
Something like this:
{% highlight scala %}

val creditCards: List[CreditCard] = getCreditCards()
val allPremiums = creditCards.map(CreditCard.getPremium).sum

{% endhighlight %}

However the compiler isn't going to like this, because `CreditCard.getPremium` requires two parameters.
Partial application to the rescue! We can partially apply the total number of credit cards and use that function to map the credit cards to their premiums. 
All we need to do is curry the `getPremium` function by changing it to use multiple parameter lists and we're good to go.

The result should look something like this:
{% highlight scala %}

val creditCards: List[CreditCard] = getCreditCards()

val getPremiumWithTotal = Convertible.getCredit(creditCards.length)_

val allPremiums = creditCards.map(getPremiumWithTotal).sum

{% endhighlight %}

So, yeah, that's why Currying is awesome. It allows us to easily reuse more abstract functions. 
We can easily create more specialized functions by partially applying curried functions. 
This is especially important because we often need to pass very specific functions to higher order functions like `map`, `filter` or `reduce`.
I hope I was able to shed some light on why Currying is cool and hope you can find use in your next project.

---
layout: post
title: "A tale on Semirings"
date:   2018-11-02 18:31:25 +0100
categories: Functional
tags: Haskell PureScript Applicative Monoid Semiring
---



*Ever wondered why sum types are called sum types?
Or maybe you've always wondered why the `<*>` operator uses exactly these symbols?
And what do these things have to do with Semirings?
Read this article and find out!*

We all know and use `Monoid`s and `Semigroup`s.
They're super useful and come with properties that we can directly utilize to gain a higher level of abstractions at very little cost.
Sometimes, however, certain types can have multiple `Monoid` or `Semigroup` instances.
An easy example are the various numeric types where both multiplication and addition form two completely lawful monoid instances.

In abstract algebra there is a an algebraic class for types with two `Monoid` instances that interact in a certain way.
These are called `Semiring`s (sometimes also `Rig`) and they are defined as two `Monoid`s with some special laws that define the interactions between them.
Because they are often used to describe numeric data types we usually classify them as *Additive* and *Multiplicative*. 
Just like with numeric types the laws of `Semiring` state that multiplication has to distribute over addition and multiplying a value with the additive identity (i.e. zero) absorbs the value and becomes zero.

There are different ways to encode this as type classes and different libraries handle this differently, but let's look at how the Scala library [algebra](https://typelevel.org/algebra/) handles this.
Specifically, it defines a separate `AdditiveSemigroup` and `MultiplicativeSemigroup` and goes from there.

```haskell

class AdditiveSemigroup a where
  (+) :: a -> a -> a

class AdditiveMonoid a where
  zero :: a

class MultiplicativeSemigroup a where
  (*) :: a -> a -> a

class MultiplicativeMonoid a where
  one :: a
```

A `Semiring` is then just an `AdditiveMonoid` coupled with a `MultiplicativeMonoid` with the following extra laws:


1. Additive commutativity, i.e. `x + y === y + x`
2. Right distributivity, i.e. `(x + y) * z === (x * z) + (y * z)`
3. Left distributivity, i.e. `x * (y + z) === (x * y) + (x * z)`
4. Right absorption, i.e. `x * zero === zero`
5. Left absorption, i.e. `zero * x === zero`

To define it as a type class, we simply extend from both additive and multiplicative monoid:

```haskell
class (MultiplicativeMonoid a, AdditiveMonoid a) => Semiring a
```

Now we have a `Semiring` class, that we can use with the various numeric types like `Int`, `Number`, `BigInt` etc, but what else is a `Semiring` and why dedicate a whole blog post to it?

It turns out a lot of interesting things can be `Semiring`s, including `Boolean`s, `Set`s and [animations](https://bkase.github.io/slides/algebra-driven-design/#/).

One very interesting thing I'd like to point out is that we can form a `Semiring` homomorphism from types to their number of possible inhabitants.
What the hell is that?
Well, bear with me for a while and I'll try to explain step by step.

### Cardinality

Okay, so let's start with what I mean by cardinality.
Every type has a specific number of values it can possibly have, e.g. a `Boolean` has cardinality of 2, because it has two possible values: `true` and `false`.

So `Boolean` has two, how many do other primitive types have?
`Int` has 2^32 and `Number` has 2^64.
So far so good, that makes sense, what about something like `String`?
`String` is an unbounded type and therefore theoretically has infinite number of different inhabitants (practically of course, we don't have infinite memory, so the actual number may vary depending on your system).

For what other types can we determine their cardinality?
Well a couple of easy ones are `Unit`, which has exactly one value it can take and also `Void`, which is the "bottom" type in Haskell, which means it has 0 possible values. I.e you can never instantiate a value of `Void`, which gives it a cardinality of 0.

That's neat, maybe we can encode this in actual code.
We could create a type class that should be able to give us the number of inhabitants for any type we give it.
Since we lack dependent types we can't give a type as an input to a function, so instead we just pass a value of `a` to the function:

```haskell
class Cardinality a where
  cardinality :: a -> BigInt
```

Awesome!
Now let's try to define some instances for this type class.
We don't actually need the value passed to `cardinality` so we'll just ignore it:

```haskell
instance booleanCardinality :: Cardinality Boolean where
  cardinality _ = BigInt.fromInt 2

instance intCardinality :: Cardinality Int where
  cardinality _ = pow (BigInt.fromInt 2) (BigInt.fromInt 32)

instance numberCardinality :: Cardinality Number where
  cardinality _ = pow (BigInt.fromInt 2) (BigInt.fromInt 64)

instance unitCardinality :: Cardinality Unit where 
  cardinality _ = BigInt.fromInt 1

instance voidCardinality :: Cardinality Void where
  cardinality _ = BigInt.fromInt 0
```

Alright, this is cool, let's try it out in the REPL!
To do so, we'll use `undefined`, which can be any type at all and annotate it using the type we want.

```haskell
> cardinality (undefined :: Int)
4294967296

> cardinality (undefined :: Unit)
1

> cardinality (undefined :: Number)
18446744073709551616
```

Cool, but this is all very simple, what about things like ADTs?
Can we encode them in this way as well?
Turns out, we can, we just have to figure out how to handle the basic product and sum types.
To do so, let's look at an example of both types.
First, we'll look at a simple product type: `(Boolean, Int8)`.

How many inhabitants does this type have? 
Well, we know `Boolean` has 2 and `Int8` has 256.
So we have the numbers from `-127` to `128` once with `true` and once again with `false`.
That gives us `512` unique instances.
Hmmm....

`512` seems to be double `256`, so maybe the simple solution is to just multiply the number of inhabitants of the first type with the number of inhabitants of the second type.
If you try this with other examples, you'll see that it's exactly true, awesome!
Let's encode that fact in a type class instance:

```haskell
instance tupleCardinality :: (Cardinality a, Cardinality b) => Cardinality (a, b) where
  cardinality _ = cardinality (undefined :: a) * cardinality (undefined :: b)
```

Great, now let's look at an example of a simple sum type: `Either[Boolean, Int8]`.
Here the answer seems even more straight forward, since a value of this type can either be one or the other, we should just be able to add the number of inhabitants of one side with the number of inhabitants of the other side.
So `Either[Boolean, Int8]` should have `2 + 256 = 258` number of inhabitants. Cool!

Let's also code that up and try and confirm what we learned in the REPL:

```haskell
instance eitherCardinality :: (Cardinality a, Cardinality b) => Cardinality (Either a b) where
  cardinality _ = cardinality (undefined :: a) + cardinality (undefined :: b)
```

```haskell
> cardinality (undefined :: (Boolean, Int8))
512

> cardinality (undefined :: (Either Boolean Int8))
258

> cardinality (undefined :: (Either Int (Boolean, Unit)))
4294967298
```

So using sum types seem to add the number of inhabitants whereas product types seem to multiply the number of inhabitants.
That makes a lot of sense given their names!


So what about that homomorphism we talked about earlier? 
Well, a homomorphism is a structure-preserving mapping function between two algebraic structures of the same sort (in this case a semiring).

This means that for any two values `x` and `y` and the homomorphism `f`, we get 
1. `f(x * y) === f(x) * f(y)`
2. `f(x + y) === f(x) + f(y)`

Now this might seem fairly abstract, but it applies exactly to what we just did.
If we *"add"* two types of `Int8` and `Boolean`, we get an `Either Int8 Boolean` and if we apply the homomorphism function, `cardinality` to it, we get the value `258`.
This is the same as first calling `cardinality` on `Int8` and then adding that to the result of calling `cardinality` on `Boolean`.

And of course the same applies to multiplication and product types.
However, we're still missing something from a valid semiring, we only talked about multiplication and addition, but not about their respective identities.

What we did see, though is that `Unit` has exactly one inhabitant and `Void` has exactly zero.
So maybe we can use these two types to get a fully formed Semiring?

Let's try it out!
If `Unit` is `one` then a product type of any type with `Unit` should be equivalent to just the first type.

Turns out, it is, we can easily go from something like `(Int, Unit)` to `Int` and back without losing anything and the number of inhabitants also stay exactly the same.

```haskell
> cardinality (undefined :: Int)
4294967296

> cardinality (undefined :: (Unit, Int))
4294967296

> cardinality (undefined :: (Unit, (Unit, Int)))
4294967296
```

Okay, not bad, but how about `Void`?
Given that it is the identity for addition, any type summed with `Void` should be equivalent to that type. 
Is `Either Void a` equivalent to `a`?
It is! Since `Void` doesn't have any values an `Either Void a` can only be a `Right` and therefore only an `a`, so these are in fact equivalent types.

We also have to check for the absorption law that says that any value mutliplied with the additive identity `zero` should be equivalent to `zero`.
Since `Void` is our `zero` a product type like `(Int, Void)` should be equivalent to `Void`.
This also holds, given the fact that we can't construct a `Void` so we can never construct a tuple that expects a value of type `Void` either.

Let's see if this translates to the number of possible inhabitants as well:

Additive Identity:
```haskell
> cardinality (undefined :: (Either Void Boolean))
2

> cardinality (undefined :: (Either Void (Int8, Boolean)))
258
```

Absorption:
```haskell
> cardinality (undefined :: (Void, Boolean))
0

> cardinality (undefined :: (Void, Number))
0
```

Nice! 
The only thing left now is distributivity. 
In type form this means that `(a, (Either b c))` should be equal to `Either (a, b), (a, c)`.
If we think about it, these two types should also be exactly equivalent, woohoo!

```haskell
> cardinality (undefined :: (Boolean, (Either Int8 Int16))
131584

> cardinality (undefined :: (Either (Boolean, Int8), (Boolean, Int16)))
131584
```

## Higher kinded algebraic structures

Some of you might have heard of the `Semigroupal` or `Apply` type class. 
But why is it called that, and what is its relation to a `Semigroup`?
Let's find out!

First, let's have a look at `Semigroupal`:

```haskell
class Semigroupal f where
  product :: forall a b. f a -> f b -> f (a, b)

```

It seems to bear some similarity to `Semigroup`, we have two values which we somehow combine, and it also shares `Semigroup`s associativity requirement.

So far so good, but the name `product` seems a bit weird.
It makes sense given we combine the `A` and the `B` in a tuple, which is a product type, but if we're using products, maybe this isn't a generic `Semigroupal` but actually a multiplicative one? 
Let's fix this and rename it!

```haskell
class MultiplicativeSemigroupal f where
  product :: forall a b. f a -> f b -> f (a, b)
```

Next, let us have a look at what an additive `Semigroupal` might look like.
Surely, the only thing we'd have to change is going from a product type to a sum type:

```haskell
class AdditiveSemigroupal f where
  sum :: forall a b. f a -> f b -> f (Either a b)
```

Pretty interesting so far, can we top this and add identities to make `Monoidal`s?
Surely we can! For addition this should again be `Void` and `Unit` for multiplication:

```haskell
class (AdditiveSemigroupal f) => AdditiveMonoidal f where
  void :: f Void

class (MultiplicativeSemigroupal f) => MultiplicativeMonoidal f where
  unit :: f Unit
```

So now we have these fancy type classes, but how are they actually useful?
Well, I'm going to make the claim that these type classes already exist in most preludes today, just under different names.

Let's first look at the `AdditiveMonoidal`.
It is defined by two methods, `void` which returns an `f Void` and `sum` which takes an `f a` and an `f b` to create an `f (Either a b)`.

What type class in the Prelude could be similar?
First, we'll look at the `sum` function and try to find a counterpart for `AdditiveSemigroupal`.
Since we gave the lower kinded versions of these type classes symbolic operators, why don't we do the same thing for `AdditiveSemigroupal`?

Since it is additive it should probably contain a `+` somewhere and it should also show that it's inside some type constructor.

```haskell 
(<+>) :: forall f a b. f a -> f b -> f (Either a b)
```

Oh! The `<+>` function already exists in some libraries as an alias for `alt` which can be found on `Alt`, but it's sort of different, it takes two `f a`s and returns an `f a`, not quite what we have here.

Or is it?
These two functions are actually the same, and we can define them in terms of one another as long as we have a functor:

```haskell
sum :: forall f a b. f a -> f b -> f (Either a b)

alt :: forall f a. (Functor f) => f a -> f a -> f a
alt x y =
  let feitheraa = sum x y
  in map merge feitheraa
```

So our `AdditiveSemigroupal` is equivalent to `Alt`, so probably `AdditiveMonoidal` is equivalent to `Plus`, right?

Indeed, and we can show that quite easily.

`Plus` adds an `empty` function with the following definition:

```haskell
empty :: forall a. f a
```

This function uses a universal quantifier for `a`, which means that it works for any `a`, which then means that it cannot actually include any particular `a` and is therefore equivalent to `f Void` which is what we have for `AdditiveMonoidal`.

Excellent, so we found counterparts for the additive type classes, and we've already talked about `MultiplicativeSemigroupal`.
So the only thing left to find out is the counterpart of `MultiplicativeMonoidal`.

I'm going to spoil the fun and make the claim that `Applicative` is that counterpart.
`Applicative` adds `pure`, which takes an `a` and returns an `f a`.
`MultiplicativeMonoidal` adds `unit`, which takes no parameters and returns an `f Unit`.
So how can we go from one to another? 
Well the answer is again using a functor:

```haskell
unit :: f Unit

pure :: forall a. a => f a
pure a = imap (const a) (const ()) unit
```

`Applicative` uses a covariant functor, but in general we could use invariant and contravariant structures as well.
`Applicative` also uses `<*>` as an alias for using `product` together with `map`, which seems like further evidence that our intuition that its a multiplicative type class is correct.

So in the larger ecosystem right now we have `<+>` and `<*>`, is there also a type class that combines both similar to how `Semiring` combines `+` and `*`?

There is, it is called `Alternative`, it extends `Applicative` and `Plus` and if we were super consistent we'd call it a `Semiringal`:


```haskell
class (MultiplicativeMonoidal f, AdditiveMonoidal f) => Semiringal f
```

Excellent, now we've got both `Semiring` and a higher kinded version of it.

If it were available, we could derive a `Semiring` for any `Alternative` the same we can derive a `Monoid` for any `Plus` or `Applicative`.
We could also lift any `Semiring` back into `Alternative`, by using `Const`, just like we can lift `Monoid`s into `Applicative` using `Const`.

To end this blog post, we'll have a very quick look on how to do that.

```haskell 

instance constSemiringal :: Semiring a => Semiringal (Const a) where
  sum fb fc =
    Const ((unwrap fb) + (unwrap fc))

  product fb fc =
    Const ((unwrap fb) * (unwrap fc))

  unit = Const one

  void = Const zero
```

## Conclusion

Rings and Semirings are very interesting algebraic structures and even if we didn't know about them we've probably been using them for quite some time.
This blog post aimed to show how `Applicative` and `Plus` relate to `Monoid` and how algebraic data types form a semiring and how these algebraic structures are pervasive throughout Haskell and other functional programming languages.
For me personally, realizing how all of this ties together and form some really satisfying symmetry was really mind blowing and I hope this blog post can give some good insight on recognizing these interesting similarities throughout the prelude and other libraries based on different mathematical abstractions.
For further material on this topic, you can check out [this talk](https://www.youtube.com/watch?v=YScIPA8RbVE).



## Addendum

This article glossed over commutativity in the type class encodings.
Commutativity is very important law for semrings and the code should show that.
However, since this post already contained a lot of different type class definitions, adding extra commutative type class definitions that do nothing but add laws felt like it would distract from what is trying to be taught.

Moreover I focused on the cardinality of only the types we need, but for completions sake, we could also add instances of `Cardinality` for things like `A -> B` , `Maybe a` or `These a b`.
These are:
```haskell
  cardinality (undefined :: (a -> b)) === 
    pow (cardinality (undefined :: B)) (cardinality (undefined :: a))

  cardinality (undefined :: (Maybe a)) ===
    cardinality (undefined :: a) + 1

  cardinality (undefined :: (These a b)) ===
    (cardinality (undefined :: a)) + (cardinality (undefined :: b)) + (cardinality (undefined :: a)) * (cardinality (undefined :: b))
```

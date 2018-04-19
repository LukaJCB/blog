---
layout: post
title: "Rethinking MonadError"
date:   2018-04-15 18:31:25 +0100
categories: Functional
tags: Haskell PureScript Monad MonadError

---

`MonadError` is a very old type class, hackage shows me it was originally added in 2001, long before I had ever begun doing functional programming, just check the [hackage page](https://hackage.haskell.org/package/mtl-2.2.2/docs/Control-Monad-Error-Class.html).
In this blog post I'd like to rethink the way we use `MonadError` today.
It's usually used to signal that a type might be capable of error handling and is basically like a type class encoding of `Either`s ability to short circuit.
That makes it pretty useful for building computations from sequences of values that may fail and then halt the computation or to catch those errors in order to resume the computation.
It's also parametrized by its error type, making it one of the most common example of multi-parameter type classes.
Some very common instances include `Either` and `IO`, but there are a ton more.

We can divide instances into 3 loosely defined groups:

First we have simple data types like `Either`, `Maybe` or `These` (with `Validat` not having a `Monad` instance). 

Secondly we've got `IO` or `Aff` in PureScript and similar types. These are used to suspend side effects which might have errors and therefore need to be able to handle these.

Thirdly and least importantly, we have monad transformers, which get their instances from their respective underlying monads. Since they basically just propagate their underlying instances we're only going to talk about the first two groups for now.

The simple data types all define `MonadError` instances, but I wager they're not actually used as much. This is because `MonadError` doesn't actually allow us to deconstruct e.g. an `Either` to actually handle the errors. We'll see more on that later, next let's look at the `IO`-like types and their instances.

`IO` currently defines a `MonadError IOException IO`, in PureScript we have `MonadError Error (Aff eff)`. This means that they're fully able to raise and catch errors that might be thrown during evaluation of encapsulated side effects.
Using `MonadError` with these effect types seems a lot more sensical at first, as you can't escape `IO` even when you handle errors, so it looks like it makes sense to stay within `IO` due to the side effect capture. 

The problem I see with `MonadError` is that it does not address the fundamental difference between these two types of instances. I can pattern match an `Maybe a` with a default value to get back an `a`. With `IO` that is just not possible. So these two groups of types are pretty different, when does it actually make sense to abstract over both of them?
Well, it turns out there a few instances where it might be useful, but as we'll see later, I'm proposing something that will be equally useful to both groups.

Now before we continue, let's look at the `MonadError` type class in a bit more detail.
`MonadError` currently comprises two parts, throwing and catching errors.
To begin let's have a look at the `throw` part, sometimes also called `MonadThrow`:

```haskell
class (Monad m) => MonadError e m | m -> e where
  throwError :: e -> m a
```

This looks fine for now, but one thing that strikes me is that the `m` type seems to "swallow" errors.
If we look at `m a` we have no clue that it might actually yield an error of type `e`, that fact is not required to be represented at all.
However, that's not a really big issue, so now let's look at the `catch` part:

```haskell
class (Monad m) => MonadError e m | m -> e where
  ...
  catchError :: m a -> (e -> m a) -> m a
```

Immediately I have a few questions, if the errors are handled, why does it return the exact same type?
Furthermore if this is really supposed to handle errors, what happens if I have errors in the `e -> m a` function? 
This is even more blatant in the `try` function:

```haskell
try :: forall m e a. MonadError e m => m a -> m (Either e a)
```

Here there is no way the outer `m` still has any errors, so why does it have the same type?
Shouldn't we represent the fact that we handled all the errors in the type system?
This means you can't actually observe that the errors are now inside `Either`. That leads to this being fully legal code:

```haskell
try (try (try (try (Just 42))))
```

Another example that demonstrates this is the fact that calling `handleError`, which looks like this:

```haskell
handleError :: forall m e a. MonadError e m => m a -> (e -> a) -> m a
```
also returns an `m a`. This method takes a pure function `e -> a` and thus can not fail during recovery like `catchError`, yet it still doesn't give us any sign that it doesn't throw errors.
For `IO`-like types this is somewhat excusable as something like an unexceptional `IO` is still very uncommon, but for simple data types like `Either` or `Maybe` that function should just return an `A`, since that's the only thing it can be.
Just like with `attempt`, we can infinitely chain calls to `handleError`, as it will never change the type.

Ideally our type system should stop us from being able to write this nonsensical code and give us a way to show anyone reading the code that we've already handled errors.
Now I'm not saying that the functions on `MonadError` aren't useful, but only that they could be more constrained and thus more accurate in their representation. 


For this purpose let's try to write a different `MonadError` type class, one that's designed to leverage the type system to show when values are error-free, we'll call it `MonadBlunder` for now.

To mitigate the problems with `MonadError` we have a few options, the first one I'd like to present is using two different type constructors to represent types that might fail and types that are guaranteed not to. So instead of only a single type constructor our `MonadBlunder` class will have two:

```haskell
class MonadBlunder f g e where
```

Our type class now has the shape `(* -> *) -> (* -> *) -> * -> *`, which is quite a handful, but I believe we can justify its usefulness.
The first type parameter `f` will represent our error-handling type, which will be able to yield values of type `e`.
The second type parameter `g` will represent a corresponding type that does not allow any errors and can therefore guarantee that computations of the form `g a` will always yield a value of type `a`.

Now that we figured out the shape, let's see what we can actually do with it.
For throwing errors, we'll create a `throwError` function that should return a value inside `f`, as it will obviously be able to yield an error.

```haskell
class MonadBlunder f g e where
  throwError :: forall a. e -> f a
```

This definition looks identical to the one defined one `MonadError` so let's move on to error-handling.
For handled errors, we want to return a value inside `g`, so our `catchError` function should indeed return a `g a`:

```haskell
class MonadBlunder f g e where
  ...

  catchError :: forall a. f a -> (e -> f a) -> g a
}
```

Looks good so far, right? 
Well, we still have the problem that the function `e -> f a` might return an erronous value, so if we want to guarantee that the result won't have any errors, we'll have to change that to `g a` as well:

```haskell
class MonadBlunder f g e where
  ...

  catchError :: forall a. f a -> (e -> g a) -> g a
}
```

And now we're off to a pretty good start, we fixed one short coming of `MonadError` with this approach.

Another approach, maybe more obvious to some, might be to require the type constructor to take two arguments, one for the value and one for the error type.
Let's see if we can define `throwError` on top of it:

```haskell
class MonadBlunder b where
  throwError :: forall e a. e -> b e a
  ...
}
```

This looks pretty similar to what we already have, though now we have the guarantee that our type doesn't actually "hide" the error-type somewhere.
Next up is `catchError`. Ideally after we handled the error we should again get back a type that signals that it doesn't have any errors. 
We can do exactly that by choosing an unhabited type like `Void` as our error-type:


```haskell
class MonadBlunder b where
  ...

  catchError :: forall e a. b e a -> (e -> b Void a) -> b Void a
}
```

And this approach works as well, however now we've forced the two type parameter shape onto implementors. This `MonadBlunder` has the following kind `(* -> * -> *) -> *`.
This means we can very easily define instances for types with two type parameters like `Either`.
However, one issue might be that it's much easier to fit a type with two type parameters onto a type class that expects a single type constructor `(* -> *)` than to do it the other way around.

For example try to implement the above `MonadBlunder ` for the standard `IO`.
It's not going to be simple, whereas with the first encoding we can easily encode both `Either` and `IO`. For this reason, I will continue this article with the first encoding using the two different type constructors.

Next we're going to look at laws we can define to make sense of the behaviour we want.
The first two laws should be fairly obvious. 
If we `bind` over a value created by `throwError` it shouldn't propogate:

```haskell
throwError e >>= f === throwError e
```

Next we're going to formulate a law that states, that raising an error and then immediatly handling it with a given function should be equivalent to just calling that function on the error value:

```haskell
throwError e `catchError` f === f e
```

Another law could state that handling errors for a pure value lifted into the `F` context does nothing and is equal to the pure value in the `G` context:

```
pure a `catchError` f === pure a
```

Those should be good for now, but we'll be able to find more when we add more derived functions to our type class.
Also note that none of the laws are set in stone, these are just the ones I came up with for now, it's completely possible that we'll need to revise these in the future.

Now let's focus on adding extra functions to our type class. `MonadError` offer us a bunch of derived methods that can be really useful. For most of those however we need access to methods like `bind` for both `f` and `g`, so before we figure out derived combinators, let's revisit how exactly we define the type class.

The easiest would be to give both `f` and `g` a `Monad` constraint and move on. 
But then we'd have two type classes that both define a `throwError` function extends `Monad`, and we wouldn't be able to use them together, since that would cause ambiguities and as I've said before, the functions on `MonadError` are useful in some cases.

Instead, since I don't really like duplication and the fact that we're not going to deprecate `MonadError` overnight, I decided to extend `MonadBlunder` from `MonadError` for the `F` type, to get access to the `throwError` function.
If `throwError` and `catchError` were instead separated into separate type classes (as is currently the case in the PureScript prelude), we could extend only the `throwError` part.
This also allows us to define laws that our counterparts of functions like `try` and `ensure` are consistent with the ones defined on `MonadError`.
So the type signature now looks like this:
```haskell
class (MonadError f e, Monad g) => MonadBlunder f g e | f -> e, f -> g where
  ...
```

Now since this means that any instance of `MonadBlunder` will also have an instance of `MonadError` on `f`, we might want to rename the functions we've got so far.
Here's a complete definition of what we've come up with with `throwError` removed and `catchError` renamed to `catchBlunder`:

```haskell
class (MonadError f e, Monad g) => MonadBlunder f g e | f -> e, f -> g where
  catchBlunder :: forall a. f a -> (e -> g a) -> g a
```

Now let us go back to defining more derived functions for `MonadBlunder`.
The easiest probably being `handleError`, so let's see if we can come up with a good alternative:

```haskell
handleBlunder :: forall f g e a. MonadBlunder f g e => f a -> (e -> a) -> g a
handleBlunder fa f = catchBlunder fa (pure . f) 
```

This one is almost exactly like `catchBlunder`, but takes a function from `e` to `a` instead of to `g a`. We can easily reuse `catchBlunder` by using `pure` to go back to `e -> g a`.

Next another function that's really useful is `try`.
Our alternative, let's call it `endeavor` for now, should return a value in `g` instead, which doesn't have a `MonadError` instance and therefore can not make any additional calls to `endeavor`:


```haskell
endeavor :: forall f g e a. MonadBlunder f g e => f a -> g (Either e a)
endeavor fa =
  handleBlunder (map Right fa) Left
}
```

The implementation is fairly straightforward as well, we just handle all the errors my lifting them into the left side of an `Either` and map successful values to the right side of `Either`.

Next, let's look at the dual to `try`, sometimes called `absolve` or `rethrow`. 
For `MonadError` it turns an `f (Either e a)` back into an `f`, but we're going to use our unexceptional type again:

```haskell
absolve :: forall f g e a. MonadBlunder f g e => g (Either e a) -> f a
```


But looking at this signature, we quickly realize that we need a way to get back to `f a` from `g a`.
So we're going to add another function to our minimal definition:

```haskell
class (MonadError f e, Monad g) => MonadBlunder f g e | f -> e, f -> g, g -> f where
  catchBlunder :: forall a. f a -> (e -> g a) -> g a
  accept :: g ~> f
```

This function `accept`, allows us to lift any value without errors into a context where errors might be present.

We can now formulate a law that values in `g` never stop propagating, so `bind` should always work, we do this by specifying that calling `handleBlunder` after calling `accept` on any `g a`, is never going to actually change the value:

```haskell
(accept ga) `handleBlunder` f === ga
```

Now we can go back to implementing the `absolve` function:

```haskell
absolve :: forall f g e a. MonadBlunder f g e => g (Either e a) -> f a
absolve gea = accept gea >>= (either throwError pure)
``` 

Now that we've got the equivalent of both `try` and `rethrow`, let's add a law that states that the two should cancel each other out:

```haskell
absolve (endeavor fa) === fa
```

We can also add laws so that `handleBlunder` and `endeavor` are consistent with their counterparts now that we have `accept`:

```haskell
accept (fa `handleBlunder` f) === fa `handleError` f


accept (endeavor fa) === try fa
```

One nice thing about `try`, is that it's really easy to add a derivative combinator that doesn't go to `f e a`, but to the isomorphic monad transformer `ExceptT f e a`.
We can do the exact same thing with `endeavor`:


```haskell
endeavorT :: forall f g e a. MonadBlunder f g e => f a -> ExceptT g e a
endeavorT fa = ExceptT (endeavor fa)
```

One last combinator I'd like to "port" from `MonadError` is the `ensure` function.
`ensure` turns a successful value into an error if it does not satisfy a given predicate.
We're going to name the counterpart `assure`:

```haskell
assure :: forall f g e a. MonadBlunder f g e => g a -> (a -> e) -> (a -> Bool) -> f a
assure ga error predicate =
  accept ga >>= (\a -> 
    if predicate a then pure a else throwError (error a))
```

This plays nicely with the rest of our combinators and we can again add a law that dictates it must be consistent with `ensure`:

```haskell
ensure (accept ga) error predicate === assure ga error predicate
```

Now we have a great base to work with with laws that should guarantee principled and sensible behaviour.
Next we'll actually start defining some instances for our type class.

The easiest definitions are for `Either` and `Maybe`, though I'm not going to cover both, as the instances for `Option` can simply be derived by `Either Unit a`and I'm going to link to the code at the end.
For `Either e a`, when we handle all errors of type `e`, all we end up with is `a`, so the corresponding `g` type for our instance should be `Identity`.
That leaves us with the following definition:

```haskell
instance MonadBlunder (Either e) Identity e where
  catchBlunder :: forall a. Either e a -> (e -> Identity a) -> Identity a
  catchBlunder fa f = case fa of
    Left e -> f e
    Right a -> Identity a

  accept :: Identity ~> Either e
  accept = Right . runIdentity
```

Fairly straightforward, as `Identity a` is just `a`, but with this instance we can already see a small part of the power we gain over `MonadError`.
When we handle errors with `handleBlunder`, we're no longer "stuck" inside the `Either` Monad, but instead have a guarantee that our value is free of errors.
Sometimes it'll make sense to stay inside `Either`, but we can easily get back into `Either`, so we have full control over what we want to do.

Next up, we'll look at `IO` and the type that inspired this whole blog post `UIO`.
`UIO` is equivalent to an `IO` type where all errors are handled and is short for "unexceptional IO".
This would also work for `IO` types who use two type parameters `IO e a` where the first represents the error type and the second the actual value. There you'd choose `IO e a` as the `f` type and `IO Void a` as the `g` type. `IO Void a` there is equivalent to `UIO a`.

As one might expect, you can not simply go from `IO a` to `UIO a`, but we'll need to go from `IO a` to `UIO (Either e a)` instead, which if you look at it, is exactly the definition of `endeavor`.
Now let's have a look at how the `MonadBlunder` instance for `IO` and `UIO` looks:

```haskell
instance MonadBlunder IO UIO IOException where
  catchBlunder :: forall a. IO a -> (e -> a) -> UIO a
  catchBlunder fa f =
    unsafeFromIO (fa `catchError` (accept . f))

  accept :: UIO ~> IO
  accept = runUIO
```

And voila! We've got a fully working implementation that will allow us to switch between these two types whenever we have a guarantee that all errors are handled.
This makes a lot of things much simpler.
For example, if one wants to use `bracket` with `UIO`, you just need to `bind` to the finalizer, as `bind` is always guaranteed to not short-circuit.

We can also define instances for `ExceptT` and `MaybeT` (being isomorphic to `ExceptT f Unit a`), where the corresponding unexceptional type is just the outer `f`, so `endeavor` is just a call to `run`: 

```haskell
instance MonadBlunder (ExceptT e f) f e where
  catchBlunder :: forall a. ExceptT e f a -> (e -> f a) -> f a
  catchBlunder efa f =
    runExceptT efa >>= (\eea -> case eea of
      Left e -> f e
      Right a -> pure a
    )

  accept :: f ~> ExceptT f e
  accept = lift
```

Finally, it's also possible to create instances for other standard monad transformers like `WriterT`, `ReaderT` or `StateT` as long as their underlying monads themselves have instances for `MonadBlunder`, as is typical in mtl.
As their implementations are very similar we'll only show the `StateT` transformer instance:

```haskell
instance (MonadBlunder f g e) => MonadBlunder (StateT s f) (StateT s g) e where
  catchBlunder :: forall a. StateT s f a -> (e -> StateT s g a) -> StateT s g a
  catchBlunder sfa f =
    StateT (\s -> runStateT sfa s `catchBlunder` ((runStateT s) . f))

  accept :: StateT s g ~> StateT s f
  accept = mapStateT accept 
```

In practice this means we can call `catchBlunder` on things like `StateT s IO a` and get back a `StateT s UIO a`. Pretty neat!

## Conclusion

In this article, I've tried to present the argument that `MonadError` is insufficient for principled error handling.
We also tried to build a solution that deals with the shortcomings described earlier.
Thereby it seeks not to replace, but to expand on `MonadError` to get a great variety of error handling capabilities.
I believe the `MonadBlunder` type class, or whatever it will be renamed to, has the potential to be a great addition to the functional community at large.

You can find the full code [here](https://pursuit.purescript.org/packages/purescript-errorcontrol/0.2.1/docs/Error.Control).

Note again, that none of this is final or set in stone and before it arrives anywhere might still change a lot, especially in regards to naming (which I'm not really happy with at the moment), so if you have any feedback of any sorts, please do chime in! Would love to hear your thoughts and thank you for reading this far!

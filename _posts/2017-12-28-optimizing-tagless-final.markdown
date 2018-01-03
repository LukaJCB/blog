---
layout: post
title: "Optimizing Tagless Final"
date:   2018-01-03 18:31:25 +0100
categories: Functional
tags: Haskell PureScript tagless final

---

The following is a blog post written for PureScript, but should be able to work with Haskell with only very few modifications.

The Tagless Final encoding has gained some steam recently, with some people hailing 2017 as the year of Tagless Final.
Being conceptually similar to the Free Monad, different comparisons have been brought up and the one trade-off that always comes up is the lack or the difficulty of inspection of tagless final programs and in fact, I couldn't find a single example on the web.
This seems to make sense, as programs in the tagless final encoding aren't values, like programs expressed in terms of free structures. 
However, in this blog post, I'd like to dispell the myth that inspecting and optimizing tagless final programs is more difficult than using `Free`.

To start with, this blog post is gonna use a different tagless final encoding not based on type classes, but records instead. This allows us to treat interpreters as values.
This technique is described [here](https://gist.github.com/LukaJCB/c59b9e7b41bd1b12e5d4d3de683f6a5f).

Without further ado, let's get into it, starting with our example algebra, a very simple key-value store:

```haskell
newtype KVStore f = KVStore 
  { put :: String -> String -> f Unit
  , get :: String -> f (Maybe String)
  }
```


To get the easiest example out of the way, here's how to achieve parallelism in a tagless final program:

```haskell
program :: forall f m. Parallel m f => KVStore f -> f (Maybe String)
program (KVStore k) = do
  k.put "A" a
  x <- (<>) <$> k.get "B" `parApply` k.get "C"
  k.put "X" (fromMaybe "-" x)
  pure x
```

This programs makes use of the `Parallel` type class, that allows us to make use of the `parApply` combinator to use independent computations with a related `Applicative` type. This is already much simpler than doing the same thing with `Free` and `FreeApplicative`. For more info on `Parallel` check out the docs [here](https://pursuit.purescript.org/packages/purescript-parallel/3.2.0).

However this is kind of like cheating, we're not really inspecting the structure of our program at all, so let's look at an example where we actually have access to the structure to do optimizations with.

Let's say we have the following program:

```haskell
program :: forall f. Apply f => KVStore f -> f (List String)
program mouse (KVStore k) = (\f s _ t -> catMaybes (f : s : t : Nil)) <$>
  k.get "Cats" <*> k.get "Dogs" <*> k.put "Mice" "42" <*> k.get "Cats"
```

Not a very exciting program, but it has some definite optimization potential.
Right now, if our KVStore implementation is an asynchronous one with a network boundary, our program will make 4 network requests sequentially if interpreted with the standard `Apply` instance of something like `Aff`.
We also have a duplicate request with the `"Cats"`-key.

So let's look at what we could potentially do about this.
The first thing we should do, is extract the static information.
The easiest way to do so, is to interpret it into something we can use using a `Monoid`.
This is essentially equivalent to the `analyze` function commonly found on `FreeApplicative`.

Getting this done, is actually quite simple, as we can use `Const` as our `Applicative` data type, whenever the lefthand side of `Const` is a `Monoid`. 
I.e. if `m` has a `Monoid` instance, `Const m a` has an `Applicative` instance.
You can read more about `Const` [here](https://pursuit.purescript.org/packages/purescript-const/3.0.0).

```haskell
import Prelude
import Data.StrMap as M
import Data.Set as S
import Data.Const

analysisInterpreter :: KVStore (Const (Tuple (S.Set String) (M.StrMap String)))
analysisInterpreter = KVStore
  { put : \key value -> Const $ tuple2 S.empty (M.singleton key value)
  , get : \key -> Const $ tuple2 (S.singleton key) M.empty
  }

(Const (program analysisInterpreter))

```

By using a Tuple of `Set` and `Map` as our `Monoid`, we now get all the unique keys for our `get` and `put` operations.
Next, we can use this information to recreate our program in an optimized way.

```haskell
optimizedProgram :: forall f. Apply f => KVStore f -> f (List String)
optimizedProgram (KVStore k) = 
  let (Const (Tuple gets puts)) = program analysisInterpreter
  in traverse (\(Tuple k v) -> k.put k v) (fromFoldable puts) *> traverse k.get (fromFoldable gets)
```

And we got our first very simple optimization.
It's not much, but we can imagine the power of this technique.
For example, if we were using something like `GraphQL`, we could sum all of our `get` requests into one large request, so only one network roundtrip is made.
We could imagine similar things for other use cases, e.g. if we're querying a bunch of team members that all belong to the same team, it might make sense to just make one request to all the team's members instead of requesting them all individually.

Other more complex optimizations could involve writing a new interpreter with the information we gained from our static analysis.
One could also precompute some of the computations and then create a new interpreter with those computations in mind.

Embedding our Applicative program inside a larger monadic program is also trivial:

```haskell
program :: forall f. Apply f => String -> KVStore f -> f (List String)
program mouse (KVStore k) = (\f s _ t -> catMaybes (f : s : t)) <$>
  k.get "Cats" <*> k.get "Dogs" <*> k.put "Mice" mouse <*> k.get "Cats"

optimizedProgram :: forall f. Apply f => String -> KVStore f -> f (List String)
optimizedProgram mouse (KVStore k) = 
  let (Const (Tuple gets puts)) = program mouse analysisInterpreter
  in traverse (\(Tuple k v) -> k.put k v) (fromFoldable puts) *> traverse k.get (fromFoldable gets)

monadicProgram :: forall f. Bind f => KVStore f -> f Unit
monadicProgram (KVStore k) = do
  mouse <- k.get "Mice"
  list <- optimizedProgram (fromMaybe "64" mouse) k
  k.put "Birds" (fromMaybe "128" (head list))

```

Here we refactor our `optimizedProgram` to take an extra parameter `mouse`. Then in our larger `monadicProgram`, we perform a `get` operation and then apply its result to `optimizedProgram`.

So now we have a way to optimize our one specific program, next we should see if we can introduce some abstraction.

First we'll have to look at the shape of a generic program, they usually are functions from an interpreter `algebra f` to an expression inside the type constructor `f`, such as `f a`.

```haskell
type Program alg a = forall f. Applicative f => alg f -> f a
```

The program is only defined by the algebra and the resulting `a`, so it should work for all type constructors `f`.

```haskell
optimize :: forall alg f a m. Applicative f
         => Monoid m 
         => Program alg a
         -> alg (Const m)
         -> m -> f a
         -> alg f
         -> f a
optimize program extract restructure =
  let (Const m) = program extract
  in restructure m
```

Now we should be able to express our original optimization with this new generic approach:

```haskell
optimizedProgram :: forall f. Apply f => String -> KVStore f -> f (List String)
optimizedProgram mouse (KVStore k) =
  optimize program analysisInterpreter (\(Tuple gets puts) -> 
  traverse (\(Tuple k v) -> k.put k v) (fromFoldable puts) *> traverse k.get (fromFoldable gets))
```

So far so good, we've managed to write a function to generically optimize tagless final programs.
However, one of the main advantages of tagless final is that implementation and logic should be separate concerns.
With what we have right now, we're violating the separation, by mixing the optimization part with the program logic part.
Our optimization should be handled by the interpreter, just as the sequencing of individual steps of a monadic program is the job of the target `Monad` instance.

One way to go forward, is to create a typeclass that requires certain algebras to be optimizable.
This typeclass could be written using the generic function we wrote before, so let's see what we can come up with:


```haskell
type OptimizerReqs alg f m =
  { extract :: alg (Const m)
  , rebuild :: m -> alg f -> f (alg f)
  }

class (Monad f, Monoid m) <= Optimizer alg f m | alg -> f , f -> m where
  reqs :: OptimizerReqs alg f m

optimize :: forall alg f m a. Optimizer alg f m
         => Program alg a
         -> alg f
         -> f a
optimize prog interpreter =
  let (Const m2) = prog (reqs :: OptimizerReqs alg f m).extract
  in (reqs.rebuild m2 interpreter) >>= prog
```

This might look a bit daunting at first, but we'll go through it bit by bit.
First we define our type class `Optimizer` parameterized by an algebra `alg :: (* -> *) -> *` and a type constructor `f :: * -> *`.
This means we can define different optimizations for different algebras and different target types.
For example, we might want a different optimization for a production `Optimizer KVStore (EitherT Task e) m` and a testing `Optimizer KVStore Identity m`.
Next, for our interpreter we need a `Monoid m` for our static analysis, so we parametrize the `Optimizer` with an extra parameter `m`. 

The next two functions should seem familiar, the `extract` function defines an interpreter to get an `m` out of our program.
The `rebuild` function takes that value of `m` and the interpreter and produces an `f alg f`, which can be understood as an `f` of an interpreter.
This means that we can statically analyze a program and then use the result of that to create a new optimized interpreter and this is exactly what the `optimize` function does.
This is also why we needed the `Monad` constraint on `f`, we could also get away with returning just a new interpreter `alg f` from the `rebuild` method and get away with an `Applicative` constraint, but we can do more different things this way.


Let's see what our program would look like with this new functionality:

```haskell
monadicProgram :: forall f m. Optimizer KVStore f m => KVStore f -> f Unit
monadicProgram (KVStore k) = do
  mouse <- k.get "Mice"
  list <- optimize (program $ fromMaybe "64" mouse) (KVStore k)
  k.put "Birds" (fromMaybe "128" (head list))
```

Looking good so far, now all we need to run this is an actual instance of `Optimizer`.
We'll use the standard `Aff` for this and for simplicity our new optimization will only look at the `get` operations:

```haskell
extract :: KVStore (Const (S.Set String))
extract = KVStore 
  { get : \key -> Const $ S.singleton key
  , put : \_ _ -> Const $ S.empty
  }

rebuild :: forall e. S.Set String -> KVStore (Aff e) -> Aff e (KVStore (Aff e))
rebuild gs (KVStore interp) = 
  precomputed <#> (\m -> KVStore $ interp
        { get = \key -> case (M.lookup key m) of
            Just a -> pure $ Just a
            Nothing -> interp.get key
        })
  where 
    tupleList :: Aff e (List (Maybe (Tuple String String)))
    tupleList =
          parTraverse (\key -> interp.get key <#> (\m -> m <#> \s -> key /\ s)) (fromFoldable gs)
    precomputed :: Aff e (M.Map String String)
    precomputed = tupleList <#> (M.fromFoldable <<< catMaybes)


instance kvStoreAffOptimizer :: Optimizer KVStore (Aff e) (S.Set String) where
  reqs = { extract , rebuild }
```

Our `Monoid` type is just a simple `Set String` here, as the `extract` function will only extract the `get` operations inside the `Set`.
Then with the `rebuild` we build up our new interpreter.
First we want to precompute all the values of the program.
To do so, we just run all the operations in parallel and put them into a `Map`, while discarding values where the `get` operation returned `Nothing`.
Now when we have that precomputed `Map`, we'll create a new interpreter with it, that will check if the key given to `get` operation is in the precomputed `Map` instead of performing an actual request.
We can then lift the value into a `Aff e (Maybe String)`.
For all the `put` operations, we'll simply run the interpreter.

Now we should have a great optimizer for `KVStore` programs interpreted into an `Aff`.
Let's see how we did by interpreting into a silly implementation that only prints whenever you use one of the operations:

```haskell
testInterpreter :: forall e. KVStore (Aff e)
testInterpreter = KVStore
  { put : \_ value -> do
      liftEff $ unsafeCoerceEff $ log $ "Put something " <> value
      pure unit
  , get : \key -> do
      liftEff $ unsafeCoerceEff $ log $ "Hit network for " <> key
      pure $ Just $ key <> "!"
  }
```

Now let's run our program with this interpreter and the optimizations!

```haskell
launchAff $ monadicProgram testInterpreter
// Hit network for Mice
// Hit network for Cats
// Hit network for Dogs
// Put something: Mice!
// Put something: Cats!
```

And it works, we've now got a principled way to write programs that can then be potentially optimized.



## Conclusion

Designing a way to completely separate the problem description from the actual problem solution is fairly difficult. The tagless final encoding allows us one such fairly simple way.
Using the technique described in this blog post, we should be able to have even more control over the problem solution by inspecting the structure of our program statically.

Another thing we haven't covered here, are programs with multiple algebras, which is quite a bit more complex as you can surely imagine, maybe that will be the topic of a follow up blog post.

The code for this blog post can be found [here](https://github.com/LukaJCB/purescript-sphynx), if people find it useful enough, I'll publish and document it!

What kind of problems and techniques would you like to see with regards to tagless final?
Would love to hear from you in the comments!

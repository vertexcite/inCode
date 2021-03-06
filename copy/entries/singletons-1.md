---
title: "Introduction to Singletons (Part 1)"
categories: Haskell
series: Introduction to Singletons
tags: functional programming, dependent types, haskell, singletons, types
create-time: 2017/08/06 21:07:12
date: 2017/12/22 10:42:07
identifier: singletons-1
slug: introduction-to-singletons-1
---

Real dependent types are coming to Haskell soon!  Until then, we have the
great *[singletons][]* library :)

[singletons]: http://hackage.haskell.org/package/singletons

If you've ever run into dependently typed programming in Haskell, you've
probably encountered mentions of singletons (and the *singletons* library).
This series of articles will be my attempt at giving you the story of the
library, the problems it solves, the power that it gives to you, and how you
can integrate it into your code today![^origin]  (Also, after [my previous
April Fools post][april], people have been asking me for an actual non-joke
singletons post)

[april]: https://blog.jle.im/entry/verified-instances-in-haskell.html

[^origin]: This series will be based on [a talk][lc] I gave over the summer,
and will expand on it.

This post (Part 1) will go over first using the singleton pattern for
*reflection*, then introducing how the singletons library helps us.  Part 2
will discuss using the library for *reification*, to get types that depend on
values at runtime.  Part 3 will go into the basics singleton's
*defunctionalization* system and how we can promote value-level functions to
type-level functions, and Part 4 will delve into deeper applications of
defunctionalization.

[lc]: http://talks.jle.im/lambdaconf-2017/singletons/

I definitely am writing this post with the hope that it will be obsolete in a
year or two.  When dependent types come to Haskell, singletons will be nothing
more than a painful historical note.  But for now, singletons might be the best
way to get your foot into the door and experience the thrill and benefits of
dependently typed programming *today*!

### Prerequisites

These posts will assume no knowledge of dependent types, and, for now, only
basic to intermediate Haskell knowledge (Types, kinds, typeclasses, data types,
functions).  The material in this post *overlaps* with my [dependently typed
neural networks][ann] series, but the concepts are introduced in different
contexts.

[ann]: https://blog.jle.im/entry/practical-dependent-types-in-haskell-1.html

All code is built on *GHC 8.2.2* and with the *[lts-10.0][snapshot]*
snapshot (so, singletons-2.3.1).  However, there are negligible changes in the
GHC type system between GHC 8.0 and 8.2 (the only difference is in the
libraries, more or less), so everything should work on GHC 8.0 as well!

[snapshot]: https://www.stackage.org/lts-10.0

The content in the first section of this post, describing the singleton design
pattern, uses the following extensions:

*   DataKinds
*   GADTs
*   KindSignatures
*   RankNTypes

With some optional "convenience extensions"

*   LambdaCase
*   TypeApplications

And the second section, introducing the *singletons* library itself, uses,
additionally:

*   TemplateHaskell
*   TypeFamilies

These extension will be explained when they are used or become relevant.

The Phantom of the Types
------------------------

*(The code for this pre-singletons section is available [on github][manual-door])*

Let's start with a very common Haskell trick that most learn early in
their Haskelling journey: the [phantom type][].

[phantom type]: https://wiki.haskell.org/Phantom_type

Phantom types in Haskell are a very simple way to add a layer of "type safety"
for your types and DSL's.  It helps you restrict what values functions can take
and encode pre- and post-conditions directly into your types.

For example, in

```haskell
data Foo a = MkFoo
```

The `a` parameter is *phantom*, because nothing of type `a` in the data
type...it just exists as a dummy parameter for the `Foo` type.  We can use
`MkFoo` without ever requiring something of type `a`:

```haskell
ghci> :t MkFoo :: Foo Int
Foo Int
ghci> :t MkFoo :: Foo Bool
Foo Bool
ghci> :t MkFoo :: Foo Either      -- requires -XPolyKinds where 'Foo' is defined
Foo Either
ghci> :t MkFoo :: Foo Monad       -- requires -XConstraintKinds
Foo Monad
```

One use case of phantom type parameters is to prohibit certain functions on
different types of values and let you be more descriptive with how your
functions work together (like in [safe-money][]).  One "hello world" use case
of phantom type parameters is to tag data as "sanitized" or "unsanitized"
(`UserString 'Santitized` type vs. `UserString 'Unsanitized`) or paths as
absolute or relative (`Path 'Absolute` vs. `Path 'Relative`).  For a simple
example, let's check out a simple DSL for a type-safe door:

[safe-money]: https://ren.zone/articles/safe-money

```haskell
!!!singletons/Door.hs "data DoorState " "data Door "
```

A couple things going on here:

1.  Our type we are going to be playing with is a `Door`, which contains a
    single field `doorMaterial` describing, say, the material that the door is
    made out of. (`UnsafeMkDoor "Oak"` would be an oak door)

2.  We're using the `DataKinds` extension to create both the *type* `DoorState`
    as well as the *kind* `DoorState`.

    Normally, `data DoorState = Opened | Closed | Locked` in Haskell defines
    the type `DoorState` and the value constructors `Opened`, `Closed`, and
    `Locked`.

    However, with `DataKinds`, that statement also defines a new *kind*
    `DoorState`, with *type* constructors `'Opened`, `'Closed`, and `'Locked`.
    (note the `'` ticks!)[^ticks]

    ```haskell
    ghci> :k 'Opened
    DoorState
    ghci> :k 'Locked
    DoorState
    ```

[^ticks]: The `'` ticks are technically optional, but I find that it's good
style, at this point in Haskell, to use them whenever you can.  It'll prevent a
lot of confusion, trust me!

3.  We're defining the `Door` type with a *phantom parameter* `s`.  It's a
    phantom type because we don't actually have any *values* of type `s` in our
    data type[^kind] ...the `s` is only just there as a dummy parameter for the type.

    We can use `UnsafeMkDoor` without ever using anything of type `s`.  In
    reality, a real `Door` type would be a bit more complicated (and the direct
    `UnsafeMkDoor` constructor would be hidden).

    ```haskell
    ghci> :t UnsafeMkDoor "Birch" :: Door 'Opened
    Door 'Opened
    ghci> :t UnsafeMkDoor "Iron" :: Door 'Locked
    Door 'Locked
    ```

    We can also use the *TypeApplications* extension to write this in a bit
    more convenient way --

    ```haskell
    ghci> :t UnsafeMkDoor @'Opened "Birch"
    Door 'Opened
    ghci> :t UnsafeMkDoor @'Locked "Iron"
    Door 'Locked
    ```

[^kind]: Indeed, this is not even possible.  There are no values of type
`'SClosed`, `'SOpened`, etc.

Alternatively, we can define `Door` using [*GADT* syntax][gadt] (which requires
the `GADTs` extension)[^gadtnote].

[gadt]: https://en.wikibooks.org/wiki/Haskell/GADT#Syntax

[^gadtnote]: Actually, GADT syntax just requires `-XGADTSyntax`, but `-XGADT`
allows you to actually make GADTs (which we will be doing later), and implies
`-XGADTSyntax`

```haskell
data Door :: DoorState -> Type where
    UnsafeMkDoor :: { doorMaterial :: String } -> Door s
```

This is defining the exact same type in the alternate "GADT syntax" style of
data type declaration -- here, we define types by giving the type of its
constructors, `UnsafeMkDoor :: String -> Door s`.

`Door` here is an **indexed data type**, which is sometimes called a "type
family" in the dependently typed programming world (which is not to be confused
with type families in *GHC Haskell*, `-XTypeFamilies`, which is a language
mechanism that is related but definitely not the same).

### Phantoms in Action

At first, this seems a bit silly.  Why even have the extra type parameter if
you don't ever use it?

Well, right off the bat, we can write functions that expect only a certain type
of `Door`, and return a specific type of `Door`:

```haskell
!!!singletons/Door.hs "closeDoor ::"
```

So, the `closeDoor` function will *only* take a `Door 'Opened` (an opened
door).  And it will return a `Door 'Closed` (a closed door).

```haskell
ghci> let myDoor = UnsafeMkDoor @'Opened "Spruce"
ghci> :t myDoor
Door 'Opened
ghci> :t closeDoor myDoor
Door 'Closed
ghci> let yourDoor = UnsafeMkDoor @'Closed "Acacia"
ghci> :t closeDoor yourDoor
TYPE ERROR!  TYPE ERROR!
```

You can think of this as a nice way of catching *logic errors* at compile-time.
If your door type did not have its status in the type, the `closeDoor` could
have been given a closed or locked door, and you'd have to handle and reject it
at *runtime*.

By adding the state of the door into its type, we can encode our pre-conditions
and post-conditions directly into the type.  And any opportunity to move
runtime errors to compile-time errors should be celebrated with a party!

This would also stop you from doing silly things like closing a door twice in a
row:

```haskell
ghci> :t closeDoor . closeDoor
TYPE ERROR!  TYPE ERROR!  TYPE ERROR!
```

Do you see why?

With a couple of state transitions, we can write compositions that are
type-checked to all be legal:

```haskell
!!!singletons/Door.hs "lockDoor ::" "openDoor ::"
```

```haskell
ghci> :t closeDoor . openDoor
Door 'Closed -> Door 'Closed
ghci> :t lockDoor . closeDoor . openDoor
Door 'Closed -> Door 'Locked
ghci> :t lockDoor . openDoor
TYPE ERROR!  TYPE ERROR!  TYPE ERROR!
```

Because of the type of `lockDoor`, you *cannot* lock an opened door!  Don't
even try!  You'd have to close it first.

```haskell
ghci> let myDoor = UnsafeMkDoor @'Opened "Spruce"
ghci> :t myDoor
Door 'Opened
ghci> :t lockDoor
Door 'Closed -> Door 'Closed
ghci> :t lockDoor myDoor
TYPE ERROR!  TYPE ERROR!  TYPE ERROR!
ghci> :t closeDoor myDoor
Door 'Closed
ghci> :t lockDoor (closeDoor myDoor)
Door 'Locked
```

`lockDoor` expects a `Door 'Closed`, so if you give it a `Door 'Opened`, that's
a static compile-time type error.  But, `closeDoor` takes a `Door
'Opened` and returns a `Door 'Closed` -- so *that* is something that you can
call `lockDoor` with!

### The Phantom Menace

However, in standard Haskell, we quickly run into some practical problems if we
program with phantom types this way.

For example, how could we write a function to get the state of a door?

```haskell
doorStatus :: Door s -> DoorState
doorStatus _ = -- ?
```

(It can be done with an ad-hoc typeclass, but it's not simple, and it's prone
to implementation bugs)

And, perhaps even more important, how can you create a `Door` with a given
state that isn't known until runtime?  If we know the type of our doors at
compile-time, we can just explicitly write `UnsafeMkDoor "Iron" :: Door
'Opened` or `UnsafeMkDoor @'Opened "Iron"`.  But what if we wanted to make a
door based on a `DoorState` *value*?  Something we might not get until runtime?

```haskell
mkDoor :: DoorState -> String -> Door s
mkDoor Opened = -- ?
mkDoor Closed = -- ?
mkDoor Locked = -- ?
```

Ah hah, you say.  That's easy!

```haskell
mkDoor :: DoorState -> String -> Door s
mkDoor Opened = UnsafeMkDoor
mkDoor Closed = UnsafeMkDoor
mkDoor Locked = UnsafeMkDoor
```

Unfortunately, that's not how types work in Haskell.  Remember that for a
polymorphic type `forall s. DoorState -> String -> Door s`, the *caller* picks
the type variable.

```haskell
ghci> :t mkDoor Opened "Acacia" :: Door 'Closed
Door 'Closed
```

Oops!

### The Fundamental Issue in Haskell

We've hit upon a fundamental issue in Haskell's type system: **type erasure**.
In Haskell, types only exist *at compile-time*, for help with type-checking.
They are completely erased at runtime.

This is usually what we want.  It's great for performance, and you can bypass
things like the ad-hoc runtime type checking that you have to deal with in
dynamic languages like python.

But in our case, it makes functions like `doorState` fundamentally impossible.
Or, does it?

The Singleton Pattern
---------------------

A singleton in Haskell is a type (of kind `Type` -- that is, `*`) that has
exactly one inhabitant.  In practice (and when talking about the design
pattern), it refers to a parameterized type that, for each pick of parameter,
gives a type with exactly one inhabitant.  It is written so that pattern
matching on the *constructor* of that value reveals the unique type parameter.

```haskell
!!!singletons/Door.hs "data SingDS ::"
```

Here we're using *GADT syntax* again (but to make an actual GADT).  (Also note
that `Type` is a synonym for the `*` kind, exported from the *Data.Kind*
module)  So, if we use `SOpened`, we will get a `SingDS 'Opened`.  And if we
have a `SingDS 'Opened`, we know that it was constructed using `SOpened`.
Essentially, this gives us three values:

```haskell
SOpened :: SingDS 'Opened
SClosed :: SingDS 'Closed
SLocked :: SingDS 'Locked
```

### The Power of the Pattern Match

The power of singletons is that we can now *pattern match* on types,
essentially.

```haskell
!!!singletons/Door.hs "closeDoor ::"1 "lockDoor ::"1 "lockAnyDoor ::"
```

`lockAnyDoor` is a function that can take a door of any state (a `Door s` of
any `s`) and *lock* it using a composition of `lockDoor` or `closeDoor` as
necessary.

If we have `lockAnyDoor` take a `SingDS s` as its input (and, importantly,
make sure that the `s` in `SingDS s` is the same `s` in the `Door s`), we can
*pattern match* on the `SingDS s` to *reveal* what `s` is, to the type checker.
This is known as a **dependent pattern match**.

*   If `SingDS s`'s pattern match goes down the `SOpened ->` case, then we
    *know* that `s ~ 'Opened`[^eq].  We know that `s` must be `'Opened`,
    because `SOpened :: SingDS 'Opened`, so there really isn't anything else
    the `s` in `SingDS s` could be!

    So, if we know that `s ~ 'Opened`, that means that the `Door s` is `Door
    'Opened`.  So because `door :: Door' Opened`, we have to `closeDoor` it to
    get a `Door' Closed`, and then `lockDoor` it to get a `Door 'Locked`

    We say that `SOpened` is a *runtime witness* to `s` being `'Opened`.

*   Same for the `SClosed ->` branch -- since `SClosed :: SingDS 'Closed`, then
    `s ~ 'Closed`, so our `Door s` must be a `Door 'Closed`.  This allows us to
    simply take our `door :: Door 'Closed` and use `lockDoor` to get a `Door
    'Locked`

*   For the `SLocked ->` branch, `SLocked :: SingDS 'Locked`, so `s ~ 'Locked`, so
    our `Door s` is a `Door 'Locked`.  Our door is "already" locked, so we can
    just use the `door :: Door 'Locked` that we got!

[^eq]: `~` here refers to "type equality", or the constraint that the types on
both sides are equal.  `s ~ 'Opened` can be read as "`s` is `'Opened`".

Essentially, our singletons give us *runtime values* that can be used as
*witnesses* for types and type variables.  These values exist at runtime, so
they "bypass" type erasure.  Types themselves are directly erased, but we can
hold on to them using these runtime tokens when we need them.

Note that we can also write `lockAnyDoor` using the *LambdaCase* extension
syntactic sugar, which I think offers a lot of extra insight:

```haskell
lockAnyDoor :: SingDS s -> (Door s -> Door 'Locked)
lockAnyDoor = \case
    SOpened -> lockDoor . closeDoor  -- in this branch, s is 'Opened
    SClosed -> lockDoor              -- in this branch, s is 'Closed
    SLocked -> id                    -- in this branch, s is 'Locked
```

Here, we can see `lockAnyDoor sng` as a partially applied function that returns
a `Door s -> Door 'Locked`  For any `SingDS s` you give to `lockAnyDoor`,
`lockAnyDoor` returns a "locker function" (`Door s -> Door 'Locked`) that is
custom-made for your `SingDS`:

*   `lockAnyDoor SOpened` will return a `Door 'Opened -> Door 'Locked`.  Here,
    it has to give `lockDoor . closeDoor :: Door 'Opened -> Door 'Locked`.

*   `lockAnyDoor SClosed` will return a `Door 'Closed -> Door 'Locked` --
    namely `lockDoor :: Door 'Closed -> Door 'Locked`.

*   `lockAnyDoor SLocked` will return a `Door 'Locked -> Door 'Locked`, which
    will just be `id :: Door 'Locked -> Door 'Locked`

Note that all of these functions will *only* typecheck under the branch they
fit in.  If we gave `lockDoor` for the `SOpened` branch, or `id` for the
`SClosed` branch, that'll be a compile-time error!

#### Reflection

Writing `doorStatus` is now pretty simple --

```haskell
doorStatus :: SingDS s -> Door s -> DoorState
doorStatus SOpened _ = Opened
doorStatus SClosed _ = Closed
doorStatus SLocked _ = Locked
```

The benefit of the singleton again relies on the fact that the `s` in `SingDS
s` is the same as the `s` in `Door s`, so if the user gives a `SingDS s`, it
*has* to match the `s` in the `Door s` they give.

Since we don't even care about the `door`, we could also just write:

```haskell
!!!singletons/Door.hs "fromSingDS ::"
```

Which we can use to write a nicer `doorStatus`

```haskell
!!!singletons/Door.hs "doorStatus ::"
```

This process -- of turning a type variable (like `s`) into a dynamic runtime
value is known as **reflection**.  We move a value from the *type level* to the
*term level*.

### Recovering Implicit Passing

One downside is that we are required to manually pass in our witness.  Wouldn't
it be nice if we could have it be passed implicitly?  We can actually leverage
typeclasses to give us this ability:

```haskell
!!!singletons/Door.hs "class SingDSI" "instance SingDSI"
```

(Note that *it's impossible* to write our `SingDSI` instances improperly!  GHC
checks to make sure that this is *correct*)

And so now we can do:

```haskell
!!!singletons/Door.hs "lockAnyDoor_ ::" "doorStatus_ ::"
```

Here, type inference will tell GHC that you want `singDS :: SingDS s`, and it
will pull out the proper singleton for the door you want to check!

Now, we can call `lockAnyDoor_` *without passing in* a singleton, explicitly!


```haskell
ghci> let myDoor = UnsafeMkDoor @'Opened "Birch"
ghci> :t lockAnyDoor SOpened myDoor -- our original method!
Door 'Locked
ghci> :t lockAnyDoor singDS myDoor  -- the power of type inference!
Door 'Locked
ghci> :t lockAnyDoor_ myDoor        -- no explicit singleton being passed!
Door 'Locked
```

#### The Same Power

In Haskell, a constraint `SingDSI s =>` is essentially the same as passing in
`SingDS s` explicitly.  Either way, you are passing in a runtime witness that
your function can use.  You can think of `SingDSI s =>` as passing it in
*implicitly*, and `SingDS s ->` as passing it in *explicitly*.

So, it's important to remember that `lockAnyDoor` and `lockAnyDoor_` are the
"same function", with the same power.  They are just written in different
styles -- `lockAnyDoor` is written in explicit style, and `lockAnyDoor_` is
written in implicit style.

#### Going backwards

Going from `SingDSI s =>` to `SingDS s ->` (implicit to explicit) is very easy
-- just use `singDS` to get a `SingDS s` if you have a `SingDSI s` constraint
available.  This is what we did for `lockAnyDoor_` and `doorStatus_`.

Going from `SingDS s ->` to `SingDSI s =>` (explicit to implicit) in Haskell is
actually a little trickier.  The typical way to do this is with a CPS-like
utility function:

```haskell
!!!singletons/Door.hs "withSingDSI ::"
```

`withSingDSI` takes a `SingDS s`, and a value (of type `r`) that requires a
`SingDSI s` instance to be created.  And it creates that value for you!

To use `x`, you must have a `SingDSI s` instance available.  This all works
because in each branch, `s` is now a *specific*, monomorphic, "concrete" `s`,
and GHC knows that such an instance exists for every branch.

*   In the `SOpened` branch, `s ~ 'Opened`.  We explicitly wrote an instance of
    `SingDSI` for `'Opened`, so GHC *knows* that there is a `SingDSI 'Opened`
    instance in existence, allowing you to use/create `x`.
*   In the `SClosed` branch, `s ~ 'Closed`, so GHC knows that there is a
    `SingDSI 'Closed` instance (because we wrote one explicitly!), and gives
    *that* to you -- and so you are allowed to use/create `x`.
*   In the `SLocked` branch, `s ~ 'Locked`, and because we wrote a `SingDSI
    'Locked` explicitly, we *know* that a `SingDSI s` instance is available, so
    we can use/create `x`.

Now, we can run our implicit functions (like `lockAnyDoor_`) by giving them
explicit inputs:

```haskell
!!!singletons/Door.hs "lockAnyDoor__ ::"
```

And the cycle begins anew.

One interesting thing to point out -- note that the type of `withSingDSI` is
very similar to the type of another common combinator:

```haskell
withSingDSI :: SingDS s -> (SingDSI s => r) -> r
flip  ($)   ::        a -> (        a -> r) -> r
```

Which is a bit of a testament to what we said earlier about how a `SingDSI s =>
..)` is the same as `SingDS s -> ..`.  `flip ($)` takes a value and a function
and applies the function to that value.  `withSingDSI` takes a value and
"something like a function" and applies the "something like a function" to that
value.

### Fun with Witnesses

We can write a nice version of `mkDoor` using singletons:

```haskell
!!!singletons/Door.hs "mkDoor ::"
```

We take advantage of the fact that `SingDS s` "locks in" the `s` type variable
for `Door s`.  We can call it now with values of `SingDS`:

```haskell
ghci> :t mkDoor SOpened "Oak"
Door 'Opened
ghci> :t mkDoor SLocked "Spruce"
Door 'Locked
```

Now we can't do something silly like pass in `SLocked` to get a `Door 'Opened`.

The Singletons Library
----------------------

*(The code for this post-singletons section is available [on github][singletons-door])*

Now that we understand some of the benefits of singletons as they relate to
phantom types, we can appreciate what the singletons *library* has to offer: a
fully unified, coherent system for working with singletons of almost *all*
Haskell types!

First, there's Template Haskell for generating our singletons given our type:

```haskell
data DoorState = Opened | Closed | Locked
  deriving (Show, Eq)

genSingletons [''DoorState]

-- or
!!!singletons/DoorSingletons.hs "$(singletons "
```

This generates, for us:

```haskell
-- not the actual code, but essentially what happens
data Sing :: DoorState -> Type where
    SOpened :: Sing 'Opened
    SClosed :: Sing 'Closed
    SLocked :: Sing 'Locked
```

`Sing` is a poly-kinded type constructor (a "data family").  `STrue :: Sing
'True` is the singleton for `'True`, `SJust SOpened :: Sing ('Just 'Opened)` is
the singleton for `'Just 'Opened`, etc.

It also generates us instances for `SingI`, a poly-kinded typeclass:

```haskell
instance SingI 'Opened where
    sing = SOpened
instance SingI 'Closed where
    sing = SClosed
instance SingI 'Locked where
    sing = SLocked
```

Which is basically our `SingDSI` typeclass, except we have instances for
singletons of all kinds! (heh)  There's a `SingI` instance for `'True`, a
`SingI` instance for `10`, a `SingI` instance for `'Just 'Opened`, etc.:

```haskell
ghci> sing :: Sing 'True
STrue
ghci> sing :: Sing ('Just 'Opened)
SJust SOpened
```

We also have `withSingI`, which is equivalent to our `withSingDSI` function
earlier.[^withSingI]

```haskell
withSingI :: Sing s -> (forall r. SingI s => r) -> r
```

[^withSingI]: It is probably worth mentioning that, for practical reasons, the
implementation of *singleton*'s `withSingI` is very different than the
implementation we used for our `withSingDSI`.  However, understanding its
implementation isn't really relevant understanding how to use the library, so
we won't really go to deep into this.

Note that if you have singletons for a kind `k`, you also have instances for
kind `Maybe k`, as well.  And also for `[k]`, even!  The fact that we have a
unified way of working with and manipulating singletons of so many different
types is a major advantage of using the *singletons* library to manage your
singletons instead of writing them yourself.

```haskell
ghci> :t SOpened `SCons` SClosed `SCons` SLocked `SCons` SNil
Sing '[ 'Opened, 'Closed, 'Locked ]
-- 'SCons is the singleton for `:` (cons),
-- and 'SNil is the singleton for `[]` (nil)
```

(Remember that, because of `DataKinds`, `Maybe` is a kind constructor, who has
two type constructors, the type `'Nothing` and the type constructor `'Just :: k
-> Maybe k`)

Singletons for all kinds integrate together seamlessly, and you have mechanisms
to generate them for your own type and roll it all into the system!

### Extra Goodies

In addition to generating singletons for our libraries, it gives us convenient
functions for working with the different "manifestations" of our types.

Recall that `DoorState` has four different things associated with it now:

1.  The *type* `DoorState`, whose value constructors are `Opened`, `Closed`,
    and `Locked`.
2.  The *kind* `DoorState`, whose type constructors are `'Opened`, `'Closed`,
    and `'Locked`
3.  The singletons for `'Opened`, `'Closed`, and `'Locked`:

    ```haskell
    SOpened :: Sing 'Opened
    SClosed :: Sing 'Closed
    SLocked :: Sing 'Locked
    ```

4.  The `SingI` instances for `'Opened`, `'Closed`, and `'Locked'`

Kind of confusing, and in the future, when we have real dependent types, we can
combine all of these manifestations into the *one* thing.  But for now, we do
have to deal with converting between them, and for that, *singletons* generates
for us `fromSing :: Sing (s :: DoorState) -> DoorState`.  `fromSing` takes us
from singletons to term-level values (*reflection*):

```haskell
ghci> fromSing SOpened
Opened
```

It does this by defining a type class (actually, a "kind class"), `SingKind`,
associating each type to the corresponding datakinds-generated kind.  The
`SingKind` instance for `DoorState` links the *type* `DoorState` to the *kind*
`DoorState`.

The library also defines a neat type synonym, `type SDoorState = Sing`, so you
can do `SDoorState 'Opened` instead of `Sing 'Opened`, if you wish.

There are definitely more useful utility functions, but we will investigate
these later on in the series!  For now, you can look at the [documentation][]
for the library to see more interesting utility functions!

[documentation]: http://hackage.haskell.org/package/singletons/docs/Data-Singletons.html

The Singularity
---------------

In this post, at shortcomings in the usage of phantom types, and then saw how
singletons could help us with these.  Then, we looked at how the *singletons*
**library** makes using this pattern extremely easy and smooth to integrate
into your existing code.

You can see all of the "manual singletons" code in this post
[here][manual-door], and then see the code re-implemented using the
*singletons* library [here][singletons-door].

!!![manual-door]:singletons/Door.hs
!!![singletons-door]:singletons/DoorSingletons.hs

However, remember the question that I asked earlier, about creating a `Door`
with a given state that we don't know until runtime?  So far, we are only able
to create `Door` and `SingDS` from types we *know* at compile-time.  There is
no way we have yet to convert a `DoorState` from the value level to the type
level -- so it seems that there is no way to "load" a `Door s` with an `s` that
depends on, say, a file's contents, or user input.  The fundamental issue is
still *type erasure*.

In Part 2, we will delve into how to overcome this and break through from the
barrier of the dynamic "unsafe" runtime to the world of safe, typed, verified
code, and see how the *singletons* library gives us great tools for this.
Afterwards, in Part 3, we will learn to express more complicated relationships
with types and type-level functions using defunctionalization and the tools
from the *singletons* library, and finally break into the world of actual
"type-level programming".

As always, let me know in the comments if you have any questions!  You can also
usually find me idling on the freenode `#haskell` channel, as well, as *jle\`*.
The *singletons* [issue tracker][singgh] is also very active.
Happy haskelling!

[singgh]: https://github.com/goldfirere/singletons/issues

For further reading, check out the [original singletons paper][paper]!  It's
very readable and goes over many of the same techniques in this blog post, just
written with a different perspective and tone :)

[paper]: https://cs.brynmawr.edu/~rae/papers/2012/singletons/paper.pdf

Exercises
---------

Click on the links in the corner of the text boxes for solutions! (or just
check out [the source file][singletons-door])

These should be written in the singletons library style, with `Sing` instead of
`SingDS` and `SingI` instead of `SingDSI`.  Review the [singletons
file][singletons-door] for a comparison, if you are still unfamiliar.

1.  Write a function to unlock a door, but only if the user enters an odd
    number (as a password).

    ```haskell
    !!!singletons/DoorSingletons.hs "unlockDoor ::"1
    ```

    It should return a closed door in `Just` if the caller gives an odd number,
    or `Nothing` otherwise.

2.  Write a function that can open any door, taking a password, in "implicit
    Sing" style:

    ```haskell
    !!!singletons/DoorSingletons.hs "openAnyDoor ::"1
    ```

    This should be written in terms of `unlockDoor` and `openDoor` (see above)
    -- that is, you **should not** use `UnsafeMkDoor` directly for
    `openAnyDoor`.

    If the door is already unlocked or opened, it should ignore the `Int`
    input.

# Unification in Agda

```agda
module UnificationInAgda where
```

## Preface

Agda is a wonderful language and its unification engines are exemplary, practical, improve over time and work predictably well. Unification engines are one notable distiction between Agda and other dependently typed languages (such as Idris 1, Coq, Lean, etc). I'm saying "unification engines", because there are two of them:

- unification used for getting convenient and powerful pattern matching
- unification used for inferring values of implicit arguments

These are two completely distinct machineries. This post covers only the latter, because the former is already elaborated in detail (and implemented) by Jesper Cockx: see his exceptionally well-writted [PhD thesis](https://jesper.sikanda.be/files/thesis-final-digital.pdf) -- it's an entire high-quality book! Section "3.6 Related work" of it shortly describes differences between the requirements for the two unifications engines.

Agda only infers values that are uniquely determined by the current context. I.e. Agda doesn't try to guess: it either fails to infer a value or infers _the_ definite one. Even though this makes the unification algorithm weaker than it could be, it also makes it reliable and predictable. Whenever Agda infers something, you can be sure that this is the thing that you wanted and not just a random guess that would be different if you provided more information to the type checker (but Agda does have a guessing machinery called [Agsy](https://agda.readthedocs.io/en/v2.6.0.1/tools/auto.html) that can be used interactively, so that no guessing needs to be done by the type checker and everything inferred is visible to the user).

We'll look into basics of type inference in Agda and then move to more advanced patterns. But first, some imports:

## Imports

```agda
open import Level renaming (suc to lsuc; zero to lzero)
open import Function.Core using (_∘_; _∋_; case_of_) renaming (_|>_ to _&_)
open import Relation.Binary.PropositionalEquality
open import Data.Unit.Base using (⊤; tt)
open import Data.Bool.Base using (Bool; true; false) renaming (_∨_ to _||_; _∧_ to _&&_)
open import Data.Nat.Base  using (ℕ; zero; suc; _+_; _*_; _∸_)
open import Data.Product using (_×_; Σ; _,_; _,′_)
```

## Basics of type inference

```agda
module BasicsOfTypeInference where
```

Some preliminaries: the type of lists is defined as

```agda
  infixr 5 _∷_

  data List (A : Set) : Set where
    []  : List A
    _∷_ : A -> List A -> List A
```

Agda sees `[]` as having the following type: `∀ {A} -> List A`, however if you ask Agda what the type of `[]` is (by creating a hole in this module via `_ = ?`, putting there `[]` and typing `C-c C-d`. Or you can open the current module via `open BasicsOfTypeInference` and type `C-c C-d []` without introducing a hole), you'll get something like

    List _A_42

(where `42` is some arbitrary number that Agda uses to distinguish between variables that have identical textual names, but are bound in distinct places)

That `_A_42` is a metavariable and Agda expects it to be resolved in the current context. If the context does not provide enough information for resolution to happen, Agda just reports that the metavariable is not resolved, i.e. Agda doesn't accept the code.

In contrast, Haskell is perfectly fine with `[]` and infers its type as `forall a. [a]`.

So Agda and Haskell think of `[]` having the same type

    ∀ {A} -> List A  -- in Agda
    forall a. [a]    -- in Haskell

but Haskell infers this type on the top level unlike Agda which expects `A` to be either resolved or explicitly bound.

You can make Agda infer the same type that Haskell infers by explicitly binding a type variable via a lambda:

```agda
  _ = λ {A} -> [] {A}
```

(`_ = <...>` is an anonymous definition: we ask Agda to type check something, but don't care about giving it a name, because not going to use it later)

This definition is accepted, which means that Agda inferred its type successfully.

Note that

    _ {A} = [] {A}

means the same thing as the previous expression, but doesn't type check. It's just a syntactic limitation: certain things are allowed in patterns but not in lambdas and vice versa.

Agda can infer monomorphic types directly without any hints:

```agda
  _ = true ∷ []
```

Type inference works not only with lambdas binding implicit arguments, but also explicit ones. And types of latter arguments can depend on earlier arguments. E.g.

```agda
  id₁ = λ {A : Set} (x : A) -> x
```

is the regular `id` function, which is spelled as

```haskell
id :: forall a. a -> a
id x = x
```

in Haskell.

Partially or fully applied `id₁` doesn't need a type signature either:

```agda
  _ = id₁
  _ = id₁ {Bool}
  _ = id₁ true
```

You can even interleave implicit and expicit arguments and partial applications (and so full ones as well) will still be inferrable:

```agda
  const = λ {A : Set} (x : A) {B : Set} (y : B) -> x
  _ = const {Bool}
  _ = const true
```

## `let`, `where`, `mutual`

```agda
module LetWhereMutual where
```

In Agda bindings that are not marked with `abstract` are transparent, i.e. writing, say, `let v = e in b` is the same thing as directly substituting `e` for `v` in `b` (`[e/v]b`). For example all of these type check:

```agda
  _ : Bool
  _ = let 𝔹 = Bool
          t : 𝔹
          t = true
      in t

  _ : Bool
  _ = t where
    𝔹 = Bool
    t : 𝔹
    t = true

  𝔹 = Bool
  t : 𝔹
  t = true
  _ : Bool
  _ = t
```

Unlike Haskell Agda does not have let-generalization, i.e. this valid Haskell code:

```haskell
p :: (Bool, Integer)
p = let i x = x in (i True, i 1)
```

has to be written either with an explicit type signature for `i`:

```agda
  _ : Bool × ℕ
  _ = let i : {A : Set} -> A -> A
          i x = x
      in (i true , i 1)
```

or in an equivalent way like

```agda
  _ : Bool × ℕ
  _ = let i = λ {A} (x : A) -> x
      in (i true , i 1)
```

So Agda infers polymorphic types neither on the top level nor locally.

In Haskell types of bindings can be inferred from how those bindings are used later. E.g. the inferred type of a standalone

    one = 1

is `Integer` (see [monomorphism restriction](https://wiki.haskell.org/Monomorphism_restriction), but in

    one = 1
    one' = one :: Word

the inferred type of `one` is `Word` rather than `Integer`.

This is not the case in Agda, e.g. a type for

    i = λ x -> x

is not going to be inferred regardless of how this definition is used later. However if you use `let`, `where` or `mutual` inference is possible:

```agda
  _ = let i = λ x -> x in i true

  _ = i true where i = λ x -> x

  mutual
    i = λ x -> x
    _ = i true
```

In general, definitions in a `mutual` block share the same context, which makes it possible to infer more things than with consecutive standalone definitions. It's occasionally useful to create a bogus `mutual` block when you want the type of a definition to be inferred based on its use.

## An underspecified argument example

```agda
module UnderspecifiedArgument where
```

Another difference between Haskell and Agda is that Agda doesn't allow to leave ambiguous types. Consider a classic example: the `I` combinator can be defined in terms of the `S` and `K` combinators. In Haskell we can express that as

    k :: a -> b -> a
    k x y = x

    s :: (a -> b -> c) -> (a -> b) -> a -> c
    s f g x = f x (g x)

    i :: a -> a
    i = s k k

and [it'll type check](https://ideone.com/mZQM1f). However the Agda's equivalent

```agda
  K : {A B : Set} -> A -> B -> A
  K x y = x

  S : {A B C : Set} -> (A -> B -> C) -> (A -> B) -> A -> C
  S f g x = f x (g x)

  I : ∀ {A} -> A -> A
  I = S K K
```

results in the last `K` being highlighted with yellow (which means that not all metavariables were resolved). To see why, let's reduce `S K K` a bit:

    λ x -> K x (K x)

this computes to `λ x -> x` as expected, but the problem is that in the expression above the `K x` argument is underspecified: a `K` must receive a particular `B`, but we neither explicitly specify a `B`, nor can it be inferred from the context as the entire `K x` argument is thrown away by the outer `K`.

To fix this we can explicitly specify a `B` (any of type `Set` will work, let's pick `ℕ`):

```agda
  I′ : ∀ {A} -> A -> A
  I′ = S K (K {B = ℕ})
```

In general, Agda expects all implicits (and metavariables in general) to be resolved and won't gloss over such details the way Haskell does. Agda is a proof assistant and under the [Curry-Howard correspondence](https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence) each argument to a function represents a certain logical assumption and every such assumption must be fulfilled at the call site either explicitly or in an automated manner.

## InferenceAndPatternMatching

```agda
module InferenceAndMatching where
```

Unrestricted pattern matching generally breaks type inference. Take for instance

```agda
  _ = λ where
          zero    -> true
          (suc _) -> false
```

which is a direct counterpart of Haskell's

    isZero = \case
        0 -> True
        _ -> False


Agda colors the entire snippet in yellow meaning it's unable to resolve the generated metavariables. "What's the problem? The inferred type should be just `ℕ -> Bool`" -- you might think. Such a type works indeed:

```agda
  _ : ℕ -> Bool
  _ = λ where
          zero    -> true
          (suc _) -> false
```

But here's another thing that works:

```agda
  _ : (n : ℕ) -> n & λ where
                         zero -> Bool
                         _    -> Bool
  _ = λ where
          zero    -> true
          (suc _) -> false
```

Recall that we're in a dependently typed language and here the type of the result of a function can depend on the argument of that function. And both the

    ℕ -> Bool

    (n : ℕ) -> n & λ where
                       zero -> Bool
                       _    -> Bool

types are correct for that function. Even though they are "morally" the same, they are not definitionally equal and there's a huge difference between them: the former one doesn't have a dependency and the latter one has.

There is a way to tell Agda that pattern matching is non-dependent: use `case-of`, e.g.

```agda
  _ = λ (n : ℕ) -> case n of λ where
          zero    -> true
          (suc _) -> false
```

type checks. `case_of_` is just a definition in the standard library that at the term level is essentially

    case x of f = f x

and at the type level it restricts the type of `f` to be a non-dependent function.

Annotating `n` with its type, `ℕ`, is mandatory in this case. It seems Agda is not able to conclude that if a value is matched against a pattern, then the value must have the same type as the pattern.

Even this doesn't type check:

```agda
  _ = λ n -> case n of λ where
          zero    -> zero
          (suc n) -> n
```

even though Agda really could figure out that if `zero` is returned from one of the branches, then the type of the result is `ℕ`, and since `n` is returned from the other branch and pattern matching is non-dependent, `n` must have the same type. See [#2834](https://github.com/agda/agda/issues/2834) for why Agda doesn't attempt to be clever here.

There's a funny syntactical way to tell Agda that a function is non-dependent: just do not bind a variable at the type level. This type checks:

```agda
  _ : ℕ -> _
  _ = λ where
          zero    -> true
          (suc _) -> false
```

while this doesn't:

```agda
  _ : (n : ℕ) -> _
  _ = λ where
          zero    -> true
          (suc _) -> false
```

In the latter case Agda treats `_` as being porentially dependent on `n`, since `n` is explicitly bound. And in the former case there can't be any dependency on a non-existing variable.

## InferenceAndConstructors

```agda
module InferenceAndConstructors where
```

Since tuples are dependent, this

```agda
  _ = (1 , 2)
```

results in unresolved metas as all of these

    ℕ × ℕ

    Σ ℕ λ where
            zero -> ℕ
            _    -> ℕ

    Σ ℕ λ where
            1 -> ℕ
            _ -> Bool

are valid types for this expression, which is similar to what we've considered in the previous section, except here not all of the types are "morally the same": the last one is very different to the first two.

As in the case of functions you can use a non-dependent alternative

```agda
  _ = (1 ,′ 2)
```

(`_,′_` is a non-dependent version of `_,_`)

to tell Agda not to worry about potential dependencies.

## Unification intro

```agda
module UnificationIntro where
```

The following definitions type check:

```agda
  _ = (λ x -> x) 1
  _ = (λ x -> 2) 1
```

reassuring that Agda's type checker is not based on some simple bidirectional typing rules (if you're not familier with those, see [Bidirectional Typing Rules: A Tutorial](http://www.davidchristiansen.dk/tutorials/bidirectional.pdf), but the type checker does have a bidirectional interface ([`inferExpr`](https://hackage.haskell.org/package/Agda-2.6.1/docs/Agda-TheTypeChecker.html#v:inferExpr) & [`checkExpr`](https://hackage.haskell.org/package/Agda-2.6.1/docs/Agda-TheTypeChecker.html#v:checkExpr)) where type inference is defined in terms of type checking for the most part:

    -- | Infer the type of an expression. Implemented by checking against a meta variable. <...>
    inferExpr :: A.Expr -> TCM (Term, Type)

which means that any definition of the following form:

    name = term

can be equally written as

    name : _
    name = term

since Agda elaborates `_` to a fresh metavariable and then type checks `term` againt it, which amounts to unifying the inferred type of `term` with the meta. If the inferred type doesn't contain metas itself, then the meta standing for `_` is resolved as that type and the definition is accepted. So type inference is just a particular form of unification.

You can put `_` basically anywhere and let Agda infer what term/type it stands for. For example:

```agda
  id₂ : {A : Set} -> A -> _
  id₂ x = x
```

Here Agda binds the `x` variable and records that it has type `A` and when the `x` variable is returned as a result, Agda unifies the expected type `_` with the actual type of `x`, which is `A`. Thus the definition above elaborates to

```agda
  id₂′ : {A : Set} -> A -> A
  id₂′ x = x
```

This definition:

```agda
  id₃ : {A : Set} -> _ -> A
  id₃ x = x
```

elaborates to the same result in a similar fashion, except now Agda first records that the type of `x` is a meta and when `x` is returned as a result, Agda unifies that meta with the expected type, i.e. `A`, and so the meta gets resolved as `A`.

An `id` function that receives an explicit type:

```agda
  id₄ : (A : Set) -> _ -> A
  id₄ A x = x
```

can be called as

```agda
  _ = id₄ _ true
```

and the `_` will be inferred as `Bool`.

It's also possible to explicitly specify an implicit type by `_`, which is essentially a no-op:

```agda
  id₅ : {A : Set} -> A -> A
  id₅ x = x

  _ = id₅ {_} true
```

## Implicit arguments

```agda
module ImplicitArgumens where
```

As we've just seen implicit arguments and metavariable are closely related. Agda's internal theory has metas in it, so inference of implicit arguments amounts to turning an implicit into a metavariable and resolving it later. The complicated part however is that it's not always obvious where to bound an implicit.

For example, it may come as a surprise, but

    _ : ∀ {A : Set} -> A -> A
    _ = λ {A : Set} x -> x

gives a type error. This is because Agda greedily binds implicits, so the `A` at the term level gets automatically bound on the lhs (left-hand side, i.e. before `=`), which gives you

    _ : ∀ {A : Set} -> A -> A
    _ {_} = <your_code_goes_here>

where `{_}` stands for `{A}`. So you can't bind `A` by a lambda, because it's already silently bound for you. Although it's impossible to reference that type variable unless you explicitly name it as in

```agda
  id : {A : Set} -> A -> A
  id {A} = λ (x : A) -> x
```

There is a notorious bug that has been in Agda for ages (even since its creation probably?) called The Hidden Lambda Bug:

- tracked in [this issue](https://github.com/agda/agda/issues/1079)
- discussed in detail in [this issue](https://github.com/agda/agda/issues/2099)
- there even an entire [MSc thesis](http://www2.tcs.ifi.lmu.de/~abel/MScThesisJohanssonLloyd.pdf) about it
- and a [possible solution](https://github.com/AndrasKovacs/elaboration-zoo/tree/master/experimental/poly-instantiation)

So while the turn-implicits-into-metas approach works well, it has its edge cases. In practice, it's not a big deal to insert an implicit lambda to circumvent the bug, but it's not always clear that Agda throws a type error because of this bug and not due to something else (e.g. I was completely lost in [this case](https://github.com/agda/agda/issues/1095)). So beware.

## Inferring implicits

```agda
module InferringImplicits where
```

As we've seen previously the following code type checks fine:

```agda
  id : {A : Set} -> A -> A
  id x = x

  _ = id true
```

Here `A` is bound implicitly in `id`, but Agda is able to infer that in this case `A` should be instantiated to `Bool` and so Agda elaborates the expression to `id {Bool} true`.

This is something that Haskell would infer as well. The programmer would hate to explicitly write out the type of every single argument, so programming languages often allow not to specify types when they can be inferred from the context. Agda is quite unique here however, because it can infer a lot more than other languages (even similar dependently typed ones) due to bespoke machineries handling various common patterns. But let's start with basics.

## Arguments of data types

```agda
module ArgumentsOfDataTypes where
  open BasicsOfTypeInference
```

Agda can infer parameters/indices of a data type from a value of that data type. For example if you have a function

```agda
  listId : ∀ {A} -> List A -> List A
  listId xs = xs
```

then the implicit `A` can be inferred from a list:

```agda
  _ = listId (1 ∷ 2 ∷ [])
```

Unless, of course, `A` can't be determined from the list alone. E.g. if we pass an empty list to `f`, Agda will mark `listId` with yellow and display an unresolved metavariable `_A`:

```agda
  _ = listId []
```

Another example of this situation is when the list is not empty, but the type of its elements can't be inferred, e.g.

```agda
  _ = listId ((λ x -> x) ∷ [])
```

Here the type of `x` can be essentially anything (`ℕ`, `List Bool`, `⊤ × Bool -> ℕ`, etc), so Agda asks to provide missing details. Which we can do either by supplying a value for the implicit argument explicitly

```agda
  _ = listId {ℕ -> ℕ} ((λ x -> x) ∷ [])
```

or by annotating `x` with a type

```agda
  _ = listId ((λ (x : ℕ) -> x) ∷ [])
```

or just by providing a type signature

```agda
  _ : List (ℕ -> ℕ)
  _ = listId ((λ x -> x) ∷ [])
```

All these definitions are equivalent.

So "`A` is inferrable from a `List A`" doesn't mean that you can pass any list in and magically synthesize the type of its elements -- only that if that type is already known at the call site, then you don't need to explicitly specify it to apply `listId` to the list as it'll be inferred for you. "Already known at the call site" doesn't mean that the type of elements needs to be inferrable -- sometimes it can be derived from the context, for example:

```agda
  _ = suc ∷ listId ((λ x -> x) ∷ [])
```

The implicit `A` gets inferred here: since all elements of a list have the same type, the type of `λ x -> x` must be the same as the type of `suc`, which is known to be `ℕ -> ℕ`, hence the type of `λ x -> x` is also `ℕ -> ℕ`.

### Comparison to Haskell

In Haskell it's also the case that `a` is inferrable form a `[a]`: when the programmer writes

    sort :: Ord a => [a] -> [a]

Haskell is always able to infer `a` from the given list (provided `a` is known at the call site: `sort []` is as meaningless in Haskell as it is in Agda) and thus figure out what the appropriate `Ord a` instance is. However, another difference between Haskell and Agda is that whenever Haskell sees that some implicit variables (i.e. those bound by `forall <list_of_vars> .`) can't be inferred in the general case, Haskell, unlike Agda, will complain. E.g. consider the following piece of code:

    {-# LANGUAGE MultiParamTypeClasses, FlexibleInstances, TypeFamilies #-}

    class C a b where
      f :: a -> Int

    instance b ~ () => C Bool b where
      f _ = 0

    main = print $ f True

Even though at the call site (`f True`) `b` is determined via the `b ~ ()` constraint of the `C Bool b` instance and so there is no ambiguity, Haskell still complains about the definition of the `C` class itself:

    • Could not deduce (C a b0)
      from the context: C a b
        bound by the type signature for:
                   f :: forall a b. C a b => a -> Int
        at prog.hs:6:3-15
      The type variable ‘b0’ is ambiguous

The type of the `f` function mentions the `b` variable in the `C a b` constraint, but that variable is not mentioned anywhere else and hence can't be inferred in the general case, so Haskell complains, because by default it wants all type variables to be inferrable upfront regardless of whether at the call site it would be possible to infer a variable in some cases or not. We can override the default behavior by enabling the `AllowAmbiguousTypes` extension, which makes the code type check without any additional changes.

Agda's unification capabilities are well above Haskell's ones, so Agda doesn't attempt to predict what can and can't be inferred and allows us to make anything implicit deferring resolution problems to the call site (i.e. it's like having `AllowAmbiguousTypes` globally enabled in Haskell). In fact, you can make implicit even such things that are pretty much guaranteed to never have any chance of being inferred, for example

```agda
  const-zeroᵢ : {_ : ℕ} -> ℕ
  const-zeroᵢ = zero
```

## Under the hood

```agda
module UnderTheHood where
  open BasicsOfTypeInference
```

### Example 1

Returning to our `listId` example, when the user writes

```agda
  listId : ∀ {A} -> List A -> List A
  listId xs = xs

  _ = listId (1 ∷ 2 ∷ [])
```

here is what happens under the hood:

1. the implicit `A` gets instantiated as a metavariable `_A`
2. the type of the instantiated `listId` becomes `List _A -> List _A`
3. `List _A` (what the instantiated `listId` expects as an argument) gets unified with `List ℕ` (the type of the actual argument). We'll write this as `List _A =?= List ℕ`
4. From unification's point of view type constructors are injective, hence `List _A =?= List ℕ` simplifies to `_A =?= ℕ`, which immediately gets solved as `_A := ℕ`

And this is how Agda figures out that `A` gets instantiated by `ℕ`.

### Example 2

Similarly, when the user writes

```agda
  _ = suc ∷ listId ((λ x -> x) ∷ [])
```

1. the implicit `A` gets instantiated as a metavariable `_A`
2. the type of the instantiated `listId` becomes `List _A -> List _A`
3. `List _A` (what the instantiated `listId` expects as an argument) gets unified with `List (_B -> _B)` (the type of the actual argument). `_B` is another metavariable. Recall that we don't know the type of `x` and hence we simply make it a meta
4. `List _A` (this time the type of the result that `listId` returns) also gets unified with the expected type, which is `ℕ -> ℕ`, because `suc` prepended to the result of the `listId` application is of this type
5. we get the following [unification problem](https://en.wikipedia.org/wiki/Unification_(computer_science)#Unification_problem,_solution_set) consisting of two equations:

       List _A =?= List (_B -> _B)
       List _A =?= List (ℕ -> ℕ)

6. as before we can simplify the equations by stripping `List`s from both the sides of each of them:

       _A =?= _B -> _B
       _A =?= ℕ -> ℕ

7. the second equation gives us `A := ℕ -> ℕ` and it only remains to solve

       ℕ -> ℕ =?= _B -> _B

8. which is easy: `_B := ℕ`. The full solution of the unification problem is

       _B := ℕ
       _A := ℕ -> ℕ

### Example 3

When the user writes

```agda
  _ = λ xs -> suc ∷ listId xs
```

1. the yet-unknown type of `xs` elaborates to a metavariable, say, `_LA`
2. the implicit `A` of `listId` elaborates to a metavariable `_A`
3. `List _A` (what the instantiated `listId` expects as an argument) gets unified with `_LA` (the type of the actual argument)
4. `List _A` (this time the type of the result that `listId` returns) also gets unified with the expected type, which is `ℕ -> ℕ`, because `suc` prepended to the result of the `listId` application is of this type
5. we get the following unification problem consisting of two equations:

       List _A =?= _LA
       List _A =?= List (ℕ -> ℕ)

6. `_A` gets solved as `_A := ℕ -> ℕ`
7. and `_LA` gets solved as `_LA := List (ℕ -> ℕ)`
8. so the final solution is

       _A := ℕ -> ℕ
       _LA := List (ℕ -> ℕ)

But note that we could first resolve `_LA` as `List _A`, then resolve `_A` and then instantiate it in `List _A` (what `_LA` was resolved to), which would give us the same final solution.

In general, there are many possible routes that one can take when solving a unification problem, but some of them are less straightforward (and thus less efficient) than others. Such details are beyond the scope of this document, here we are only interested in unification problems that get generated during type checking and solutions to them. Arriving at those solutions is a pretty technical (and incredibly convoluted) thing.

## Nicer notation

In the previous section we were stripping `List` from both the sides of an equation. We were able to do this, because from the unification's point of view type constructors are injective (this has nothing to do with the [`--injective-type-constructors`](https://github.com/agda/agda/blob/10d704839742c332dc85f1298b80068ce4db6693/test/Succeed/InjectiveTypeConstructors.agda) pragma that [makes Agda anti-classical](https://lists.chalmers.se/pipermail/agda/2010/001526.html)). I.e. `List A` uniquely determines `A`.

We'll denote "`X` uniquely determines `Y`" (the notation comes from the [bidirectional typechecking](https://ncatlab.org/nlab/show/bidirectional+typechecking) discipline) as `X ⇉ Y`. So `List A ⇉ A`.

An explicitly provided argument (i.e. `x` in either `f x` or `f {x}`) uniquely determines the type of that argument. We'll denote that as `(x : A) ⇉ A`.

We'll denote "`X` does not uniquely determine `Y`" as `X !⇉ Y`.

We'll also abbreviate

    X ⇉ Y₁
    X ⇉ Y₂
    ...
    X ⇉ yₙ

as

    X ⇉ Y₁ , Y₂ ... Yₙ

(and similarly for `!⇉`).

We'll denote "`X` can be determined in the current context" by

    ⇉ X

Finally, we'll have derivitation trees like

    X        Y
    →→→→→→→→→→
      Z₁ , Z₂        A
      →→→→→→→→→→→→→→→→
             B

which reads as "if `X` and `Y` are determined in the current context, then it's possible to determine `Z₁` and `Z₂`, having which together with `A` determined in the current context, is enough to determine `B`".

## Type functions

```agda
module TypeFunctions where
  open BasicsOfTypeInference
```

Analogously to `listId` we can define `fId` that works for any `F : Set -> Set`, including `List`:

```agda
  fId : ∀ {F : Set -> Set} {A} -> F A -> F A
  fId a = a
```

Unfortunately applying `fId` to a list without explicitly instantiating `F` as `List`

```agda
  _ = fId (1 ∷ 2 ∷ [])
```

results in both `F` and `A` not being resolved. This might be surprising, but there is a good reason for this behavior: there are multiple ways `F` and `A` can be instantiated, so Agda doesn't attempt to pick a random one. Here's the solution that the user would probably have had in their mind:

    _F := List
    _A := ℕ

but this one is also valid:

    _F := λ _ -> List ℕ
    _A := Bool

i.e. `F` ignores `A` and just returns `List ℕ`:

```agda
  _ = fId {λ _ -> List ℕ} {Bool} (1 ∷ 2 ∷ [])
```

Even if you specify `A = ℕ`, `F` still can be either `List` or `λ _ -> List ℕ`, so you have to specify `F` (and then the problem reduces to the one that we considered earlier, hence there is no need to also specify `A`):

```agda
  _ = fId {List} (1 ∷ 2 ∷ [])
```

Therefore, `F A` (where `F` is a type variable) uniquely determines neither `F` nor `A`, i.e. `F A !⇉ F , A`.

### Comparison to Haskell

A type application of a variable is injective in Haskell. I.e. unification of `f a` and `g b` (where `f` and `g` are type variables) forces unification of `a` and `b`, as well as unification of `f` and `g`. I.e. not only does `f a ⇉ a` hold for arbitrary type variable `f`, but also `f a ⇉ f`. This makes it possible to define functions like

    fmap :: Functor f => (a -> b) -> f a -> f b

and use them without compulsively specifying `f` at the call site each time.

Haskell is able to infer `f`, because no analogue of Agda's `λ _ -> List ℕ` is possible in Haskell as its surface language doesn't have type lambdas. You can't pass a type family as `f` either. Therefore there exists only one solution for "unify `f a` with `List Int`" in Haskell and it's the expected one:

    f := List
    a := Int

For a type family `F` we have `F a !⇉ a` (just like in Agda), unless `F` is an [injective type family](https://gitlab.haskell.org/ghc/ghc/wikis/injective-type-families).

## Data constructors

```agda
module DataConstructors where
```

Data constructors are injective from the unification point of view and from the theoretical point of view as well (unlike type constructors). E.g. consider the type of vectors (a vector is a list whose length is statically known):

```agda
  infixr 5 _∷ᵥ_
  data Vec (A : Set) : ℕ -> Set where
    []ᵥ  : Vec A 0
    _∷ᵥ_ : ∀ {n} -> A -> Vec A n -> Vec A (suc n)
```

The `head` function is defined like that over `Vec`:

```agda
  headᵥ : ∀ {A n} -> Vec A (suc n) -> A
  headᵥ (x ∷ᵥ _) = x
```

I.e. we require an input vector to have at least one element and return that first element.

`n` can be left implicit, because `suc n ⇉ n`. In general, for a constructor `C` the following holds:

    C x₁ x₂ ... xₙ ⇉ x₁ , x₂ ... xₙ

A simple test:

```agda
  _ = headᵥ (0 ∷ᵥ []ᵥ)
```

Here we pass a one-element vector to `headᵥ` and Agda succesfully infers the implicit `n` of `headᵥ` to be `0` (i.e. no elements in the vector apart from the first one).

During unification the implicit `n` gets instantiated to a metavariable, say, `_n` and `suc _n` (the expected length of the vector) gets unified with `suc zero` (i.e. 1, the actual length of the vector), which amounts to unifying `_n` with `zero`, which immediately results in `n := zero`.

Instead of having a constant vector, we can have a vector of an unspecified length and infer that length by providing `n` to `headᵥ` explicitly, as in

```agda
  _ = λ {n} xs -> headᵥ {ℕ} {n} xs
```

The type of that definition is `∀ {n} -> Vec ℕ (suc n) -> ℕ`.

We started by binding two variables without specifying their types, but those got inferred from how arguments are used by `headᵥ`.

Note that `_⇉_` is transitive, i.e. if `X ⇉ Y` and `Y ⇉ Z`, then `X ⇉ Z`. For example, since `Vec A n ⇉ n` (due to `Vec` being a type constructor) and `suc n ⇉ n` (due to `suc` being a data constructor), we have `Vec A (suc n) ⇉ n` (by transitivity of `_⇉_`).

## Reduction

If `X` reduces to `Y` (we'll denote that as `X ~> Y`) and `Y ⇉ Z`, then `X ⇉ Z`.

E.g. if we define an alternative version of `headᵥ` that uses `1 +_` instead of `suc`:

```agda
  headᵥ⁺ : ∀ {A n} -> Vec A (1 + n) -> A
  headᵥ⁺ (x ∷ᵥ _) = x
```

the `n` will still be inferrable:

```agda
  _ = headᵥ⁺ (0 ∷ᵥ []ᵥ)
```

This is because `1 + n` reduces to `suc n`, so the two definitions are equivalent.

Note that Agda looks under lambdas when reducing an expression, so for example `λ n -> 1 + n` and `λ n -> suc n` are two definitionally equal terms:

```agda
  _ : (λ n -> 1 + n) ≡ (λ n -> suc n)
  _ = refl
```

Note also that Agda does not look under pattern matching lambdas, so for example these two functions

    λ{ zero -> zero; (suc n) -> 1 + n }
    λ{ zero -> zero; (suc n) -> suc n }

are not considered definitionally equal. In fact, even

    _ : _≡_ {A = ℕ -> ℕ}
        (λ{ zero -> zero; (suc n) -> suc n })
        (λ{ zero -> zero; (suc n) -> suc n })
    _ = refl

is an error despite the two functions being syntactically equal. Here's the funny error:

    (λ { zero → zero ; (suc n) → suc n }) x !=
    (λ { zero → zero ; (suc n) → suc n }) x of type ℕ
    when checking that the expression refl has type
    (λ { zero → zero ; (suc n) → suc n }) ≡
    (λ { zero → zero ; (suc n) → suc n })

## Pattern matching

Generally speaking, pattern matching breaks inference. We'll consider various cases, but to start with the simplest ones we need to introduce a slightly weird definition of the plus operator:

```agda
module WeirdPlus where
  open DataConstructors

  _+′_ : ℕ -> ℕ -> ℕ
  zero  +′ m = m
  suc n +′ m = n + suc m
```

because the usual one

    _+_ : ℕ -> ℕ -> ℕ
    zero  + m = m
    suc n + m = suc (n + m)

is subject to certain unification heuristics, which the weird one doesn't trigger.

We'll be using the following function for demonstration purposes:

```agda
  idᵥ⁺ : ∀ {A n m} -> Vec A (n +′ m) -> Vec A (n +′ m)
  idᵥ⁺ xs = xs
```

### A constant argument

`idᵥ⁺` applied to a constant vector

```agda
  _ = idᵥ⁺ (1 ∷ᵥ 2 ∷ᵥ []ᵥ)
```

gives us yellow, because Agda turns the implicit `n` and `m` into metavariables `_n` and `_m` and tries to unify the expected length of a vector (`_n +′ _m`) with the actual one (`2`) and there are multiple solutions to this problem, e.g.

```agda
  _ = idᵥ⁺ {n = 1} {m = 1} (1 ∷ᵥ 2 ∷ᵥ []ᵥ)
  _ = idᵥ⁺ {n = 2} {m = 0} (1 ∷ᵥ 2 ∷ᵥ []ᵥ)
```

Howewer as per the previous the section, we do not really need to specify `m`, since `_+′_` is defined by recursion on `n` and hence for it to reduce it suffices to specify only `n`:

```agda
  _ = idᵥ⁺ {n = 1} (1 ∷ᵥ 2 ∷ᵥ []ᵥ)
```

since with `n` specified this way the `_n` metavariable gets resolved as `_n := 1` and the expected length of an argument, `_n +′ _m`, becomes `suc m`, which Agda knows how to unify with `2` (the length of the actual argument).

Specifying `m` instead of `n` won't work though:

```agda
  _ = idᵥ⁺ {m = 1} (1 ∷ᵥ 2 ∷ᵥ []ᵥ)
```

Agda can't resolve `_n`. This is because `_+′_` is defined by pattern matching on its first variable, so `1 +′ m` reduces to `suc m`, but `n +′ 1` is stuck and doesn't reduce to anything when `n` is a variable/metavariable/any stuck term. So even though there's a single solution to the

    n +′ 1 =?= 2

unification problem, Agda is not able to come up with it, because this would require arbitrary search in the general case and Agda's unification machinery carefully avoids any such strategies.

### A non-constant argument

`idᵥ⁺` applied to a non-constant vector has essentially the same inference properties.

Without specializing the implicit arguments we get yellow:

```agda
  _ = λ n m (xs : Vec ℕ (n +′ m)) -> idᵥ⁺ xs
```

Specializing `m` doesn't help, still yellow:

```agda
  _ = λ n m (xs : Vec ℕ (n +′ m)) -> idᵥ⁺ {m = m} xs
```

And specializing `n` (with or without `m`) allows Agda to resolve all the metas:

```agda
  _ = λ n m (xs : Vec ℕ (n +′ m)) -> idᵥ⁺ {n = n} xs
  _ = λ n m (xs : Vec ℕ (n +′ m)) -> idᵥ⁺ {n = n} {m = m} xs
```

### Examples

So we have the following rule of thumb: whenever the type of function `h` mentions function `f` at the type level, every argument that gets pattern matched on in `f` (including any internal function calls) should be made explicit in `h` and every other argument can be left implicit (there are a few exceptions to this rule, which we'll consider below, but it applies in most cases).

#### Example 1

`idᵥ⁺` mentions `_+′_` in its type:

    idᵥ⁺ : ∀ {A n m} -> Vec A (n +′ m) -> Vec A (n +′ m)

and `_+′_` pattern matches on `n`, hence Agda won't be able to infer `n`, i.e. the user will have to provide and so it should be made explicit:

    idᵥ⁺ : ∀ {A m} n -> Vec A (n +′ m) -> Vec A (n +′ m)

At the same time `_+′_` doesn't match on its second argument, `m`, hence we leave it implicit.

#### Example 2

A function mentioning `_∸_`

    _-_ : ℕ -> ℕ -> ℕ
    n     - zero  = n
    zero  - suc m = zero
    suc n - suc m = n - m

at type level has to receive both the arguments that get fed to `_∸_` explicitly as `_∸_` matches on both of them:

```agda
  idᵥ⁻ : ∀ {A} n m -> Vec A (n ∸ m) -> Vec A (n ∸ m)
  idᵥ⁻ n m xs = xs
```

and none of

```agda
  _ = idᵥ⁻ 2 _ (1 ∷ᵥ []ᵥ)  -- `m` can't be inferred
  _ = idᵥ⁻ _ 1 (1 ∷ᵥ []ᵥ)  -- `n` can't be inferred
```

is accepted unlike

```agda
  _ = idᵥ⁻ 2 1 (1 ∷ᵥ []ᵥ)
```

#### Example 3

A function mentioning `_*_`

    _*_ : ℕ -> ℕ -> ℕ
    zero  * m = zero
    suc n * m = m + n * m

at the type level has to receive both the arguments that get fed to `_*_` explicitly, even though `_*_` doesn't directly match on `m`. This is because in the second clause `_*_` expands to `_+_`, which does match on `m`. So it's

```agda
  idᵥ* : ∀ {A} n m -> Vec A (n * m) -> Vec A (n * m)
  idᵥ* n m xs = xs
```

and none of

```agda
  _ = idᵥ* 2 _ (1 ∷ᵥ 2 ∷ᵥ []ᵥ)  -- `m` can't be inferred
  _ = idᵥ* _ 1 (1 ∷ᵥ 2 ∷ᵥ []ᵥ)  -- `n` can't be inferred
```

type check, unlike

```agda
  _ = idᵥ* 2 1 (1 ∷ᵥ 2 ∷ᵥ []ᵥ)
```

#### Example 4

With this definition:

```agda
  ignore2 : ∀ n m -> Vec ℕ (n +′ m) -> Vec ℕ (m +′ n) -> ℕ
  ignore2 _ _ _ _ = 0
```

it suffices to explicitly provide either `n` or `m`:

```agda
  _ = ignore2 2 _ (1 ∷ᵥ 2 ∷ᵥ 3 ∷ᵥ []ᵥ) (4 ∷ᵥ 5 ∷ᵥ 6 ∷ᵥ []ᵥ)
  _ = ignore2 _ 1 (1 ∷ᵥ 2 ∷ᵥ 3 ∷ᵥ []ᵥ) (4 ∷ᵥ 5 ∷ᵥ 6 ∷ᵥ []ᵥ)
```

This is because with explicitly provided `n` Agda can determine `m` from `n +′ m` and with explicitly provided `m` Agda can determine `n` from `m +′ n`.

#### Example 5

In the following definition we have multiple mentions of `_+′_` at the type level:

```agda
  ignore2p : ∀ {m p} n -> Vec ℕ (n +′ (m +′ p)) -> Vec ℕ (n +′ m) -> ℕ
  ignore2p _ _ _ = 0
```

and three variables used as arguments to `_+′_`, yet only the `n` variable needs to be bound explicitly. This is due to the fact that it's enough to know `n` to determine what `m` is (from `Vec ℕ (n +′ m)`) and then knowing both `n` and `m` is enough to determine what `p` is (from `Vec ℕ (n +′ (m +′ p))`). Which can be written as

         n
         →
    n    m
    →→→→→→
       p

Note that the order of the `Vec` arguments doesn't matter, Agda will postpone resolving a metavariable until there is enough info to resolve it.

A test:

```agda
  _ = ignore2p 1 (3 ∷ᵥ 4 ∷ᵥ 5 ∷ᵥ 6 ∷ᵥ []ᵥ) (1 ∷ᵥ 2 ∷ᵥ []ᵥ)
```

#### Example 6

A very similar example:

```agda
  ignore1p : ∀ {m p} n -> Vec (Vec ℕ (n +′ m)) (n +′ (m +′ p)) -> ℕ
  ignore1p _ _ = 0
```

Just like in the previous case it's enough to provide only `n` explicitly as the same

         n
         →
    n    m
    →→→→→→
       p

logic applies. Test:

```agda
  _ = ignore1p 1 ((1 ∷ᵥ 2 ∷ᵥ []ᵥ) ∷ᵥ (3 ∷ᵥ 4 ∷ᵥ []ᵥ) ∷ᵥ (5 ∷ᵥ 6 ∷ᵥ []ᵥ) ∷ᵥ []ᵥ)
```

#### Large elimination

```agda
module LargeElimination where
  open BasicsOfTypeInference
```

So far we've been talking about functions that pattern match on terms and return terms, but in Agda we can also pattern match on terms and return types. Consider

```agda
  ListOfBoolOrℕ : Bool -> Set
  ListOfBoolOrℕ false = List Bool
  ListOfBoolOrℕ true  = List ℕ
```

This function matches on a `Bool` argument and returns *the type* of lists with the type of elements depending on the `Bool` argument.

Having an identity function over a `ListOfBoolOrℕ b`

```agda
  idListOfBoolOrℕ : {b : Bool} -> ListOfBoolOrℕ b -> ListOfBoolOrℕ b
  idListOfBoolOrℕ xs = xs
```

we can show that the implicit `b` can't be inferred, as this:

```agda
  _ = idListOfBoolOrℕ (1 ∷ 2 ∷ 3 ∷ [])
```

results in unresolved metas, while this:

```agda
  _ = idListOfBoolOrℕ {b = true} (1 ∷ 2 ∷ 3 ∷ [])
```

is accepted by the type checker.

The reason for this behavior is the same as with all the previous examples: pattern matching blocks inference and `ListOfBoolOrℕ` is a pattern matching function.

### Generalization

```agda
module Generalization where
```

In general: given a function `f` that receives `n` arguments on which there's pattern matching anywhere in the definition of `f` (including calls to other functions in the body of `f`) and `m` arguments on which there is no pattern matching, we have the following rule (for simplicity of presentation we place `pᵢ` before `xⱼ`, but the same rule works when they're interleaved)

    p₁    ...    pₙ        (f p₁ ... pₙ x₁ ... xₘ)
    →→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→
                  x₁    ...    xₘ

i.e. if every `pᵢ` can be inferred from the current context, then every `xⱼ` can be inferred from `f p₁ ... pₙ x₁ ... xₘ`.

There is an important exception from this rule and this is what comes next.

### [Constructor-headed functions](https://wiki.portal.chalmers.se/agda/pmwiki.php?n=ReferenceManual.FindingTheValuesOfImplicitArguments)

```agda
module ConstructorHeadedFunctions where
  open BasicsOfTypeInference
  open DataConstructors
```

Consider a definition of `ListOfBoolOrℕ` that is slightly different from the previous one, but is isomorphic to it:

```agda
  BoolOrℕ : Bool -> Set
  BoolOrℕ false = Bool
  BoolOrℕ true  = ℕ

  ListOfBoolOrℕ′ : Bool -> Set
  ListOfBoolOrℕ′ b = List (BoolOrℕ b)
```

Here `ListOfBoolOrℕ′` does not do any pattern matching itself and instead immediately returns `List (BoolOrℕ b)` with pattern matching performed in `BoolOrℕ b`. There's still pattern matching on `b` and the fact that it's inside another function call in the body of `ListOfBoolOrℕ′` does not change anything as we've discussed previously. Yet `id` defined over such lists:

```agda
  idListOfBoolOrℕ′ : {b : Bool} -> ListOfBoolOrℕ′ b -> ListOfBoolOrℕ′ b
  idListOfBoolOrℕ′ xs = xs
```

does not require the user to provide `b` explicitly, i.e. the following type checks just fine:

```agda
  _ = idListOfBoolOrℕ′ (1 ∷ 2 ∷ 3 ∷ [])
```

This works as follows: the expected type of an argument (`ListOfBoolOrℕ′ _b`) gets unified with the actual one (`List ℕ`):

    ListOfBoolOrℕ′ _b =?= List ℕ

after expanding `ListOfBoolOrℕ′` we get

    List (BoolOrℕ _b) =?= List ℕ

as usual `List` gets stripped from both the sides of the equation:

    BoolOrℕ _b =?= ℕ

and here Agda has a special rule, quoting the wiki:

> If all right hand sides of a function definition have distinct (type or value) constructor heads, we can deduce the shape of the arguments to the function by looking at the head of the expected result.

In our case two "constructor heads" in the definition of `BoolOrℕ` are `Bool` and `ℕ`, which are distinct, and that makes Agda see that `BoolOrℕ` is injective, so unifying `BoolOrℕ _b` with `ℕ` amounts to finding the clause where `ℕ` is returted from `BoolOrℕ`, which is

    BoolOrℕ true  = ℕ

and this determines that for the result to be `ℕ` the value of `_b` must be `true`, so the unification problem gets solved as

    _b := true

`BoolOrℕ` differs from

    ListOfBoolOrℕ : Bool -> Set
    ListOfBoolOrℕ false = List Bool
    ListOfBoolOrℕ true  = List ℕ

in that the latter definition has the same head in both the clauses (`List`) and so the heuristic doesn't apply. Even though Agda really could have figured out that `ListOfBoolOrℕ` is also injective. I.e. the fact that `ListOfBoolOrℕ` is not consdered invertible is more of an implementation detail than a theoretical limination.

Here's an example of a theoretical limitation: a definition like

```agda
  BoolOrBool : Bool -> Set
  BoolOrBool true  = Bool
  BoolOrBool false = Bool
```

can't be inverted, because the result (`Bool` in both the cases) does not determine the argument (either `true` or `false`).

#### Example 1: universe of types

There's a standard technique ([the universe pattern](https://groups.google.com/forum/#!msg/idris-lang/N9_pVqG8dO8/mHlNmyL6AwAJ)) that allows us to get ad hoc polymorphism (a.k.a. type classes) for a closed set of types in a dependently typed world.

We introduce a universe of types, which is a data type containing tags for actual types:

```agda
  data Uni : Set where
    bool nat : Uni
    list : Uni -> Uni
```

interpret those tags as types that they encode:

```agda
  ⟦_⟧ : Uni -> Set
  ⟦ bool   ⟧ = Bool
  ⟦ nat    ⟧ = ℕ
  ⟦ list A ⟧ = List ⟦ A ⟧
```

and then mimic the `Eq` type class for the types from this universe by directly defining equality functions:

```agda
  _==Bool_ : Bool -> Bool -> Bool
  true  ==Bool true  = true
  false ==Bool false = true
  _     ==Bool _     = false

  _==ℕ_ : ℕ -> ℕ -> Bool
  zero  ==ℕ zero  = true
  suc n ==ℕ suc m = n ==ℕ m
  _     ==ℕ _     = false

  mutual
    _==List_ : ∀ {A} -> List ⟦ A ⟧ -> List ⟦ A ⟧ -> Bool
    []       ==List []       = true
    (x ∷ xs) ==List (y ∷ ys) = (x == y) && (xs ==List ys)
    _        ==List _        = false

    _==_ : ∀ {A} -> ⟦ A ⟧ -> ⟦ A ⟧ -> Bool
    _==_ {nat   } x y = x ==ℕ    y
    _==_ {bool  } x y = x ==Bool y
    _==_ {list A} x y = x ==List y
```

`_==_` checks equality of two elements from any type from the universe.

Note that `_==List_` is defined mutually with `_==_`, because elements of lists can be of any type from the universe, i.e. they can also be lists, hence the mutual recursion.

A few tests:

```agda
  -- Check equality of two equal elements of `ℕ`.
  _ : (42 == 42) ≡ true
  _ = refl

  -- Check equality of two non-equal elements of `List Bool`.
  _ : ((true ∷ []) == (false ∷ [])) ≡ false
  _ = refl

  -- Check equality of two equal elements of `List (List ℕ)`.
  _ : (((4 ∷ 81 ∷ []) ∷ (57 ∷ 2 ∷ []) ∷ []) == ((4 ∷ 81 ∷ []) ∷ (57 ∷ 2 ∷ []) ∷ [])) ≡ true
  _ = refl

  -- Check equality of two non-equal elements of `List (List ℕ)`.
  _ : (((4 ∷ 81 ∷ []) ∷ (57 ∷ 2 ∷ []) ∷ []) == ((4 ∷ 81 ∷ []) ∷ [])) ≡ false
  _ = refl
```

It's possible to leave `A` implicit in `_==_` and get it inferred in the tests above precisely because `⟦_⟧` is constructor-headed. If we had `bool₁` and `bool₂` tags both mapping to `Bool`, inference for `_==_` would not work for booleans, lists of booleans etc. In the version of Agda I'm using inference for naturals, lists of naturals etc still works though, if an additional `bool` is added to the universe, i.e. breaking constructor-headedness of a function for certain arguments does not result in inference being broken for others.

#### Example 2: `boolToℕ`

Constructor-headed functions can also return values rather than types. For example this function:

```agda
  boolToℕ : Bool -> ℕ
  boolToℕ false = zero
  boolToℕ true  = suc zero
```

is constructor-headed, because in the two clauses heads are constructors and they're different (`zero` vs `suc`).

So if we define a version of `id` that takes a `Vec` with either 0 or 1 element:

```agda
  idVecAsMaybe : ∀ {b} -> Vec ℕ (boolToℕ b) -> Vec ℕ (boolToℕ b)
  idVecAsMaybe xs = xs
```

then it won't be necessary to specify `b`:

```agda
  _ = idVecAsMaybe []ᵥ
  _ = idVecAsMaybe (0 ∷ᵥ []ᵥ)
```

as Agda knows how to solve `boolToℕ _b =?= zero` or `boolToℕ _b =?= suc zero` due to `boolToℕ` being invertible.

`idVecAsMaybe` supplied with a vector of length greater than `1` correctly gives an error (as opposed to merely reporting that there's an unsolved meta):

    -- suc _n_624 != zero of type ℕ
    _ = idVecAsMaybe (0 ∷ᵥ 1 ∷ᵥ []ᵥ)

Note that `boolToℕ` defined like that:

```agda
  boolToℕ′ : Bool -> ℕ
  boolToℕ′ false = zero + zero
  boolToℕ′ true  = suc zero
```

is not considered to be constructor-headed as the second test in

```agda
  idVecAsMaybe′ : ∀ {b} -> Vec ℕ (boolToℕ′ b) -> Vec ℕ (boolToℕ′ b)
  idVecAsMaybe′ xs = xs

  _ = idVecAsMaybe′ []ᵥ
  _ = idVecAsMaybe′ (0 ∷ᵥ []ᵥ)
```

is now yellow. But not the first one. Which is weird. Can't explain that.

I.e. Agda does not always reduce the rhs of the clauses of a function when checking for constructor-headedness, which is unfortunate.



#### Example 3: polyvariadic `zip`

```agda
module PolyvariadicZip where
  open import Data.List.Base as List
  open import Data.Vec.Base as Vec renaming (_∷_ to _∷ᵥ_; [] to []ᵥ)
```

We can define this family of functions over vectors:

    replicate : ∀ {n} → A → Vec A n
    map : ∀ {n} → (A → B) → Vec A n → Vec B n
    zipWith : ∀ {n} → (A → B → C) → Vec A n → Vec B n → Vec C n
    zipWith3 : ∀ {n} → (A → B → C → D) → Vec A n → Vec B n → Vec C n → Vec D n

(the Agda stdlib provides all functions but the last one)

Can we define a generic function that covers all of the above? Its type signature should look like this:

    (A₁ -> A₂ -> ... -> B) -> Vec A₁ n -> Vec A₂ n -> ... -> Vec B n

Yes: we can parameterize a function by a list of types and compute those n-ary types from the list. Folding a list of types into a type, given also the type of the result, is trivial:

```agda
  ToFun : List Set -> Set -> Set
  ToFun []       B = B
  ToFun (A ∷ As) B = A -> ToFun As B
```

This allows us to compute the n-ary type of the function. In order to compute the n-ary type of the result we need to map the list of types with `λ A -> Vec A n` and turn `B` (the type of the resulting of the zipping function) into `Vec B n` (the type of the final result):

```agda
  ToVecFun : List Set -> Set -> ℕ -> Set
  ToVecFun As B n = ToFun (List.map (λ A -> Vec A n) As) (Vec B n)
```

It only remains to recurse on the list of types in an auxiliary function (n-ary `(<*>)`, using Haskell jargon) and define `zipN` in terms of that function:

```agda
  apN : ∀ {As B n} -> Vec (ToFun As B) n -> ToVecFun As B n
  apN {[]}     ys = ys
  apN {A ∷ As} fs = λ xs -> apN {As} (fs ⊛ xs)

  zipN : ∀ {As B n} -> ToFun As B -> ToVecFun As B n
  zipN f = apN (Vec.replicate f)
```

Some tests verifying that the function does what it's supposed to:

```agda
  _ : zipN 1 ≡ (1 ∷ᵥ 1 ∷ᵥ 1 ∷ᵥ []ᵥ)
  _ = refl

  _ : zipN suc (1 ∷ᵥ 2 ∷ᵥ 3 ∷ᵥ []ᵥ) ≡ (2 ∷ᵥ 3 ∷ᵥ 4 ∷ᵥ []ᵥ)
  _ = refl

  _ : zipN _+_ (1 ∷ᵥ 2 ∷ᵥ 3 ∷ᵥ []ᵥ) (4 ∷ᵥ 5 ∷ᵥ 6 ∷ᵥ []ᵥ) ≡ (5 ∷ᵥ 7 ∷ᵥ 9 ∷ᵥ []ᵥ)
  _ = refl
```

Note how we do not provide the list of types explicitly in any of these cases, even though there's pattern matching on that list.

```agda
  _ : ∀ {n} -> ToFun _ _ ≡ (ℕ -> ℕ -> ℕ)
  _ = refl

  _ : ∀ {n} -> ToVecFun _ _ n ≡ (Vec ℕ n -> Vec ℕ n -> Vec ℕ n)
  _ = refl
```

### Constructor/argument-headed functions

```agda
module ConstructorArgumentHeadedFunctions where
  open DataConstructors
```

Recall that we've been using a weird definition of plus

> because the usual one
>
>     _+_ : ℕ -> ℕ -> ℕ
>     zero  + m = m
>     suc n + m = suc (n + m)
>
> is subject to certain unification heuristics, which the weird one doesn't trigger.

The usual definition is this one:

    _+_ : ℕ -> ℕ -> ℕ
    zero  + m = m
    suc n + m = suc (n + m)

As you can see here we return one of the arguments in the first clause and the second clause is constructor-headed. Just like for regular constructor-headed function, Agda has enhanced inference for functions of this kind as well.

Quoting the [changelog](https://github.com/agda/agda/blob/064095e14042bdf64c7d7c97c2869f63f5f1f8f6/doc/release-notes/2.5.4.md#pattern-matching):

> Improved constraint solving for pattern matching functions
> Constraint solving for functions where each right-hand side has a distinct rigid head has been extended to also cover the case where some clauses return an argument of the function. A typical example is append on lists:
>
>     _++_ : {A : Set} → List A → List A → List A
>     []       ++ ys = ys
>     (x ∷ xs) ++ ys = x ∷ (xs ++ ys)
>
> Agda can now solve constraints like ?X ++ ys == 1 ∷ ys when ys is a neutral term.

#### Example 1: back to `idᵥ⁺`

Now if we come back to this example (TODO: add highlighting somehow?):

> `idᵥ⁺` applied to a non-constant vector has essentially the same inference properties.
>
> Without specializing the implicit arguments we get yellow:
>
>
>     _ = λ n m (xs : Vec ℕ (n +′ m)) -> idᵥ⁺ xs
>
>
> Specializing `m` doesn't help, still yellow:
>
>
>     _ = λ n m (xs : Vec ℕ (n +′ m)) -> idᵥ⁺ {m = m} xs
>

but define `idᵥ⁺` over `_+_` rather than `_+′_`:

```agda
  idᵥ⁺ : ∀ {A n m} -> Vec A (n + m) -> Vec A (n + m)
  idᵥ⁺ xs = xs
```

then

```agda
  _ = λ n m (xs : Vec ℕ (n + m)) -> idᵥ⁺ {m = m} xs
```

type checks perfectly.

And

```
  _ = λ n m (xs : Vec ℕ (n + m)) -> idᵥ⁺ xs
```

still gives yellow, because it's still ambiguous.

Additionally, this now also type checks:

```agda
  _ = idᵥ⁺ {m = 0} (1 ∷ᵥ 2 ∷ᵥ []ᵥ)
```

This is because instantiating `m` at `0` in `idᵥ⁺` makes `_+_` constructor-headed, because if we inline `m` in the definition of `_+_`, we'll get:

    _+0 : ℕ -> ℕ
    zero  +0 = zero
    suc n +0 = suc (n +0)

which is clearly constructor-headed.

And

```agda
  _ = idᵥ⁺ {m = 1} (1 ∷ᵥ 2 ∷ᵥ []ᵥ)
```

still does not type check, because inlining `m` as `1` does not make `_+_` constructor-headed:

    _+1 : ℕ -> ℕ
    zero  +1 = suc zero
    suc n +1 = suc (n +1)





####

```agda
  1OrDouble : Bool -> ℕ -> ℕ
  1OrDouble false n = suc zero
  1OrDouble true  n = 0 + 0
```

```agda
  _ : 1OrDouble _ 0 ≡ 1
  _ = refl
```




## Eta-rules

```agda
module EtaRules where
```

Agda implements eta-rules for "negative" types (TODO: link).

One such rule is that a function is definitionally equal to its eta-expanded version:

```agda
  _ : ∀ {A : Set} {B : A -> Set} -> (f : ∀ x -> B x) -> (f ≡ (λ x -> f x))
  _ = λ f -> refl
```

Usefulness of this eta-rule is not something that one thinks of much, but that is only until they try to work in a language that doesn't support the rule (spoiler: it's a huge pain).

All records support eta-rules by default (that can be switched off for a single record via an explicit `no-eta-equality` mark or for all records via TODO: what).

The simplest record is one with no fields:

```agda
  record Unit : Set where
    constructor unit
```

The eta-rule for `Unit` is "all terms of type `Unit` are equal to `unit`":

```agda
  _ : (u : Unit) -> u ≡ unit
  _ = λ u -> refl
```

Consequently, since all terms of type `Unit` are equal to `unit`, they are also equal to each other:

```agda
  _ : (u1 u2 : Unit) -> u1 ≡ u2
  _ = λ u1 u2 -> refl
```

This eta-rule applies to `⊤`, precisely because `⊤` is defined as a record with no fields.

For a record with fields the eta-rule is "an element of the record is always the constructor of the record applied to its fields". For example:

```agda
  record Triple (A B C : Set) : Set where
    constructor triple
    field
      fst : A
      snd : B
      thd : C
  open Triple

  _ : ∀ {A B C} -> (t : Triple A B C) -> t ≡ triple (fst t) (snd t) (thd t)
  _ = λ t -> refl
```

Supporting eta-equality for sum types is possible in theory (TODO: link), but Agda does not implement that. Any `data` definition in Agda does not support eta-equality, including an empty `data` declaration like

```agda
  data Empty : Set where
```

(which is always isomorphic to `Data.Empty.⊥` and is how `⊥` is defined in the first place).

Eta-rules for records may seem not too exciting, but there are a few important use cases.

### Computing predicates

### N-ary things

Composition (TODO: link)

```agda



--   _ = Bool ∷ᵥ []ᵥ
```

    λ g y          -> g y
    λ g f x        -> g (f x)
    λ g f x₁ x₂    -> g (f x₁ x₂)
    λ g f x₁ x₂ x₃ -> g (f x₁ x₂ x₃)


```agda



  -- _ : Vec.map suc (1 ∷ᵥ 2 ∷ᵥ 3 ∷ᵥ []ᵥ) ≡ _
  -- _ = refl

  -- _ : Vec.zipWith _+_ (1 ∷ᵥ 2 ∷ᵥ 3 ∷ᵥ []ᵥ) (4 ∷ᵥ 5 ∷ᵥ 6 ∷ᵥ []ᵥ) ≡ _
  -- _ = refl

  -- _ : zipN suc (1 ∷ᵥ 2 ∷ᵥ 3 ∷ᵥ []ᵥ) ≡ _
  -- _ = refl

  -- _ : zipN _+_ (1 ∷ᵥ 2 ∷ᵥ 3 ∷ᵥ []ᵥ) (4 ∷ᵥ 5 ∷ᵥ 6 ∷ᵥ []ᵥ) ≡ _
  -- _ = refl

  -- _ : zipN {_ ∷ []} suc (1 ∷ᵥ 2 ∷ᵥ 3 ∷ᵥ []ᵥ) ≡ _
  -- _ = refl

  -- _ : zipN {_ ∷ _ ∷ []} _+_ (1 ∷ᵥ 2 ∷ᵥ 3 ∷ᵥ []ᵥ) (4 ∷ᵥ 5 ∷ᵥ 6 ∷ᵥ []ᵥ) ≡ _
  -- _ = refl
```

```agda
module zipVN where
  open import Data.Vec.Base renaming (_∷_ to _∷ᵥ_; [] to []ᵥ)

  ToFun : ∀ {k} -> Vec Set k -> Set -> Set
  ToFun {zero}  As B = B
  ToFun {suc k} As B = head As -> ToFun (tail As) B

  apN
    : ∀ k {As : Vec Set k} {B n}
    -> Vec (ToFun As B) n
    -> ToFun (map (λ A -> Vec A n) As) (Vec B n)
  apN zero             ys = ys
  -- Have to pattern match on `As` here, otherwise stuff doesn't reduce properly
  -- and the whole thing does not type check.
  apN (suc k) {_ ∷ᵥ _} fs = λ xs -> apN k (fs ⊛ xs)

  zipN
    : ∀ k {As : Vec Set k} {B n}
    -> ToFun As B
    -> ToFun (map (λ A -> Vec A n) As) (Vec B n)
  zipN k f = apN k (replicate f)

  _ : zipN 1 suc (1 ∷ᵥ 2 ∷ᵥ 3 ∷ᵥ []ᵥ) ≡ _
  _ = refl

  _ : zipN 2 _+_ (1 ∷ᵥ 2 ∷ᵥ 3 ∷ᵥ []ᵥ) (4 ∷ᵥ 5 ∷ᵥ 6 ∷ᵥ []ᵥ) ≡ _
  _ = refl
```

-- Vec.map f xs



## Universe levels

```agda
module UniverseLevels where
```

There are a bunch of definitional equalities associated with universe levels. Without them universe polymorphism would be nearly unusable. Here are the equalities:

```agda
_ : ∀ {α} -> lzero ⊔ α ≡ α
_ = refl

_ : ∀ {α} -> α ⊔ α ≡ α
_ = refl

_ : ∀ {α} -> lsuc α ⊔ α ≡ lsuc α
_ = refl

_ : ∀ {α β} -> α ⊔ β ≡ β ⊔ α
_ = refl

_ : ∀ {α β γ} -> (α ⊔ β) ⊔ γ ≡ α ⊔ (β ⊔ γ)
_ = refl

_ : ∀ {α β} -> lsuc α ⊔ lsuc β ≡ lsuc (α ⊔ β)
_ = refl
```

A demonstration of how Agda can greatly simplify level expressions using the above identites:

```agda
_ : ∀ {α β γ} -> lsuc α ⊔ (γ ⊔ lsuc (lsuc β)) ⊔ lzero ⊔ (β ⊔ γ) ≡ lsuc (α ⊔ lsuc β) ⊔ γ
_ = refl
```

These special rules also give us an ability to define a less-than-or-equal-to relation on levels:

```agda
_≤ℓ_ : Level -> Level -> Set
α ≤ℓ β = α ⊔ β ≡ β
```

which in turn allows to [emulate cumulativity of universes](http://effectfully.blogspot.com/2016/07/cumu.html) in Agda (although there is an experimental option [`--cumulativity`](https://agda.readthedocs.io/en/latest/language/cumulativity.html) that makes the universe hierarchy cumulative).

The list of equalities shown above is not exhaustive. E.g. if during type checking Agda comes up with the following constraint:

    α <= β <= α

it gets solved as `α ≡ β`.





















## Local definitions

no let-generalization
lets are transparent
https://agda.readthedocs.io/en/v2.6.0.1/language/mutual-recursion.html
https://agda.readthedocs.io/en/v2.6.0.1/language/let-and-where.html

## Pattern matching


```agda
module Blah where
  data Uni : Set where
    bool₁ bool₂ : Uni

  -- record Tag {ι α} {I : Set ι} (i : I) (A : Set α) : Set α where
  --   constructor tag
  --   field untag : A
  --   tagOf = ι

  -- tagWith : ∀ {ι α} {I : Set ι} {A : Set α} -> (i : I) -> A -> Tag i A
  -- tagWith _ = tag

  -- ⟦_⟧ : Uni -> Set
  -- ⟦ bool₁ ⟧ = Tag bool₁ Bool
  -- ⟦ bool₂ ⟧ = Tag bool₂ Bool

  record Bool₁ : Set where
    constructor mkBool₁
    field unBool₁ : Bool
  open Bool₁

  record Bool₂ : Set where
    constructor mkBool₂
    field unBool₂ : Bool
  open Bool₂

  ⟦_⟧ : Uni -> Set
  ⟦ bool₁ ⟧ = Bool₁
  ⟦ bool₂ ⟧ = Bool₂

  _==Bool_ : Bool -> Bool -> Bool
  true  ==Bool true  = true
  false ==Bool false = true
  _     ==Bool _     = false

  _==_ : ∀ {A} -> ⟦ A ⟧ -> ⟦ A ⟧ -> Bool
  _==_ {bool₁} x y = unBool₁ x ==Bool unBool₁ y
  _==_ {bool₂} x y = unBool₂ x ==Bool unBool₂ y

  _ : (mkBool₁ true == mkBool₁ false) ≡ false
  _ = refl
```


{-

record Tag {α β} {A : Set α} (B : (x : A) -> Set β) (x : A) : Set (α ⊔ β) where
  constructor tag
  field el : B x
  tagOf = x
open Tag public

-- Explicit tagging.
tagWith : ∀ {α β} {A : Set α} {B : (x : A) -> Set β} -> (x : A) -> B x -> Tag B x
tagWith _ = tag

Tag₂ : ∀ {α β γ} {A : Set α} {B : A -> Set β} -> (∀ x -> B x -> Set γ) -> ∀ x -> B x -> Set γ
Tag₂ C x y = Tag (uncurry C) (x , y)


## Inconvenient recursion

Vector.foldl



## Constructor-headed functions

Mention   `_ = idᵥ⁺ {m = 0} (1 ∷ᵥ 2 ∷ᵥ []ᵥ)`


module ConstructorHeadedFunctions where
  open import Data.Unit.Base
  open import Data.List.Base

  record Arg {α} {A : Set α} (x : A) : Set where
    unarg = x

  check : ∀ {α} {A : Set α} -> A -> A -> A
  check _ y = y

  module _ where
    private
      this : ∀ {A B : Set} {f : A -> B} {xs} -> Arg (map f xs) -> ⊤
      this = check this _

-- map : (A → B) → List A → List B
-- map f []       = []
-- map f (x ∷ xs) = f x ∷ map f xs

  map′ : {A B : Set} -> (A -> B) -> List A -> List B
  map′ f = foldr (_∷_ ∘ f) []

  module _ where
    private
      this : ∀ {A B : Set} {f : A -> B} {xs} -> Arg (map′ f xs) -> ⊤
      this = check this _

  _ : map suc _ ≡ (1 ∷ 2 ∷ 3 ∷ [])
  _ = refl

  _ : map′ suc _ ≡ (1 ∷ 2 ∷ 3 ∷ [])
  _ = refl

  module _ where
    private
      this : ∀ {n m} -> Arg (n + m) -> ⊤
      this = check this _

  module _ where
    private
      this : ∀ {m} n -> Arg (n + m) -> ⊤
      this = check this _

  module _ where
    private
      this : ∀ {n} m -> Arg (n + m) -> ⊤
      this = check this _

  module _ where
    private
      this : ∀ {n} m -> Arg (n ∸ m) -> ⊤
      this = check this _

  module _ where
    private
      this : ∀ {m} n -> Arg (n ∸ m) -> ⊤
      this = check this _

  module _ where
    private
      this : ∀ {m} n -> Arg (n * m) -> ⊤
      this = check this _

  module _ where
    private
      this : ∀ {n} m -> Arg (n * m) -> ⊤
      this = check this _

  module _ where
    private
      this : ∀ {n} -> Arg (Data.Nat.Base.pred n) -> ⊤
      this = check this _

nary : ℕ -> Set -> Set
nary  0      A = A
nary (suc n) A = A -> nary n A

_ : nary _ (ℕ -> ℕ) ≡ (ℕ -> ℕ)
_ = refl





## A function is not dependent enough

-- ᵏ′ : ∀ {α β} {A : Set α} {B : A -> Set β} -> (∀ {x} -> B x) -> ∀ x -> B x
-- ᵏ′ y x = y

mention the Kipling paper

## mention the Jigger

## Talk about heterogeneous equality?

## η-laws

## auto

-- If (n ∸ m) is in canonical form,
-- then (n ≤ℕ m) reduces either to (⊤) or to (⊥).
-- The value of (⊤) can be inferred automatically,
-- which is exploited by the (ᵀ≤ᵀ) constructor of the (_≤_) datatype.
-- It would be nice to have a type error, when (n ≤ℕ m) reduces to (⊥).
_≤ℕ_ : ℕ -> ℕ -> Set
0     ≤ℕ _     = ⊤
suc _ ≤ℕ 0     = ⊥
suc n ≤ℕ suc m = n ≤ℕ m

```agda
open import Data.Product

_ : {A : Set} {B : A -> Set} {p : Σ A B} -> (proj₁ p , proj₂ p) ≡ p
_ = refl
```

record Is {α} {A : Set α} (x : A) : Set α where
  ¡ = x
open Is

! : ∀ {α} {A : Set α} -> (x : A) -> Is x
! _ = _



_-⁺_ : ∀ {m} -> ℕ -> Is (suc m) -> ℕ
n -⁺ im = n ∸ ¡ im

```agda
-- record ⊤ : Set where
--   constructor tt
--
-- tt′ : ⊤
-- tt′ = _
```

## Inferring functions

## Higher-order dynamic pattern unification

http://adam.gundry.co.uk/pub/pattern-unify/
www.cse.chalmers.se/~abela/unif-sigma-long.pdf
https://www.researchgate.net/publication/228571999_Type_checking_in_the_presence_of_meta-variables

Ulf Norell. [Towards a practical programming language based on dependent type theory](http://www.cse.chalmers.se/~ulfn/papers/thesis.html). PhD thesis, 2007.



mention _% = _∘_?














mention Jesper's work and the green slime problem




lazily match on index of a singleton, then match on the singleton where it's needed


-- ∀ A B -> (f : ∀ x -> B x) -> (C : (A -> Set) -> Set) -> (z : C B) -> C B

forCheckT1 :: Syntax
forCheckT1 = forall "A"
           $ forall "B"
           $ Pi "f" (forall "x" $ App "B" [var "x"])
           $ Pi "C" ((var "A" ~> Star) ~> Star)
           $ Pi "z" (App "C" [var "B"])
           $ App "C" [var "B"]

forCheck1 :: Syntax
forCheck1 = Lam "A" $ Lam "B" $ Lam "f" $ Lam "C" $ Lam "z" $ var "z"









forCheckT2 :: Syntax
forCheckT2 = forall "A"
           $ Pi "f" (forall "B" $ forall "x" $ App "B" [var "x"])
           $ Pi "C" ((Pi "B" (var "A" ~> Star) $ forall "x" $ App "B" [var "x"]) ~> Star)
           $ Pi "z" (App "C" [var "f"])
           $ App "C" [var "f"]

forCheck2 :: Syntax
forCheck2 = Lam "A" $ Lam "f" $ Lam "C" $ Lam "z" $ var "z"

```agda
_ : (A : Set) -> (f : ∀ B x -> B x) -> (C : ((B : A -> Set) -> ∀ x -> B x) -> Set) -> C f -> C f
_ = λ A f C z -> z
```

-- The expression

-- ∀ A -> (f : ∀ B x -> B x) -> (C : (B : A -> Type) -> ∀ x -> B x) -> C f -> C f

-- is elaborated to

-- (A : ?0) -> (f : (B : ?1 A) -> (x : ?2 A B) -> B x) ->
--   (C : (B : A -> Type) -> (x : ?3 A f B) -> B x) -> C f -> C f

-- `?0` is solved by `?0 ≡ Type`.

-- `B x` can't type check, because `B` doesn't have enough Πs in its type,
-- so it's replaced by `?4 A B x` which will compute to `B x` as soon as
-- there are enough Πs and `B x` is successfully type checked.

-- `?3` is solved by `?3 ≡ \A f B -> A`.

-- The expression now is

-- (A : Type) -> (f : (B : ?1 A) -> (x : ?2 A B) -> ?4 A B x) ->
--   (C : (B : A -> Type) -> (x : A) -> B x) -> C f -> C f

-- An attempt to type check `C f` forces unification of these two types:

-- (B : ?1 A)      -> (x : ?2 A B) -> ?4 A B x
-- (B : A -> Type) -> (x : A)      -> B x

-- `?1` is solved by `?1 ≡ \A -> A -> Type`.

-- Now there are enough Πs, so `?4 A B x` is type checked, which gives `?2 ≡ \A B -> A` and
-- `?4 ≡ \A B x -> B x`. It only remains to unify

-- (x : A) -> B x
-- (x : A) -> B x

-- which is trivial.

-- The final `C f` is type checked with all metas being already solved, so it's trivial too.

-- The fully elaborated expression:

-- (A : Type) -> (f : (B : A -> Type) -> (x : A) -> B x) ->
--   (C : (B : A -> Type) -> (x : A) -> B x) -> C f -> C f








```agda
open import Function

_ : (A : Set) -> (B : A -> Set) -> (f : ∀ x -> A -> B x) -> _
_ = λ A B f x -> f x x
```

-- The expression

-- ∀ A -> (B : A -> Type) -> (f : ∀ x -> A -> B x) -> _

-- is elaborated to

-- (A : ?0) -> (B : A -> Type) -> (f : (x : ?1 A B) -> A -> B x) -> ?2 A B f

-- which after type checking becomes

-- (A : Type) -> (B : A -> Type) -> (f : (x : A) -> A -> B x) -> ?2 A B f

-- Then

-- \A B f x -> f x x

-- is checked against this type. The problem simplifies to

-- (\x -> f x x) ∈? ?2 A B f

-- thus `?2 A B f` represents a functional type, so two fresh metavariables
-- (for domain and codomain) are introduced and the problem now looks like

-- (\x -> f x x) ∈? (x : ?3 A B f) -> ?4 A B f x

-- which simplifies to

-- f x x ∈? ?4 A B f x

-- with `x` being of type `?3 A B f`. `f` receives two `A`s, but is applied to two `?3 A B f`s,
-- hence the former is unified with the latter and `?3` is solved by `?3 ≡ \A B f -> A`.

-- The inferred type of `f x x` is `B x` while the expected is `?4 A B f x`,
-- hence `?4 ≡ \A B f x -> B x`.

-- Finally, the original type of `\x -> f x x` is unified with
-- the type `\x -> f x x` was checked against, which after normalization looks like

-- ?2 A B f =?= (x : A) -> B x

-- which is trivially `?2 ≡ \A B f -> (x : A) -> B x`.

-- The fully elaborated expression:

-- testLam : (A : Type) -> (B : A -> Type) -> (f : (x : A) -> A -> B x) -> (x : A) -> B x
-- testLam A B f x = f x x


forNonDepT :: Syntax
forNonDepT = forall "A"
           $ Pi "B" (var "A" ~> Star)
           $ (forall "B" $ (Pi "x" (var "A") $ var "B") ~> var "A" ~> var "B")
           ~> (forall "x" $ App "B" [var "x"]) ~> (forall "x" $ App "B" [var "x"])

forNonDep :: Syntax
forNonDep = Lam "A" $ Lam "B" $ Lam "C" $ App "C" [(:?)]

-- Left "?9 'A 'B 'C and 'B 'x can't be unified"
testNonDep = evalTCM1 (stypecheck forNonDep forNonDepT)






  1.4) `f' is constructor-headed [1]. For example `Data.Product.N-ary' defines

    _^_ : ∀ {ℓ} → Set ℓ → ℕ → Set ℓ
    A ^ 0           = Lift ⊤
    A ^ 1           = A
    A ^ suc (suc n) = A × A ^ suc n

It's not constructor-headed: if (A ^ n = ℕ × ℕ) you get two solutions:
  1) A = ℕ     , n = 2
  2) A = ℕ × ℕ , n = 1

But

    _^_ : ∀ {ℓ} → Set ℓ → ℕ → Set ℓ
    A ^ 0       = Lift ⊤
    A ^ (suc n) = A × A ^ suc n

is constructor-headed, and Agda can infer both `A' and `n' from (A ^ n
= B), if `B' is known.

You can make the first definition constructor-headed by wrapping `A'
in the (A ^ 1) case in an additional datatype:

    record Wrap {α} (A : Set α) : Set α where
      constructor wrap
      field el : A

    _^_ : ∀ {ℓ} → Set ℓ → ℕ → Set ℓ
    A ^ 0           = Lift ⊤
    A ^ 1           = Wrap A
    A ^ suc (suc n) = A × A ^ suc n

    test-wrap : _ ^ _
    test-wrap = 0 , 3 , 5 , wrap 9

2) Agda usually can't infer `f' from (f x)

    fun : {L : Set -> Set} -> L ℕ -> ⊤
    fun _ = _

    fail-fun : ⊤
    fail-fun = fun []

results in unresolved metas. But you can help she by limiting the
scope of search using instance arguments:

    record IsNatList (L : Set -> Set) : Set where
      field f : ⊤

    instance
      inst : IsNatList List
      inst = record { f = _ }

    fun-inst : {L : Set -> Set} {{_ : IsNatList L}} -> L ℕ -> ⊤
    fun-inst _ = _

    test-fun-inst : ⊤
    test-fun-inst = fun-inst []

No unresolved metas. Strangely this doesn't work if I omit the
pointless field in the `IsNatList' datatype.

3) Agda's definitional equality is beta-eta, so (proj₁ p , proj₂ p)
reduces to just `p' during typechecking. Using eta-rules Agda can
infer elements of records:

    auto-records : ⊤ × Lift {ℓ = Le.suc (Le.suc Le.zero)} ⊤
    auto-records = _

4) Sometimes Agda can infer more stuff in mutual blocks, but it's
black magic to me.

5) Have a look at [2] for more complicated cases.

[1] http://wiki.portal.chalmers.se/agda/pmwiki.php?n=ReferenceManual.FindingTheValuesOfImplicitArguments
[2] http://www2.tcs.ifi.lmu.de/~abel/talkTLCA11.pdf
-}






{-
I said "Agda usually can't infer `f' from (f x)", but she can in some
useful cases. As an example

    bad-fancy-apply : ∀ {n} -> (List ℕ -> Vec ℕ n) -> List ℕ -> Vec ℕ n
    bad-fancy-apply = id

Here `n' doesn't depend on anything, so if we pass the `fromList'
function, which is defined as

    fromList : ∀ {a} {A : Set a} → (xs : List A) → Vec A (List.length xs)
    fromList List.[]         = []
    fromList (List._∷_ x xs) = x ∷ fromList xs

to `bad-fancy-apply`, we'll get cannot-instantiate-error, because the
length of the resulting vector does depend. But we can fix this:

    fancy-apply : {k : List ℕ -> ℕ} -> ((xs : List ℕ) -> Vec ℕ (k xs))
-> (xs : List ℕ) -> Vec ℕ (k xs)
    fancy-apply = id

    test-fancy-apply : _
    test-fancy-apply = fancy-apply fromList (1 ∷ 2 ∷ 3 ∷ [])

Now `fancy-apply' reflects the dependency in `fromList'. And Agda is
able to infer that `k'.

Here is an even more fancy example:

    z : (k : Level -> Level) -> Set (suc (k zero))
    z k = Set (k zero)

    s : {j : (Level -> Level) -> Level}
      -> ((k : Level -> Level) -> Set (suc (j k)))
      -> (k : Level -> Level)
      -> Set (suc (k zero ⊔ j (k ∘ suc)))
    s r k = Set (k zero) -> r (k ∘ suc)

    crescendo : {j : (Level -> Level) -> Level}
              -> ((k : Level -> Level) -> Set (j k)) -> Set (j id)
    crescendo r = r id

    test : crescendo (s (s (s z))) ≡ (Set₀ -> Set₁ -> Set₂ -> Set₃)
    test = refl

Still Agda can infer pretty complicated `j' in both `s' and `crescendo'.

I also encoded an ECC, and a bounded dependent quantifier was

    _≥Π_ : ∀ {α}
         -> (A : Type α) {k : ∀ {α'} {A' : Type α'} -> A' ≤ A -> level}
         -> (∀ {α'} {A' : Type α'} {le : A' ≤ A} -> ≤⟦ le ⟧ᵂ -> Type (k le))
         -> Type (α ⊔ᵢ k (≤-refl A))

And inferring `k' still wasn't a problem.

But it's hard to predict, when such an inferring becomes a problem. I
encountered some cases, that look simple to me, but not to Agda.
-}













```agda
open import Level renaming (suc to lsuc; zero to lzero)
open import Relation.Binary.PropositionalEquality
open import Function

{- z : (k : Level -> Level) -> Set (lsuc (k lzero))
z k = Set (k lzero)

s : {j : (Level -> Level) -> Level}
  -> (∀ k -> Set (lsuc (j k)))
  -> (∀ k -> Set (lsuc (k lzero ⊔ j (k ∘ lsuc))))
s r k = Set (k lzero) -> r (k ∘ lsuc)

crescendo : {j : (Level -> Level) -> Level} -> (∀ k -> Set (j k)) -> Set (j id)
crescendo r = r id

test : crescendo {λ z₁ →
                      lsuc (z₁ lzero) ⊔
                      lsuc
                      (z₁ (lsuc lzero) ⊔ z₁ (lsuc (lsuc lzero)) ⊔
                       z₁ (lsuc (lsuc (lsuc lzero))))} (s (s (s z))) ≡ (Set₀ -> Set₁ -> Set₂ -> Set₃)
test = refl-}

z : ∀ l -> Set (lsuc l)
z l = Set l

s : {j : Level -> Level}
  -> (∀ l -> Set (lsuc (j l)))
  -> (∀ l -> Set (lsuc (l ⊔ j (lsuc l))))
s r l = Set l -> r (lsuc l)

crescendo : {j : Level -> Level} -> (∀ l -> Set (j l)) -> Set (j lzero)
crescendo r = r lzero

test : crescendo (s (s (s z))) ≡ (Set₀ -> Set₁ -> Set₂ -> Set₃)
test = refl

```

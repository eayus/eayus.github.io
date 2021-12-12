---
layout: post
title: "Types are contexts"
description: ""
tags: [web, jekyll]
---

# Introduction

For anyone that has ever implemented a language with algebraic data types, the idea that types are just contexts is probably already familiar to you. Typically, when a compiler reaches a data type definition, it simply adds the constructors' types into its context. Additionally, it might explicitly mark the types as being constructors, so that we know how to pattern match on it later. For example, the definition of natural numbers:

{% highlight haskell %}
data Nat : Type where
    Zero : Nat
    Succ : Nat -> Nat
{% endhighlight %}

Would result in the following context:

{% highlight haskell %}
con Nat  : Type
con Zero : Nat
con Succ : Nat -> Nat
{% endhighlight %}

Where `con` marks the type as being a constructor. Now, if we want to pattern match on a value of type `Nat`, we can look for all the entries marked by `con` which return a `Nat`. 


However, from a type-theoretic perspective, pattern matching is not fundamental, and is really just syntactic sugar around an _eliminator_ for a type. If we pattern match on a `Nat`, we have to cover two cases: the case of `Zero`, and the case of `Succ n`. We can capture this eliminator in the following type (a non-dependently typed eliminator for simplicity):

{% highlight haskell %}
ElimNat : Nat -> a -> (Nat -> a) -> a
{% endhighlight %}

The first argument is the `Nat` to case split on. The second argument is the value to return if the supplied `Nat` is `Zero`. The third argument is a function which computes the value to return if the supplied `Nat` is `Succ n`, where the predecessor `n` is passed to the function.


The idea is that we can deal away with the `con` hackery by additionally adding the eliminator to the context when we process a data type definition.


{% highlight haskell %}
Nat     : Type
Zero    : Nat
Succ    : Nat -> Nat
ElimNat : Nat -> a -> (Nat -> a) -> a
{% endhighlight %}

Unfortunately, we aren't quite done yet. When I was describing what `ElimNat` meant, I described some very specific semantics that we have not told the compiler about yet. For example, the compiler doesn't know that it that two expressions like `ElimNat Zero x f` and `x` are actually equivalent, because we haven't told it so! In a dependently typed language this can be expressed with a few equalities:

{% highlight haskell %}
ElimZero : ElimNat Zero     x f = x
ElimSucc : ElimNat (Succ n) x f = f n
{% endhighlight %}

In total, that gives us the following description of the natural numbers, in context form:



{% highlight haskell %}
Nat      : Type
Zero     : Nat
Succ     : Nat -> Nat
ElimNat  : Nat -> a -> (Nat -> a) -> a
ElimZero : ElimNat Zero     x f = x
ElimSucc : ElimNat (Succ n) x f = f n
{% endhighlight %}


# But what about...

### No eliminators?

Interestingly, defining a type as a context (hereon _contype_) is a generalised version of even GADTs. With all this new power, it can be interesting to think about what different contypes may represent. For example, what about the contype for the natural numbers, but without an eliminator? What would that type even mean?

{% highlight haskell %}
Nat  : Type
Zero : Nat
Succ : Nat -> Nat
{% endhighlight %}

Unfortunately the answer seems to be rather boring. With no way to destruct the type, all values are indistinguishible, and so our contype is just equivalent to the unit type. (I don't know category theory, but I suspect that it would be very useful in analysing these contypes).

### No constructors?


What about a type without any constructors?

{% highlight haskell %}
X  : Type
ElimX : X -> a
{% endhighlight %}

This is the familiar `Void` type!

### Non structural?

ADTs and GADTs _structural_, in the sense that the way they are constructed is symmetric to the way they are destructed. For example, our contype for naturals contains a single destructor that handles case for every constructor. But what if we designed a type where this wasn't the case? Where the way to construct it and destruct it were not necessarily symmetric?


Consider the following definition of sets (which for simplicity, assumes elements of an arbitrary type are comparable with `==`):

{% highlight haskell %}
Set       : Type -> Type
Empty     : Set a
Add       : a -> Set a -> Set a
Elem      : a -> Set a -> Bool
ElemEmpty : Elem x Empty     = False
ElemAdd   : Elem x s         = b
         -> Elem x (Add y s) = (x == y || b)
{% endhighlight %}

We've defined two constructors `Empty` and `Add`, and one eliminator, `Elem`. As a result, we have what looks a lot like the description of an abstract data type. The crucial difference between this contype, and say, a record with those fields, is that a contypes are _closed_. We know that given a `Set a`, these are the _only_ primitive ways to interact with this data type. All other functionality must be defined in terms of these primitives.

### Other stuff?

Honestly I'm sure there's way crazier stuff you could come up with that what I've got here. Another example I've just thought of while I'm writing this sentence: this kind of system has natural support for mutual data types, as there's nothing to stop you intertwining the definitions of multiple types. Let me know if there's any other crazy stuff you could do!

# Is this even useful?

Would a system which defines types in this way ever be useful? Firstly, allowing users to define their types like this necessitates a separate _implementation_ which satisfies the constraints set out in the types (see my previous blog post, _"Types: values vs representation"_). If this were not the case, it would be trivial to define a type like this: 

{% highlight haskell %}
A     : Type
ElimA : A -> a
{% endhighlight %}

Which makes the system logically unsound. By requiring an implementation, we ensure this can never happen since writing the `ElimA` function would be impossible.

One idea is that the implementation could be built purely from fast primitives, allowing for performant pure structures. Though, these ideas are nothing new. A system like this is just a dependently-typed lambda calculus extended with fast primitives, plus an "abstract datatype" system like ML's _abstype_. But it's interesting to think about anyway.

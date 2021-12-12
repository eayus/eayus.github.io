layout: post
title: "Types: values vs representation"
tags: [web, jekyll]
---

# Introduction

On a fundamental level, why is it so much more difficult to implement GADTs efficiently when compared to standard C-style structs? One might think that allowing the user to define their types more accurately would enable compilers to come up with a more specialised implementation. I believe the reason is that C structs and GADTs are not two different approaches to the same problem, but solutions to two different problems entirely.


I believe there are two distinct properties of a type - a definition of the values it contains, and a definition of how to represent those values in memory.

 * Functional languages, specifically dependently typed ones, are excellent at defining a type's values. However, the data type's memory representation is left to be inferred by the compiler, often resulting in inefficient code.

 * Low level languages, notably C, define _only_ a type's memory representation. In my opinion, this is the general reason why it is so difficult to write bug-free C code. A `void*` might describe the layout, but gives us absolutely no information on what values the type contains!

To get the best of both worlds, perhaps we need a language which allows us to specify both these properties.


# The problem, by example


### Natural numbers


Whichever property of types the language choses to define, it typically evolves to include awkward workarounds for defining the other property. For example, in a functional language we might define the Peano naturals as follows:

{% highlight haskell %}
data Nat : Type where
    Zero : Nat
    Succ : Nat -> Nat
{% endhighlight %}

However, any non-trivial program using this raw definition will likely suffer serious performance problems, because the compiler infers the _type representation_ poorly. To work around this, compilers will often have in-built support for this type, which transforms it into a standard machine integer at run-time. `Agda` and `Idris` both do this.

### Optionals

Another example is the `Option`/`Maybe` type:

{% highlight haskell %}
data Maybe : Type -> Type where
    None : Option a
    Some : a -> Option a
{% endhighlight %}

A compiler might represent this in memory as a pair of a tag (describing which constructor has been applied), and a pointer (pointing to the `a` if we're in the `Some` variant). While this reprsentation is certainly not egregious, there's still room for improvement - we could instead represent this as a single pointer which is `null` in the `None` case. `Rust` hardcodes an optimisation of this sort when using an `Option<&T>`.


### Tagged Unions

Of course, the problem exists in the other direction too. Only specifying a data's representation leaves its values ambiguous. Consider the following C definition:


{% highlight c %}
enum tag {
    tag_a = 0,
    tag_b = 1,
};

union val {
    struct a a;
    struct b b;
};

union type {
    enum tag tag;
    union val val;
};
{% endhighlight %}

It is a tagged union, where the tag is used to discern which type is in the union. As a programmer, we understand this link, however the compiler doesn't. The compiler doesn't understand that the value `b` will be undefined if the tag is `tag_a`. In other words, we have done a poor job of describing the _type values_ to the compiler. The common workaround to this issue is to use a macro to generate the "safe accessor" functions which ensure the tag is checked before accessing the union.


# The best of both worlds?

Evidently, using one definition to describe both a type's values and representation always leaves one area lacking. The obvious solution is to define them separately! A definition of a type's values can be seen as the "interface" for the representation to implement. Of course, we might need to attach some extra constraints to ensure that the representation is faithful. Unfortunately, figuring out a nice way to do this is a problem on its own. So that the conclusion to this post is not completely underwhelming, below is a simple example of how something like this might be implemented.


{% highlight haskell %}
data Nat as I32 : Type where

    Zero : Nat
    Zero = 0

    Succ : Nat -> Nat
    Succ n = 1 + n
{% endhighlight %}

In this case, we implemented the naturals as a 32 bit int value, `I32`. `0` and `1` are literals for `I32`, and `+` is a addition on `I32` values.

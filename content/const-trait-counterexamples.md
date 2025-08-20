+++
title = "Const Trait Counterexamples"
date = "2025-08-20"
description = "what led us down this path"
[taxonomies]
tags = ["rust"]
+++

Hi. I'm the lead for Rust's const traits project group. We hope to stabilize const traits soon, but this is a complex feature with huge amounts of design considerations, and we keep getting the same comments from different people who probably have less familiarity with the feature and its design.

That's quite fair because I can't require everyone to have followed every discussion everywhere. I have followed loads of discussions, so this summarizes some of the counterarguments we've had so far.

If you're interested in the language design for how we plan to allow calling trait methods in const contexts, this should be a good complementary resource to the currently open [RFC](https://github.com/rust-lang/rfcs/pull/3762). You might disagree with parts of this post, though. 

## Current proposal summary

Declare a trait as `const` so you can use it in trait bounds:

```rust
const trait PartialEq<Rhs: ?Sized = Self> {
    fn eq(&self, other: &Rhs) -> bool;
    fn ne(&self, other: &Rhs) -> bool { !self.eq(other) }
}

const fn foo<T: ~const PartialEq>(x: &T, y: &T) -> bool {
    x.eq(y)
}
```

Make impls `const` so we can satisfy those bounds:

```rust
pub struct MyType<T>(T);

impl const PartialEq for MyType<()> {
    fn eq(&self, other: &Self) -> bool {
        true
    }
}

impl PartialEq for MyType<u8> {
    fn eq(&self, other: &Self) -> bool {
        self.0 == other.0
    }
}

fn non_const_context() {
    assert!(foo(&MyType(0u8), &MyType(0u8)));
    assert!(foo(&MyType(()), &MyType(())));
}

const CONST_CONTEXT: () =  {
    // assert!(foo(&MyType(0u8), &MyType(0u8))); ERRORS
    assert!(foo(&MyType(()), &MyType(())));
};
```

For generic const code, a `Destruct` trait will be made available (not necessarily included in the first stabilization) to allow the type to be dropped at compile time:

```rust
pub const fn callit<T: ~const Destruct, F: ~const FnOnce(&T)>(x: T, f: F) {
    f(&x)
}
```

`T: Destruct` is true for all `T`, while `T: ~const Destruct` only holds if the compiler knows its destructor can be run in compile time. (See proposal 4 for more on this)

## Meaning of `~const`

> [!Note]
> We use `~const` throughout this document but it is feasible to switch this to any other compatible syntax (like `[const]` or `(const)`). In fact, the nightly Rust as of writing accepts both `T: ~const Trait` and `T: [const] Trait`. I'll stick to `~const` for simplicity.

`~const` is a modifier on trait bounds. (including super traits, e.g. \
`const trait A: ~const B {}`) In general, they only need to be proven if you're using them from const contexts. Hence they're also called "maybe-const" or "const-if-const" bounds.

`~const` bounds on a function are only enforced when you call it. This allows us to instantiate a function in const contexts even if we can't call it:

```rust
pub const INSTANTIATE_BUT_NOT_CALL: fn(&MyType<u8>, &MyType<u8>) -> bool = {
    <MyType<u8> as PartialEq>::eq
};
```

But note that constness isn't a generic parameter and shouldn't be considered instantiated for each call to a `const fn`:<sup>1</sup> If you think about the function `foo` and pass in a `T` that does not implement `const PartialEq`, the constness of `foo` does _not_ change due to the unsatisfied const predicate - `const fn` is "always-const" (and not <nobr>"maybe-const"</nobr>) in a sense that it simply imposes additional constraints if called in const contexts.
{% aside() %} 1: This particular wrong understanding (specifically, constness as a generic effect parameter which is then "tied" to the constness of the trait bounds) runs into a myriad of issues if you think down the line too hard. I have some thoughts about it but it will probably be in a different post. {% end %}

```rust
const A: bool = foo(&MyType(0u8), &MyType(0u8));
```

The call above errors not because `foo` becomes non-const when `T = MyType<u8>` - it errors because `MyType<u8>: const PartialEq` isn't satisfied.

With the current proposal and model, we now explore a few alternatives that Rust team members have thought through, and I will explain why they can't really work.

## proposal 1: complicity in implicity

_Why not make `~const` the default and use an opt-out for non-const bounds?_

It is expected that people will write `~const` bounds a lot. What if we made `T: Trait` implicitly const-when-const if it's inside a const item? i.e. making the following work:

```rust
const fn foo<T: Add>(x: T, y: T) -> T::Output {
    x + y
}
```

Users of non-const trait bounds in `const fn` will now use an opt-out syntax such as `?const`:<sup>2</sup>
{% aside() %} 2: note that the original `LazyCell` source code has the trait bound on the `impl`. This part is only for illustrating the opt-out bound so we don't get into the nitty gritty yet.  {% end %}

```rust
impl<T, F> LazyCell<T, F> {
    pub const fn new(f: F) -> LazyCell<T, F> where F: ?const FnOnce() -> T {
        LazyCell { state: UnsafeCell::new(State::Uninit(f)) }
    }
}
```

Why can't this work? The very first drawback this proposal faces is that these things are already possible on stable today:

```rust
trait Trait {}
impl Trait for () {}

fn something() {}

const fn foo1(x: &dyn Trait) {}
const fn foo2(x: &impl Trait) {}
const fn foo3<T: Trait>(x: &T) {}
const fn foo4(x: fn()) {}
const X: fn() = something;
const A: &'static dyn Trait = &();
```

If we ever wanted to allow `impl Trait`/`dyn Trait`/`fn()` to be callable in const contexts, then they should follow the same opt-out, necessitating the following change:

```rust
trait Trait {}
impl Trait for () {}

fn something() {}

const fn foo1(x: &dyn ?const Trait) {}
const fn foo2(x: &impl ?const Trait) {}
const fn foo3<T: ?const Trait>(x: &T) {}
const fn foo4(x: ?const fn()) {}
const X: ?const fn() = something;
const A: &'static dyn ?const Trait = &();
```

This then runs into multiple problems.

### 1A: editioning everything

We already allow trait bounds that mean non-const in `const fn`. This means if we were to change `T: Trait` in `const fn` to what we currently mean by `T: ~const Trait`, we have to do it over an edition since it is a breaking change.

But of course we can't make these changes immediately. Suppose we're currently in the 2024 edition and we accept this plan to migrate everyone to use `?const` where necessary. We can accept `?const` in any edition as it is equivalent to `T: Trait` now. Now in 2026 edition we deprecate non-`?const` bounds, we emit a warning everytime someone writes `T: Trait` but means `T: ?const Trait`. In 2028 edition `T: Trait` now means `T: ~const Trait` (implicit behavior). We need to do this change over two editions because just doing it in one edition is even more problematic.

Suppose `FnOnce` becomes a const trait. This poses issues for crates in edition 2024 or below, who have written this:

```rust
pub struct MyWrapper<F>(F);

impl<F> MyWrapper<F> {
    const fn new(f: F) -> Self where F: FnOnce() {
        MyWrapper(f)
    }
}
```

It is dangerous for crates having non-const bounds that never went through the intermediate 2026 edition to directly migrate to edition 2028, as edition migration processes primarily rely on things compilers can catch. There are things that `cargo fix --edition` can't catch, and there are people who upgrade editions by simply bumping the number and fixing the issues that arise with bumping (how can you fault them? I personally never thought editions could significantly change behavior and meaning of syntax).

This is a major meaning change (perhaps bigger than [disjoint closure capture](https://doc.rust-lang.org/edition-guide/rust-2021/disjoint-capture-in-closures.html) and [if-let rescope](https://doc.rust-lang.org/edition-guide/rust-2024/temporary-if-let-scope.html) which won't break/change too much) because there are tons of `const fn` with trait bounds that all currently mean non-const.<sup>3</sup> Their migration path to a potential 2028 edition is very scary to me.
{% aside() %} 3: See [this search](https://github.com/search?q=%2Fconst+fn+%5Cw%2B%3C%5Cw%2B%3A%2F++language%3ARust+NOT+is%3Afork&type=code), and to a lesser extent, [this search](https://github.com/search?q=%2Fconst+fn.*where+%5Cw%2B%3A%2F++language%3ARust+NOT+is%3Afork&type=code) {% end %}

### 1B: non const traits and implicit bounds

```rust
trait NonConstTrait {}

const fn foo<T: NonConstTrait>() {}
```

On the outset, there appears to be three choices:
1. Compile this, with `T: NonConstTrait` conditionally const
2. Compile this, but `T: NonConstTrait` doesn't become conditionally const (not `~const NonConstTrait`)
3. Make this a compile error.

#1 can't really work due to breaking change complications when methods change their bounds to `~const` (see 2A),
#2 can't work either because it would make it a breaking change for someone to change their trait into a const trait (the `T: NonConstTrait` becomes stricter if `NonConstTrait` betrays its name and becomes `const`)

So the only viable choice is #3. However, this has the ability to confuse many people, specifically with 1C and 1D.

### 1C: impl confusion

Should `impl` blocks also be a part of this elision? That is, should the following have a <nobr>non-const</nobr> `T: Trait` bound applied to `foo` but `~const` bound applied to `bar`?

```rust
impl<T: Trait> MyType<T> {
    fn foo(self, x: T) {}
    const fn bar(self, x: T) {}
}
```

If yes, this can be super confusing. 1B will turn this into an error on `bar` if `Trait` is not a const trait. It's also a candidate for accidentally stricter bounds than necessary (when will users know when to use `?const`? cc 1D)

If not, it can also be super confusing. The rules for when `T: Trait` implicitly means `T: ~const Trait` would be complicated, hard to teach, and slightly inconsistent, given that something like the following _should_ have implicit `~const` at the impl-level.

```rust
impl<T: Trait> const Trait for MyType<T> {}
```

All of this is mostly because of the weird double meaning of `T: Trait` (on non-const contexts, equivalent to `T: ?const Trait` in const contexts, but `~const` in <nobr>const-contexts</nobr>)

An explicit opt-in scheme, with `~const` bounds at the `impl` level, if allowed, would not have the same issues, as it is clear what the user intended, and it is clear that it would apply to `const fn`s.

### 1D: usage path

Proposal 1 will make writing stricter bounds the default, as `T: Trait` in `const fn` is stricter than `T: ?const Trait`. It is only with non-const traits (1B) where a user will be prompted to use `?const`.

This is a broad assumption that users will most of the times want `~const`, similar to `Sized` vs. `?Sized`, but probably less correct than the latter. There are many constructors with trait bounds (e.g. `LazyLock` with `F: FnOnce`) that don't need to call methods at compile time. They can be
intended for storage (so the stored types' methods can later be called at runtime), or just using traits' associated consts.

On the other hand, `~const` as opt-in would be required only when someone attempts to call a trait's methods (easily suggestable by compiler errors), which would leave people more naturally using `T: Trait` to mean non-const if they don't need `~const`. 

### 1E: virality

Remember the snippet at the beginning of this proposal which contained function pointers and dyn traits, and impl traits? Well.. about those.

Suppose in the future we might want to allow `dyn ~const Trait` or `~const fn()` pointers. Notwithstanding the potential complexity for the compiler to support these, but let's say `struct A(pub ~const fn())` is possible, and it means that the field must be a function pointer callable at compile time if `A` is being constructed at compile time. `dyn ~const Trait` operates similarly.

This then means an implicit `~const` version now has to either (1) affect all types containing `fn()` pointers and `dyn Trait`s by making them imply `~const` or (2) create an opt-in syntax specifically for these things.

(1) is self-evidently problematic; (2) feels extremely inconsistent, why use opt-in in some places but opt-out in others?

## proposal 2: selectiveness

_Why have const apply to the entire trait?_

This has many layers to unwrap: why do we have to mark the trait at all? Can we have <nobr>per-method</nobr> choices? Can we do refinement with bounds on specific methods?

We'll answer these in this section, but we'll start with the most general fact: If we want `impl`s to be `const` or non-const, there must be a way to distinguish a trait that allows const implementations and a trait that does not. And all current traits (before const traits stabilize) must not allow const implementations.

### 2A: Which Gender Is Your Trait

Let's look at [`Iterator::sum`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.sum):

```rust
trait Iterator {
    type Item;
    fn sum<S>(self) -> S
    where
        Self: Sized,
        S: Sum<Self::Item>,
    {
        Sum::sum(self)
    }
}
```

For any generic parameter `T` where `T: ~const Iterator`, we can't really call `T::sum::<usize>`. Right now the bound on `S` is `S: Sum<Self::Item>` but it should really be `S: ~const Sum<Self::Item>`. If we allowed calling `T::sum::<usize>`, and the bound later turns `~const`, it would break if `usize` remains having a non-const `Sum` impl. So to avoid breaking changes we have two choices:

1. Allow `T: ~const Trait` on non-const traits but can't call any of the methods
2. Disallow `T: ~const Trait` on non-const traits.

Our current design uses #2, as #1 feels quite counterintuitive: the idea of `~const` bounds is to allow calling methods on them, so it isn't really useful. It would also prevent any actual uses of generic items with `~const` bounds on non-const traits, as `const` `impl`s cannot be written without the trait being `const`. With #2, we're able to give trait authors _one_ chance to make sure their bounds are either non-const or `~const` as they see fit. As once the trait is `const`, turning existing non-const bounds into `~const` would be a breaking chnage.

In any case, the compiler must know what is a `const trait` and what is not.

### 2B: the constness divide

The alternative here would be to allow `~const Tr` bounds on `Tr` without it being marked a const trait, but not allow calls to methods until the methods are marked const in some way. (i.e. per-method constness)

```rust
trait A {
    ~const fn foo() {}
    fn bar() {}
}

const fn uwu<T: ~const A>() {
    T::foo(); // can be used
    // T::bar(); errors
}
```

This has its own caveats.

First, this means there are never-const methods in a trait. Some methods allow callers in const contexts and require const implementations while some do not. I don't think that's really useful. If a user writes `T: ~const Trait`, they'd normally expect every method in `Trait` to now be const-callable. It's quite rare for a trait to be designed in a way that allows some methods to be callable in const contexts while others not. (at least, no one has ever provided a concrete example) Those use cases would be covered by separating them into two traits anyways.

Second, we _still_ need a way to figure out whether a trait is `const` to decide whether to allow `const` `impl`s. We can't allow `const` `impl`s for non-const traits, to allow existing non-const traits to transition their methods into const.

But that means once `A` makes `foo` `~const` and publishes a new crate version, they've lost their ability to make `bar` `~const` in the future, as downstream crates would happily do this:

```rust
impl const A for MyType {
    const fn foo() {}
    fn bar() { println!("some non-const operation") }
}
```

That is super awkward to both teach and learn. This awkwardness was even more extensively discussed in my [HackMD doc](https://hackmd.io/@beef/rJ1We7aCkl) from four months ago. 

### 2C: per-method bounds

Okay, so, it's really awkward for fine-grained per-method constness to co-exist with whole-trait constness (necessary for trait bounds as well as whether to allow `const` `impl`s). What if we dealt away with whole-trait constness entirely?

Consider `~const` bounds on methods. Some scheme like:

```rust
const fn foo<T: PartialEq>(x: &T, y: &T) -> bool where T::eq: ~const {
    x.eq(y)
}
```

Where `impl`s can't be wholly `const` or non-const, but individual methods can become `const` on their own:

```rust
trait B {
    fn owo();
}

impl B for () {
     const fn owo() {} // now possible
}
```

But that proposal has a _lot_ of downsides. 

Because all methods' signatures are now lying to you. Consider the common case of some impl using a generic type's methods:

```rust
trait C {
    fn some_fun_method();
}

impl<T: C> C for Option<T> {
    const fn some_fun_method() /* where T::some_fun_method: ~const */ {
        // T::some_fun_method()
    }
}
```

in `Option<T>::some_fun_method`, we have no idea whether we can actually call `T::some_fun_method`, without writing a bound. Making this bound apply to the entire trait impl seems awful to require (defeats the entire point of an individual method being `const` and writing `~const` bounds on individual methods), so then this per-method bound now applies to `some_fun_method`. That also means the requirements for a particular method to be `const` may be stricter than what is written on the trait method signature. (`C::some_fun_method` has no additional restrictions whereas `<Option<T> as C>::some_fun_method` does). 

This is pretty much not avoidable, as traits upstream cannot know what methods downstream impls want to call (in this case, `T` is not even nameable at the upstream trait `C`)

Therefore, all const fns that attempt to call some trait methods on a generic parameter must add a bound on that specific method.

This is particularly frustrating for `Iterator`, which has loads of extension methods (that can all be overriden to do something non-const, under this scheme), and bounds are necessary for all methods being called, resulting in the following for a function that should have been super simple:

```rust
const fn sum_double<I>(i: I) -> u32
where
    I: IntoIterator<Item = u32>,
    I::into_iter: ~const,
    <I::IntoIter as Iterator>::map: ~const,
    <Map<I::IntoIter, const fn(u32) -> u32>>::sum::<u32>: ~const,
{
    const fn double(x: u32) -> u32 { x*2 }
    i.into_iter().map(double).sum()
}
```

This proposal might also need special annotations on trait methods that provide a `const` default method body but otherwise doesn't require downstream `impl`s to be `const` (otherwise you wouldn't know which default method bodies are callable from `const`). That adds an additional layer of syntax complexity which needs to be designed.

There was also a bit of Zulip discussion on this. See [this topic](https://rust-lang.zulipchat.com/#narrow/channel/328082-t-lang.2Feffects/topic/Is.20const.20a.20trait.20modifier.20at.20all.3F) and its discussion, mainly before May 8th.

## proposal 3: isn't it just `const`?

_Why not use `T: const Trait` for const-when-const bounds?_

This is one of the proposals which actually has _some_ merit. The idea is to use `T: const Trait` for the "const-if-const" syntax, while thinking about the future with `const(always)` or `=const` to represent always-const bounds.

Because it turns out we actually do need always-const bounds, for the trait bound to be used in an assoc const-item `const A: () = ();`, in a const block `const { 1 + 2 }`, or in const generic arguments `[T; { 1 + 2 }]`. Those could become usage sites that require a stricter bound than `~const`, so we must think about reserving the "always-const" bound for them.

My main reservation with this proposal is that it sort of justifies a change of const items, const blocks into using the same new syntax for always-const bounds. so `const(always) X: i32 = 42;` and `const(always) { 2 * 3 * 7 }` instead of what we have now which reserves `T: const Trait` for always-const to keep the consistency. But we always have _some_ form of (small, potentially acceptable) inconsistency one way or the other (such as `~const` in `const fn`, see proposal 6), so it might end up being an option we end up choosing.

## proposal 4: destructive interference

_Why `Destruct`? Why not just `T: ~const Drop`?_

We are in a special place because some destructors are non-const. We already allow <nobr>non-const</nobr> `impl Drop for MyType`s, so there needs to be a way to distinguish types that can be dropped in compile time from types that cannot.

This leads to us to a new marker trait called `Destruct`, which is automatically implemented. Non-const `Destruct` holds for all types, but `const Destruct` only holds if the type's `Drop` impl (if it exists) is const and the type's components all implement `const Destruct`.

Therefore, you must sprinkle your functions with `T: ~const Destruct` bounds when you're constifying them, if generic parameters need to be dropped at any point in the function.

Why not `T: ~const Drop`? Well it makes a huge asymmetry with `T: Drop` bounds, as the latter would only hold if `T` has a manual `Drop` impl. It would also make `T: const Drop` not imply `T: Drop`, which has weird language-level implications as well as compiler-level implications. (we have run into trait bound cache issues with this in the past<sup>4</sup>)
{% aside() %} 4: it _might_ be fine to desugar `T: ~const Drop` into `T: ~const Destruct` while keeping `Destruct` hidden. I still don't think that's a good option because of the asymmetry. {% end %}

`T: Drop` is also a valid bound even though you might not have seen this. `pin-project` [uses](https://github.com/taiki-e/pin-project/blob/b8e1b1640fa0edd0b27da9e084bec827eb927f2d/pin-project-internal/src/lib.rs#L107-L132) this bound to prevent any user from writing their own custom destructors on their types. So no matter what scheme we end up choosing, we _must_ have at least two traits. One to represent the ability to be destructed, one to represent whether or not a type has a custom destructor. Otherwise we run into asymmetry between `T: const Drop` vs `T: Drop`.

## proposal 4.5: destructive interference, part 2

_Can we make `T: ~const Destruct` implicit?_

This is hard to say, depending on what is meant by "implicit". Inferring whether requiring `~const Destruct` for generic types based on whether the code has a possibility to drop the type is no-go, because changing the type signature/trait bounds based on the body has loads of bad semver complications and is generally not possible in the compiler architecture.

Making `~const Destruct` implicit for all generic params is not good, either. Many generic API interfaces want to assume as little about their types as possible, so a broad scheme like this necessitates an opt-out syntax which seems too odd to have/hard to design.

A more limited plan might be to infer a `: ~const Destruct` super trait if some methods on a trait take `self` by-value, that also has issues because the following example means shouldn't have `const trait Add: ~const Destruct` inferred:

```rust
#![feature(const_ops)]
#![feature(const_trait_impl)]
#![feature(const_precise_live_drops)]

use std::ops::Add;

pub struct Wrapper<T>(T);

impl<T: ~const Add> const Add<Wrapper<T>> for Wrapper<T> {
    type Output = Wrapper<T::Output>;
    fn add(self, other: Self) -> Self::Output {
        Wrapper(self.0 + other.0)
    }
}
```

One possible plan (will likely be our actual plan) is to lint const traits that take `self` by-value and recommend adding a `~const Destruct` super trait. That way it will save crate users relying on that trait to not have to add `~const Destruct` bounds everywhere, at the same time not assuming everyone wants this.

## proposal 5: academic zealotry

_Let's follow the approach used in academic paper X.._

This section was adapted from a [similar comment I made on Zulip](https://rust-lang.zulipchat.com/#narrow/channel/328082-t-lang.2Feffects/topic/On.20.60.28const.29.60.20bounds/near/516019998) in May.
 
Academic research is exciting work. Language design in Rust is different (maybe slightly boring?) because it's mostly a weighing of the pros and cons of every possible alternatives and finding ways to practically make the language more capable.

The work on const traits in Rust is often linked to work on effects in other (academic) programming languages, results published via research papers, etc.

We then often see an excitement to apply those academic thinking models to Rust.

Those thinking models often appear super convoluted to me. It sometimes looks like practicality has been dismissed in favor of generality. That's fine, but I don't think they are very compatible with Rust.

I think it would be fair to consider the role of formal modeling of effects as a different programming language and how programmers using that programming language model their functions and their ability to be called in different contexts. (i.e. runtime vs. compile time)

The academic way of thinking about effects/const traits holds the same amount of weight as some other programming language's way of thinking about effects/const traits to me. It's nice to try to incorporate the bits that would work for us, but incompatible things are incompatible. Forcing Rust's model to be losslessly transferable to a formal model is equivalent to forcing Rust's model to be losslessly transferable to a different programming language, say Zig. We can't assume that ideas about programming languages suddenly become applicable to all programming languages just because those ideas are published to a peer-reviewed journal. Our RFCs are always peer-reviewed, too.

It is cool to think about effects or capabilities and how we can encode them as modifiers to entities in the type system, be it traits, trait bounds, functions, impls, etc. We're not in the best place to unify them because `const` is the inverse of an effect (it prevents you from calling non-const items) while `async` is an actual effect (it allows you to call other `async` items). At the same time `const` seems like it wants to apply to the whole trait, while partial maybe-`async` traits seem very desirable.

If we wanted to unify them to make our version of effects closer to formal modeling, we must do so for everything that we currently have, not just const traits. So we should also think about whether it is really *feasible* to do so in the first place, and then decide whether or not we can proceed with a stabilization of const traits with the pieces that make the most sense for the language *right now*, without having to make our effects story consistent first.

## proposal 6: drowning in conditionals

_We should have `~const fn` and `~const trait` and `impl ~const`, etc._

The idea here is that `~const` is "conditionally const", i.e. may or may not require const impls depending on whether called from a const context. Given that `const fn` is also "maybe-const" (i.e. could be called from both non-const and const contexts), we should make them also use `~const`, like `~const fn` for consistency and also distinguish with `const X: () = ();` items.
 
My biggest feeling here is that it changes up everything for no gain. I think rarely anyone benefits from this. It might feel consistent but I'd rather not let us be consistent for consistent's sake.

But still, there are other consistency arguments that contradict this. `const fn`, `const trait`, and `impl const`, and const items/const blocks all restrict their bodies to operations performable in compile time, i.e. they must all call `const fn`s. In terms of restriction they work mostly the same except for which trait's methods they can call (in `const fn` you can call methods from `T: ~const Tr` but in `const` items you can't)

Also, it makes little sense to have an "always-const" fn such that it cannot be called in <nobr>non-const</nobr> fns. But that's what `~const fn` seems to imply exists.

The other idea here relates to the "meaning of `~const`" section. The constness of the `fn` never changes. It should always be evaluatable at compile time. But when you pass in some `T` into `const fn foo<T: ~const Tr>` that doesn't `impl const Tr`, and try to call it from compile time, it's not that `foo` now is non-const (`foo` is fully prepared to be evaluated in compile time) but that you aren't satisfying `foo`'s constraints.

## proposal 7: questionable conditions

_`?const` is okay for "maybe-const".. right?_

We have received proposals like this because `~const` introduces a new sigil, and the alternatives don't seem to be as good. Why don't we use `T: ?const Trait` to mean `T: ~const Trait`?

The main reason here is that `?` has an existing meaning in trait-bound adjacent areas, which is to _relax_ a bound, or to _disable_ a default, such as `?Sized`. `T: ~const Trait` is a stricter bound than `T: Trait`, so using `?` for it isn't really nice. This is also the reason proposal 1 (and the ancient implementation before it got switched due to issues aforementioned) uses `?const` for opt-out.

Using `?const` for opt-in, on the other hand, isn't a good syntax proposal.

## let's try picking wavelengths for these sheds..

These are things we should actually try to form consensus on before stabilizing:

* Syntax of the const-when-const bounds. There's `~const` and `[const]` and `(const)`
    * We can figure this out along with possibility of proposal 3 (`~const` => `const`, `const` => `=const` or other) on the table. 
* Order of keywords - whether to use `const impl Trait for Ty` or `impl const Trait for Ty`
    * Had some [discussion on the RFC PR](https://github.com/rust-lang/rfcs/pull/3762#issuecomment-3172115089) recently.
* Naming of `Destruct`
    * Not really discussed anywhere, but we might want to find a better name for `Destruct` if we want to stabilize it. `Droppable`? `Destroy`?

## Save this for the future

These are some features that are _not_ essential for const traits. Including them will unnecessarily enlarge the scope
of the RFC. But it might still be useful to propose them later.

* `const fn` trait methods - where downstream has to implement as `const fn` no matter whether `impl` is const or not.
* `~const fn` pointers, `dyn ~const Trait`, `impl ~const Trait`
* Figuring out something for the "really const" distinction (related to proposal 6), if that is really worth it - what
we should do with `const {}` blocks and `const X: Ty = ...` items.
* Figuring out a way to configure derives (built-in or custom) to generate const implementations.
    * Although we might still want to recommend a way for custom derives to start generating const impls.

## So, what's next?

I started my work on const traits on [July 1st, 2021](https://github.com/rust-lang/rust/pull/86750), making it so that
const trait impls can be called across different crates. I've worked on the implementation of this feature since then,
and now it's been four years.

We might still have many years to go, but hopefully this post helps making the language design discourse better.

If you would like to get involved, feel free to go comment on the [open RFC] (please comment on specific lines or on
the file using the PR review feature, this makes discussion threaded and much easier to follow), or follow discussions
occuring on the [#t-lang/effects](https://rust-lang.zulipchat.com/#narrow/channel/328082-t-lang.2Feffects) Zulip channel.

I suppose you can also consider supporting me on [GitHub Sponsors](https://github.com/sponsors/fee1-dead) :D

[open RFC]: https://github.com/rust-lang/rfcs/pull/3762

## Acknowledgements

* Thanks [errs] for the encouragement to write this post, and [oli] for commenting on initial drafts of the proposal and providing more thinking on Proposal 4 and 4.5.

[errs]: https://github.com/compiler-errors
[oli]: https://github.com/oli-obk

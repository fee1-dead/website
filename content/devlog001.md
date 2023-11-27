+++
title = "Devlog 001"
paginate_by = 5
date = "2023-11-27"
description = ".. What's the point of them, anyway?"
[taxonomies]
tags = ["rust", "devlog"]
+++


I've been thinking of starting a devlog for a long time. There are many reasons why I couldn't just start it previously:

1. Lack of time. Do I really have time to do this stuff? Maybe I do, but am I going to spend too much time trying to get everything to look good and exactly as I wanted it on first try? Historically, that answer has been yes. A few days ago, I realized that that's not good, so instead of spending time on the looks I am just going to write some stuff.
2. Perfectionism. Old me wanted a perfectly built place that matched what I wanted, but I have changed. Now I just want a place to write some ideas and updates.
3. Focus. I mean, blog posts that actually get read by other people have a clear focus in what they want to write about, and I usually don't. Do I really need that for a blog? Not really.

Hopefully, by the time someone reads the above three points, they will also know that I ramble about things that don't really interest them. I guess that's fine, though, since maybe this can be a place where I write for myself and not for others. This is my first post ever, so there's time and space to explore things!

Okay here's some exciting technical stuff:

### Working with effects in Rust

In the Rust compiler, we're working on [an initiative](https://blog.rust-lang.org/inside-rust/2022/07/27/keyword-generics.html) to make an internal refactoring while not changing the user-facing aspects of [a feature](https://github.com/rust-lang/rust/issues/67792) named `const_trait_impl`. The old implementation simply made the feature a special case, treating it in many different ways in many different places, and had many issues that I won't go into in this dev log. (maybe for a separate blog post though!) But here are some progress and challenges recently:

#### Renabling effects in `core`

As a step towards a more mature feature, we previously [enabled](https://github.com/rust-lang/rust/pull/114776) effects in `core`, which made `const fn` and methods desugar into generic, depending on whether the function is called from a const context, like so:

```rust
const fn a() {}
// turns to..
fn a<const host: bool = true>() {}
```

Here, the `host` flag is set to `false` if the function is called in compile time, while it is set to `true` if called from a context that runs in runtime. This desugaring _shouldn't_ have any effects on anyone using the standard library or anything within the standard library. But we [disabled](https://github.com/rust-lang/rust/pull/116856) `effects` after finding out that actually, you can just call `a::<true>()` when this desugaring is enabled, breaking the assumption that this feature is invisible to users. This was [fixed](https://github.com/rust-lang/rust/pull/117171) shortly later, while a similar fallout that caused rustdoc to display these desugared parameters in the generated documentation was fixed in [two](https://github.com/rust-lang/rust/pull/116670) [PRs](https://github.com/rust-lang/rust/pull/117531) by oli-obk and fmease. After these fixes were merged, I went ahead and [reenabled](https://github.com/rust-lang/rust/pull/117825) effects.

#### Effects desugaring with default type params

The `PartialEq` trait is defined like this:

```rust
pub trait PartialEq<Rhs: ?Sized = Self> {
    /* stuff */
}
```

And if we had the effects desugaring, it would be:

```rust
pub trait PartialEq<Rhs: ?Sized = Self, const host: bool = true> {
    /* stuff */
}
```

And here we have a problem, since if someone tried to implement `const PartialEq`, their impl could look like:

```rust
impl const PartialEq for () {}
// desugars to..
impl<const host: bool> PartialEq<host> for () {}
```

But this (currently) generates a mismatch error since a const param `host` is supplied to a type param `Rhs`! To fix this, I have two ideas in mind:

1. Give the desugaring of `impl const` and `: ~const` bounds special treatment, such that the parameters it desugar into must be dealt with specifically as opposed to the normal params supplied (this might involve changing the internal HIR representation of generic args)
2. Give the params special treatment, but only use some sort of an internal marker to distinguish desugared host bounds. We'd change the logic somewhere when handling those params and fit the desugared param into its place correctly.

I'll say they're quite similar and we might have to end up doing both. We'll see!

#### Checking all method calls for effects

Somewhere in the code for Rust's type checker, there's a function that _enforces effects_. This means that we try to equate effects to what we expect them to be. Here's an example to illustrate:

```rust
// all desugared, since you probably know what it means
fn some_const_fn<const host: bool = true>() {}

fn a_non_const_fn() {
    some_const_fn::<?>();
}

fn another_const_fn<const host: bool = true>() {
    some_const_fn::<?>();
}

const CONST_ITEM: () = {
    some_const_fn::<?>();
};
```

Before typeck (the process that we call type-checking), effect parameters are unknown (marked with a question mark), and the `enforce_context_effects` comes in and fills them in. These examples are all different:

```rust
// all desugared, since you probably know what it means
fn some_const_fn<const host: bool = true>() {}

fn a_non_const_fn() {
    // not a const-context, so fill in `true`.
    some_const_fn::<true>();
}

fn another_const_fn<const host: bool = true>() {
    // in a const-fn context (could be called in runtime _or_ compile time),
    // fill in the host param that this function has.
    some_const_fn::<host>();
}

const CONST_ITEM: () = {
    // in an always-const context (must be evaluated at compile time),
    // fill in `false`.
    some_const_fn::<false>();
};
```

While trying to make `PartialEq` work as a `const_trait`, I noticed that method calls desugared by calling `a == b` don't get checked. [This PR](https://github.com/rust-lang/rust/pull/118282) fixed that. Another step towards enabling effects by default for all crates!


### Where are the other sections?

I didn't have time so I'll end here. I do want to summarize the state of some personal projects that I have been working on for the past few years in a future devlog, but I also want to get this post _done_, so maybe in the next one!

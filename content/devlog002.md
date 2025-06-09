+++
title = "Devlog 002"
date = "2025-06-09"
description = "I promise it's cool stuff"
[taxonomies]
tags = ["rust", "wikimedia", "devlog", "dev"]
+++

Hi y'all! It's been [a while](../devlog001/) since I've done a devlog, and I figured it's time to share a bunch of things
I have worked on, just to celebrate what I've done and take some time to reflect on what I could do next.

Let's get into it!

## Rust related stuff

### Const Traits

I've been working on this feature for almost [four years](https://github.com/rust-lang/rust/pull/86750) now, and the
feature is only recently getting some momentum towards finally being accepted. [Oli] opened a [new RFC] which drew a lot
of commentary and the lang team seems ready to accept, though only after we figure out how the syntax works. There's a
lot of design work, which includes debating the syntax and finding reasons why one color for the shed works better than
the other, which is happening at the [#t-lang/effects Zulip channel]. I tried to articulate reasons for the colors I
prefer, and I was moderately successful in convincing the lang team into adopting some of the choice.

I'm actually hoping that const traits can get stabilized this year, given the amount of time and effort everyone is
putting in to polish the language design into completion. I'm glad that I was able to help (even though [errs] did a
really fantastic job of figuring out the compiler architecture to make the thing robust and easy to develop) on the
compiler side to push things to a stable and well-tested state.

[Oli]: https://github.com/oli-obk
[new RFC]: https://github.com/rust-lang/rfcs/pull/3762
[#t-lang/effects Zulip channel]: https://rust-lang.zulipchat.com/#narrow/channel/328082-t-lang.2Feffects
[errs]: https://github.com/compiler-errors

### Frontmatters

[This](https://github.com/rust-lang/rust/pull/140035) was a particularly cool PR because I think I like working on the
compiler frontend. The backstory to that, though, was that I gave a lot of pedantic reviews in the original PR and made
[epage], the original author who drafted an implementation, quite annoyed at me. But at the same time, I wanted to
ensure that the compiler has a solid quality code to work off of, especially when it comes to initial implementations of
a feature. So I offered to implement it. It wasn't too bad because it wasn't my first time working on the lexer/parser
part of the frontend, with my experience implementing [c-str literals]. Mapping out all the possible error cases and how
best to handle them did take some time though. It was a nice learning experience nonetheless.

[epage]: https://github.com/epage
[c-str literals]: https://github.com/rust-lang/rust/pull/113476

### Reviews

I really like spending time to review PRs. At time of writing, I have 272 [reviewed PRs] that have been merged into
rust-lang/rust. I'm also sometimes really nitpicky when it comes to reviews. I think it is good to point out things that
the other person may have not noticed instead of silently accepting, and it is also good to hear from different
perspectives whether I am a reviewer or a PR author. I want to do more reviews and if I get bored I might steal some.

[reviewed PRs]: https://github.com/rust-lang/rust/pulls?q=is%3Apr+assignee%3Afee1-dead+is%3Amerged+

### AST visitor stuff

I'm trying to deduplicate the AST `Visitor` and `MutVisitor` functions in rustc. The work is currently tracked at
[#127615], and it's mostly an exercise for me to get back at developing rustc in my free time. I've done some really
clever stuff in the mean time:

`Visitor` is allowed to return a `ControlFlow` while visiting stuff, this is controlled by the implementer's choice when
`impl`ing `Visitor` and setting the `Result` associated type to `ControlFlow` instead of the `()` by default.

`MutVisitor` does not allow returning `ControlFlow`s and only allows returning `()`s. This creates some hassle while
writing the macro that emits functions for both visitors. My first solution to this is to use something like the
following:

```rust
fn visit_id<$($lt,)? V: $Visitor$(<$lt>)?>(vis: &mut V, id: &$($lt)? $($mut)? NodeId) $(-> <V as Visitor<$lt>>::Result)? {
    /* implementation goes here */
}
```

Although trivially copyable, it has a lot of characters and served as an extra step when converting from the normal
`V::Result`. I then decided to use a sealed super trait trick in my [latest PR]:

```rust
mod sealed {
    use rustc_ast_ir::visit::VisitorResult;

    /// This is for compatibility with the regular `Visitor`.
    pub trait MutVisitorResult {
        type Result: VisitorResult;
    }

    impl<T> MutVisitorResult for T {
        type Result = ();
    }
}

use sealed::MutVisitorResult;

pub trait MutVisitor: Sized + MutVisitorResult<Result = ()> {
    /* methods go here */
}
```

Then, we can use `V::Result` however we like, _and_ it is a drop-in replacement because we _know_ that it is always
going to be `()`. Really proud of myself for this one.

[#127615]: https://github.com/rust-lang/rust/issues/127615
[latest PR]: https://github.com/rust-lang/rust/pull/142240

## Wikimedia related stuff

### WikiAuthBot-ng

[This](https://github.com/fee1-dead/wikiauthbot-ng) is a bot that uses OAuth to connect users on Discord to their
identity on Wikipedia. Some Wikipedia-related servers require people to authenticate to talk, while others want to use
it to allow easy querying of identity from Discord.

The original version (WikiAuthBot) was written in Python and had a lot of bugs. I rewrote it in Rust and added a lot of
additional functionalities. Working with databases stood out particularly in my process in making this a completed,
stable project. I first went with using the provided MariaDB instance on [Toolforge], then I thought it was too slow and
switched to using Redis as a persistent storage. Then I thought it was too slow again and I tried SQLite. But Toolforge
uses NFS for all hosted projects and SQLite is known to not work as optimally via NFS. I then went back to MariaDB and
it got fast again. All of these involved writing small Rust code that translated data from one place to the other, which
was quite some hassle.

In the end though it worked out.

[Toolforge]: https://wikitech.wikimedia.org/wiki/Help:Toolforge

### DeadbeefBot

[This](https://github.com/fee1-dead/deadbeefbot) was yet another project. It's tasks can be found on its
[Wikipedia page], but it essentially helps maintain some formatting of Wikipedia pages. The code was not too hard and I
worked on its design quite some time already, but it took some procrastinating for me to actually get this registered as
a cron job. But I got there in the end.

[Wikipedia page]: https://en.wikipedia.org/wiki/User:DeadbeefBot

### `w`

Yeah, it's [another MediaWiki API library](https://github.com/fee1-dead/w). I originally wanted to use the name `mw`
seeing that it wasn't used at the time, but it got taken by a [different](https://crates.io/crates/mw) crate. I talked
to the person who took that name and they refused to give it to me. Luckily the even shorter name `w` was available, so
yay!

`w` wants to be minimal because it doesn't want to make assumptions about how one should use it. My previous library was
a failure as I was too ambitious in wanting to represent all possible API combinations into types (turns out it's so
huge and messy, and in the end I would've only done the request part only anyways). So I decided to push that onto the
users. Users should decide how to serialize their requests and deserialize their responses.

[`mwapi`] was a similar library, but my submissiveness to capitalism pushed me towards wanting a less restrictive
license than GPL, and also because I can't easily [work on multiple wiki instances at the same time]. The latter feature
was really needed for WikiAuthBot-ng as it needs to work with Wikipedia servers in many different languages (so
different API instances).

I hope I can polish it so that it is more ready for general use. I am satisfied about it enough to use it in my new
projects and I will add to it as I find missing features while using.

[`mwapi`]: https://gitlab.wikimedia.org/repos/mwbot-rs/mwbot/-/tree/main/mwapi
[work on multiple wiki instances at the same time]: https://phabricator.wikimedia.org/T389030

### Script syncing bot

[This](https://github.com/fee1-dead/usync) was my solution to a long-running problem on Wikipedia: How can user scripts
be deployed from GitHub to Wikipedia, as users try to load from the script hosted _on_ Wikipedia? Yeah I know this might
sound silly but many times it involved primitive copy-pasting locally to update the Wikipedia page.

This bot is still going through the community processes, but I'm hoping to deploy it soon :D

## Lingering ideas

Oh I love some of these ideas.

### Non-GNU Linux from Scratch

I've always wondered how you can make a Linux system from Scratch without using gcc or GNU libc. I used to love a lot of
these esoteric Linux software stuff. It's obviously possible: there are [projects](https://os.ewe.moe/about) that have
achieved this feat, but I have yet to see a nice tutorial on a step-by-step construction of this system, similar to the
legendary LFS book. I hope to write a blogpost when I find out about how myself.

### WASM based package managing

This is semi-related to the thing above. I've been using Nix for a _long_ time and both the language and the package
manager are driving me a bit crazy. It's so burdensome to start a new package and/or write your own configuration,
because there are so many different ways to achieve the same thing and there is no rigid way of how things get to be
structured.

I want to fix that, and my Rusted brain wants me to make something in Rust. So I'm thinking of making package
descriptions from Rust files that will compile to WebAssembly, then get loaded by the package manager. I'm hoping to be
able to use some of the free time that I have at the moment to try something out, after I had done a complete LFS
walkthrough and understood the process.

### Rustfmt competitor

I really want a replace for rustfmt because I had a suboptimal experience when trying to improve the codebase. A lot of
things are locked behind rustfmt's stability guarantee which forbids breaking changes to formatted code and forces new
things behind an edition gate. I don't like it because I don't think the codebase is solid enough for such a stability
guarantee to work. More cleanups are necessary. I don't think [Topiary](https://github.com/tweag/topiary) will be good
enough in a reasonable timeframe to compete with rustfmt (it might be a little too ambitious).

More investigation needed.

### SPI stuff

On Wikipedia there's this process called Sock puppet investigations that investigates whether someone has used multiple
accounts to abuse. There are two things that I really want to work on, when I get time:
- CheckUsers have access to IP and additional data that helps determine whether two accounts are operated by the same
person. I want to improve the CheckUser interface because I think the current interface is not streamlined enough for
a smooth operation. This will start as a user script adding shortcuts, and I plan to gradually add more stuff to have
a more complete investigation experience.
- SPIs are really hard. Now that we have large language models, it might be really interesting to gather data from past
cases and see which ones have resulted in blocks being placed and which ones have been declined. There are of course
levels to that, for example sometimes it might result in CheckUser being run and sometimes it won't. Having data and
using models to analyze this might give people who work at SPI a lot of insight as to what makes good SPI cases good
(factoring in variables such as response times, what actions resulted from the report, basic data about the filer and
the users that have been filed against), and maybe it may also be useful to predict (for fun) the course of new SPI
cases as they come in.

## Wrapping Up

I like writing and it is nice to document stuff that I'm doing. But I just do a lot of things and it is sometimes hard
to fully summarize. I hope after this long post I can pick up and do more regular devlogs next.

Did you like this post? Let me know :D

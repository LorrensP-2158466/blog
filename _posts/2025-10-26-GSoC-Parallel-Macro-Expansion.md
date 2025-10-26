---
title: "GSoC '25 Work Product: Parallel Macro Expansion"
date: 2025-10-26
layout: post
---

# The project

A lot of work has been done to parallelise the Rust compiler, some parts are already parallelised, like codegen and processes after
HIR-lowering, such as type-checking, borrow-checking and MIR optimisation. For this years GSoC project I had the chance to parallelise the macro expansion algorithm.

# Preface

While the title of this post includes "**Parallel** Macro Expansion", I never got to parallelise the algorithm during this project. This was due to the sheer (unexpected) complexity of the project and
numerous complex issues/roadblocks that we faced as the project went on (one such roadblock even [surprised](https://rust-lang.zulipchat.com/#narrow/channel/421156-gsoc/topic/Project.3A.20Parallel.20Macro.20Expansion/near/542317541) my mentor).
This post will highlight my work and the issues/roadblocks we faced.

# The idea

The current algorithm is order-dependent, which prevents parallelisation. Simply put, it looks like this:

1. Collect unresolved imports and macro invocations.
2. Resolve a single collected import.
3. Commit: write the resolved import back into its module.
4. IF PROGRESS GOTO 1
5. Resolve a single collected macro.
6. Commit: expand the resolved macro.
7. IF PROGRESS GOTO 4
8. GOTO 1

The idea is to parallelise `2-4`, called import resolution, and `5-7`, called macro expansion[^1].

[^1]: The overall algorithm is called Macro Expansion, but it really is the expansion of the entire AST. But because macros influence everything related to the AST, even imports, we keep this name as is.


# My work

As said before, I was meant to do both parts of the algorithm, but I only made progress on the import resolution algorithm. To be complete, I'll explain this briefly as well.

It iteratively resolves undetermined imports until no further progress can be made, committing each resolved import to the module resolutions before proceeding. The goal is to isolate
these resolutions so that each import in the undetermined set can be processed independently and committed afterwards. We called this batched import resolution because resolving and
committing are done over the entire *batch* of undetermined imports.

As some of you may observe, we would also need to remove any kind of mutability in the `Resolver` that happens during this process, to avoid conflicts with other imports being resolved.

If we successfully implemented the above 2 points, it would be trivial to parallelise the algorithm.

## Removing mutability

These changes essentially boiled down into splitting the logic of the local crate and external crates. Items defined in external crates are considered a cache because they can not be changed once compiled,
so we only populate external crates when we actually need them.

### Macro Maps
The `Resolver` keeps a map of all compiled macros in a field called `macro_map`. You can access a compiled macro using a [`DefId`](https://rustc-dev-guide.rust-lang.org/hir.html#identifiers-in-the-hir). This map was read from or updated in this method:
```rust
pub(crate) fn get_macro_by_def_id(&mut self, def_id: DefId) -> &MacroData {
    if self.macro_map.contains_key(&def_id) {
        return &self.macro_map[&def_id];
    }
    let macro_data = ...; // not relevant
    self.macro_map.entry(def_id).or_insert(macro_data) // the mutable call
}
```
However, local macros were always present in the map when this method was called, so the map was only updated here when we added an external macro.
[This PR](https://github.com/rust-lang/rust/pull/143657) fixed this by splitting the maps and changing the logic to account for this.

### Defining bindings

An essential method of the `Resolver` is `define_binding`. Anytime you want to add particular binding (i.e. a name pointing to a thing) to a module, you'd call this method:
```rust
pub fn define_binding(...);
```
However, we would only call this for external items when we eventually parallelise the algorithm. [This PR](https://github.com/rust-lang/rust/pull/143884)
introduced `define_local` and `define_extern`, where `define_extern` takes a `&Resolver`.
The leading blockers here were underscore disambiguators (a way to differentiate `_` items) and external module maps (remember macro maps), fixed by [Vadim Petrochenkov](https://github.com/petrochenkov) in [#144272](https://github.com/rust-lang/rust/pull/144272)[^2] and [#143550](https://github.com/rust-lang/rust/pull/143550) respectively.

[^2]: We actually had to change underscore disambiguators for external items again some time later; you can find that [here](https://github.com/rust-lang/rust/pull/147805).

This change was very important, `define_binding` was used by the `resolutions` method of the `Resolver`, this method provides a way to access the resolutions of a particular module. But if
we want to access the resolutions of an external module, we first need to populate its resolutions if this has not yet been done (remember this is a cache). Now that we can use `define_extern` here, we can convert the `resolutions` method to
take a `&Resolver` instead of a mutable one. This allowed us to convert a lot of methods from `&mut Resolver` to `&Resolver`.

### Smaller but relevant changes

#### CrateLoader

A bit of context here: when an external crate is referenced through an `extern crate` item or path, the `Resolver` will try to load the referenced crate. This is done through the `CrateLoader`, which wraps the `CStore`
an important data structure of the compiler that keeps information of every external crate used by the currently compiling crate.

The problem was that the `CrateLoader` also had a mutable reference to some state in the `Resolver`,
which meant we needed a mutable reference to the `Resolver` to construct it. But the `Resolver` actually never used that state, only the `CrateLoader`, [this PR](https://github.com/rust-lang/rust/pull/144059)
refactored the logic and that state of the `CrateLoader` into the `CStore`. Now anytime we need the logic of the `CrateLoader`, we can use the `CStore` instead, removing the need for a mutable `Resolver`.

#### Cachify `ExternPreludeEntry.binding`

Updating this field is classified as a cache and was thus wrapped in a `Cell` in [#144605](https://github.com/rust-lang/rust/pull/144605).

### Conditional Mutable Resolver

Not all methods could be refactored like we did above; some require a mutable resolver when we are in later stages of name resolution. Some context here: when macro expansion is done, the compiler
does a bunch of postprocessing on the AST, things like finalising imports/macros, error reporting and [a bunch of other things](https://github.com/rust-lang/rust/blob/469357eb48d1c1b583386785ed0f846b9e7e0904/compiler/rustc_resolve/src/lib.rs#L1861), thus mutating the `Resolver`.
These methods know when to do extra things based on an optional parameter, and only when this parameter is present do we mutate the resolver.

We could have taken the approach I explained above, but it would have taken a lot of effort and time because these methods are quite complex and big. So we introduced a `Conditional Mutable Resolver` or `CmResolver` for short in [#144912](https://github.com/rust-lang/rust/pull/144912), which can give out mutable reference to the `Resolver` based on a particular condition:
```rust
pub(crate) struct RefOrMut<'a, T> {
  p: &'a mut T,
  mutable: bool,
}
impl<'a> RefOrMut<'a, T>{
  // You would get a mutable reference like so:
  pub fn get_mut(&mut self) -> &mut T{
    match self.mutable{
      false => panic!("Can't get a mutable reference"),
      true  => self.p,
    }
  }
}
type CmResolver<'r, 'ra, 'tcx> = RefOrMut<'r, Resolver<'ra, 'tcx>>;
// construct:
impl<'ra, 'tcx> Resolver<'ra, 'tcx>{
  fn cm<'r>(&'r mut self) -> CmResolver<'r, 'ra, 'tcx>{
    CmResolver::new(self, !self.assert_speculative)
  }
}
```

`assert_speculative` is a flag set and unset in the code for import resolution. Now we can change the relevant methods to take in a `self: CmResolver` instead of a `&mut Resolver`.
This type acts as a safeguard, it would panic if we try to get an exclusive reference during import resolution.

A related change was to add the same kind of wrappers around `Cell` and `RefCell` using the same flag, it would also panic if we would mutate the underlying `(Ref)Cell`. This was introduced in [#146283](https://github.com/rust-lang/rust/pull/146283) which would allow us
to make some important types `Send` and `Sync` safe in the future.

## Batched Import Resolution

This is where the fun starts and is still ongoing (it began on Aug 8), you can track the work [here](https://github.com/rust-lang/rust/pull/145108).
As I said in the beginning, we must resolve all undetermined imports, collect their side effects, and write them to the `Resolver` the same way the current algorithm does.
Import resolution works with 2 kinds of imports: single imports (`use foo::bar`) and glob imports (`use foo::*`), the side-effects from these are also different. Converting this algorithm proved to be difficult.
Any change I made that messed up the resolution order failed to compile `std`, which made it hard to debug.

### The std prelude import

The first thing I encountered, which took me off guard, was a bug caused by the implicit prelude the compiler injects into your crate. It's the first import that gets resolved, and every import after that depends on it:
```rust
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::v1::*; // a glob import
#[macro_use]
extern crate std;
```

Because we collect side-effects of imports, we also collect the side-effects of this special prelude import. Delaying these side-effects caused a bunch of confusing errors. [This PR](https://github.com/rust-lang/rust/pull/145322)
resolves the prelude import the moment we encounter it in the AST[^3], thus never adding it to the set of undetermined imports. The `Resolver` thus always sees the side effects of the prelude import when resolving
the undetermined set.

[^3]: It also helped a little with [this important PR](https://github.com/rust-lang/rust/pull/139493), yeah!

### Glob imports

The next challenge I faced was because of the way glob imports work. As you may know, glob imports import *everything* from the module, so for `use foo::*`, every public binding defined in `foo` will be imported.
It is however possible that bindings can be defined in a module *after* the glob import is resolved, so the `Resolver` needs a way to import these new bindings. This is done with a field in the [`Module`](https://doc.rust-lang.org/stable/nightly-rustc/rustc_resolve/struct.ModuleData.html)
called `glob_importers: Vec<Import>`, it's a way to find every resolved glob that references that module, so when we define a new binding, we go through that list and import it using those glob imports.
This works as intended in the current algorithm, but when in batched resolution, this can cause problems because adding a glob import to a module is a side-effect.
Consider the following example where we first resolve `pub use bar::baz` and then `use foo::*`:
```rust
mod foo {
    pub fn a() {}
    pub use bar::baz;
}
use foo::*;
fn main(){
  baz();
}
```
We first resolve `pub use bar::baz` and create a binding for it. Then we resolve `use foo::*`, which at that point only finds `a`. Next, we commit `baz` to `foo` and the bindings from `foo::*` to the root
and only then add `use foo::*` to the `glob_importers` of module `foo`. Because `use foo::*` was already resolved earlier, `baz` cannot be imported through it, which leads to an error when we try to use it.
To fix it we iterate twice over the side-effects: first to set all `glob_importers`, and then apply the remaining side-effects, This ensures glob import correctly import all bindings.
So here `baz` actually has a way to be imported by `use foo::*`;

### Side-effect problems

Incorrectly applying side effects is a recurring theme in this PR: single imports have a different kind of side effects, these are per namespace[^4]. The thing I did wrong was mindlessly copy-pasting
the original code to the new `commit_side_effects` code. But this was wrong, some imports go through multiple rounds of resolution until we determined that each namespace has a binding or not. The problem was when
we resolved a particular import a second time for another namespace, we actually reset the bindings for other already determined namespaces, which broke a lot of things. The fix for this was a
`Option<Option<NameBinding>>`, where the outer `Option` specified if a side-effect was present, and the inner `Option` was the actual side-effect for that namespace.

[^4]: Yes, Rust has namespaces, but not the kind you're thinking of, you can read more about it [here](https://rustc-dev-guide.rust-lang.org/name-resolution.html#namespaces).

### Glob imports and #[macro_use]

Once we fixed the above things, CI was green and my mentor/reviewer was content with the changes, we could run [Crater](https://github.com/rust-lang/crater) with my PR to see if it breaks code in the wild.
And boy it did, when analysing the crater report I noticed that all[^5] regressions (i.e. things that failed but shouldn't) were caused due to the [`rust-embed`](https://crates.io/crates/rust-embed) crate. For some
reason, it's derive macro `RustEmbed` exported as `Embed` (this export is important) couldn't be resolved by its dependents. I'll show the reduced case before explaining myself:

[^5]: There was one other reason some crates failed, it was due to a single line I introduced to make CI green, but we expected these. [#147984](https://github.com/rust-lang/rust/pull/147984) is fixing this.
```rust
// proc-macro crate rust_embed_impl/lib.rs
#[proc_macro_derive(RustEmbed)]
pub fn derive_input_object(input: TokenStream) -> TokenStream { ... }
```
  
```rust
// crate rust_embed/lib.rs
#[macro_use] // imports the `RustEmbed` macro
extern crate rust_embed_impl;
pub use rust_embed_impl::*; // imports and exports the `RustEmbed` macro

pub trait RustEmbed { ... }

pub use RustEmbed as Embed; // exports the `RustEmbed` trait *and* macro as `Embed`
```

The example above shows what happens with the current algorithm, and it's what most people *would* expect. But this is actually wrong and should be an error, but why? The `#[macro_use]` import
and the `pub use rust_embed_impl::*;` glob import are actually **ambiguous** imports, because they import the same thing, but in a totally different way, the `macro_use` imports privately while the glob
imports publicly. Currently, the compiler can only detect ambiguous imports when they import different things, so developers introduced such imports/exports in their code without knowing it's wrong.
The algorithm we introduced highlighted this bug in the compiler. I'll explain why.

With batched import resolution, we see things more clearly, we first resolve `pub use rust_embed_impl::*;` and find the `RustEmbed` macro as a resolution, then we resolve `pub use RustEmbed as Embed`, which finds
the `RustEmbed` trait and macro. But here is the catch: that macro is imported by the `#[macro_use]` import, not the one imported by the glob because it hasn't been not committed yet. After that, we commit the glob import, so we essentially exported a private macro, which makes the `Embed` macro invisible to the dependents of this crate (you can still use `RustEmbed` because of the glob). And because every
crate did `#[derive(Embed)]`, they all failed to compile:
```rust
use rust_embed::Embed;

#[derive(Embed)] // ERROR: `Embed` is not a derive macro, only a trait. Consider using `RustEmbed` instead.
struct Foo;
```

This took me a long time to figure out. Unfortunately, I can't fix it by altering the implementation because imports are required to be independent of their resolution order, which
isn't the case here. So until we have fixed this bug in the compiler, we need to hack these kinds of imports. You can find the current hack [here](https://github.com/rust-lang/rust/pull/145108/commits/88b3dafd89d1dc3063913caf197c561773f778f3), but I'm unsure if that is enough.
It checks whether a resolution from a public glob import in a particular module shadows a `#[macro_use]` import, and if that's the case, we set the visibility of that binding to public when we export it out of the crate.

## Future Work

Vadim is currently fixing some things related to ambiguous imports because there are other cases like above, these are [#147995](https://github.com/rust-lang/rust/pull/147995) and [#147984](https://github.com/rust-lang/rust/pull/147984).

Then we can convert the necessary types in the resolver to be used in a parallel context using the types described in [the conditional mutability section](#conditional-mutable-resolver) and the types in [rustc_data_structures::sync](https://doc.rust-lang.org/stable/nightly-rustc/rustc_data_structures/sync/index.html).
And finally convert the import resolution loop to a parallel loop.

----





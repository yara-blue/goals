# Extended reflection

| Metadata             |                                       |
|:---------------------|:--------------------------------------|
| Contact              | @yara-blue                            |
| Funding contact      | [Hexcat](https://hexcat.nl)           |
| Status               | Proposed                              |
| What and why         | Enable code analysis using reflection |
| Timespan             | 2026-2027                             |
| Roadmap              | Easily write code analysis            |
| Highlight            | Const and reflection                  |
| Zulip channel        | [#goals/reflection](https://rust-lang.zulipchat.com/#narrow/channel/572285-goals.2Freflection) |
| [compiler] champion  | @oli-obk                              |
| [lang] champion      | @nikomatsakis                         |
| [types] champion     | @oli-obk                              |

## Summary

Code analysis tools are currently separate tools outside the compiler. This has led to a few large code-analysis projects. We will extend the reflection API such that code-analysis can run as a compiler plugin during const-eval.

## Motivation

### The status quo

Code analysis is commonly done by [building your own compiler driver](https://rustc-dev-guide.rust-lang.org/rustc-driver/intro.html). That will then run as a separate tool in the build system. This has a few downsides:

* You need to run the compiler once normally, once for Clippy, once for your proof checker, once for every linter, and so on.
* You need to use the internal API of rustc, which changes regularly (see also [rustc_public](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_public/)).
* You need to update all your linters to new versions of the rustc libraries to use new language features.
* Lints need to be aggregated into lint checkers, rather than allowing crates to ship their own lints.

In practice this has led to similar situations like with serialization, where one large project (in this case Clippy) is where all the lints live, instead of having a decentralized system where each crate can ship lints specifically for its users.

Some tools ingest [rustdoc-json](https://rust-lang.github.io/rfcs/2963-rustdoc-json.html) instead of building a compiler driver. This sidesteps some of the issues above, but also limits the code analysis to the information exposed by rustdoc-json. A well-known tool taking this approach is [cargo-semver-checks](https://github.com/obi1kenobi/cargo-semver-checks).

There are four existing projects that try to solve this outside the compiler:
* [charon](https://github.com/AeneasVerif/charon) (ongoing)
* [dylint](https://github.com/trailofbits/dylint) (ongoing)
* [redpen](https://github.com/estebank/redpen) (abandoned 2 years ago)
* [marker](https://github.com/rust-marker/marker) (abandoned 3 years ago)

charon and dylint are both compiler drivers that provide an interface for other code analysis tools to plug into. However, even universal adoption of either would address only some of the issues described above.

### What we propose to do about it

We already aim to stabilize a minimal reflection API (see [min-type-reflection](https://goals.rust-lang.org/2026/min-type-reflection.html)). We should extend that interface to expose enough of the compiler so code-analysis can be done in const-eval. This way analysis will run directly in the compiler.

To get there, we will add a `#[rustc_linttime]` attribute to clearly separate functions that are useful for linting but could break semver (e.g. accessing private fields and layout information). We may remove this distinction later (see the FAQ for why). We will then add a way to register these functions as lints, along with APIs for emitting lint warnings and errors.

With that in place, we will extend the reflection API for code analysis, restricting it to _linttime_ where needed. As a part of this, we will:
- Design a maximally desugared, minimal representation of Rust code (function bodies). We will demonstrate a purely textural representation and compare it to a structured representation (possibly basing the former on the latter).
- Expose APIs for performing various trait solver operations (such as projection normalization and proving bounds).

In the future, we want to expose the entire API to executable wasm blobs as part of the [reflect-from-wasm goal](https://goals.rust-lang.org/2026/reflect-from-wasm.html).

### Design axioms

- Prefer small, composable, targeted methods, continuing the approach in [min-type-reflection](https://goals.rust-lang.org/2026/min-type-reflection.html). For example, instead of adding `type_id.is_f32()`, we should let users use the existing `type_id.size()` with `matches!(type_id.kind(), TyKind::Float)`. Then, when Rust gets an `f16` type, we need no new methods.
- Guide what demonstrators we develop by concrete use cases (e.g. existing lints or proof checkers).
- Build upon the work done by other projects such as _rustc-public_ and _charon_. 
- Avoid mixing semver-breaking APIs with the rest of the reflection interface.
- Create APIs that are clearly marked as prototypes (e.g. macros/attrs with placeholder names) for lint registration and emission, to avoid limiting future design work.

### The "shiny future" we are working towards

We're shooting for a future where it is easy to apply your own and third-party code analysis, such as model checkers, lints, and diagnostics, to any Rust project. See the [TODO: Code analysis roadmap](https://goals.rust-lang.org/2026/roadmap-code-analysis.html) for details.

This is currently an open ended goal, however with most of this goal in place users will be able to write code-analysis tools in const-eval. We will have a (limited) model-checker demonstrator which proof whether a piece of code can panic.

### Work items over the next year

| Task                                                          | Owner(s)   | Notes |
|---------------------------------------------------------------|------------|-------|
| Introduce `#[rustc_linttime]` functions                       | @yara-blue |       |
| Create APIs for registering lints and emitting diagnostics    | @yara-blue |       |
| Design a maximally desugared, minimal language representation | @yara-blue |       |
| Extend the reflection API and build the demonstrator          | @yara-blue |       |
| Publish a call for testing                                    | @yara-blue |       |
| Extend the API further based on user feedback                 | @yara-blue |       |
  

## Team asks

| Team       | Support level | Notes                                                    |
|------------|---------------|--------------------------------------------------------- |
| [compiler] | Medium        | Standard reviews                                         |
| [lang]     | Large         | Design of the minimal language                           |
| [libs]     | Small         | Linter API review (prototype)                            |
| [types]    | Medium        | Trait solver API                                         |

## Funding

| Purpose | Cost | Funded | Sponsor(s) |
|---------|------|--------|------------|
| Contributor | Ask | No |  |

## FAQ

### How does this not cause new semver hazards?

If code analysis panics or otherwise crashes, we will emit a new deny-by-default lint instead of failing compilation. This means that private changes inside a crate may trigger the lint in downstream builds, but it will be possible to disable it. Such changes are generally considered semver-compatible (see [the Cargo Book](https://doc.rust-lang.org/cargo/reference/semver.html#new-lints) and [RFC 1105](https://rust-lang.github.io/rfcs/1105-api-evolution.html#lints)).

### The effect of lints on upstream crates

Such extensive code-analysis available in the language enables adding features
that look like language features outside the compiler. These will inevitably
create some interesting scenarios, for example when code analysis in an upstream crate emits a warning pointing to a crate further upstream.

For example, imagine authoring the crate `resampler` that depends on the `fft` crate, which in turn relies on `complex`. 

The `fft` crate uses code analysis to ensure its API can never panic:

```rust
// in fft's lib.rs
use complex::Complex;

#[forbid_panicking]
pub fn fft(data: &[f32]) -> &[Complex<f32>] {
    // ...
    let c = Complex::from(a, b);
   //  ...
}
```

Now `complex` puts out a patch release which changes the `Complex::from` implementation so that it can now panic. I'd argue this is not semver-breaking, at least not in the current definition. Now you run `cargo update` and the version of `complex` in your tree is updated, but now `fft` starts emitting lints. The lint emitted is easily ignored, but should you? Did `fft` break? Did `complex`? Where do you raise an issue, if you do at all?

We'll take this into account while designing and experimenting with the interface.

### Why even do any const-eval and not just Wasm blobs?

Const-eval allows for easier experimentation at this stage without much cost to maintenance (considering `rustc_comptime` already exists, it is trivial to extend it with `rustc_linttime`).

We also want to compare running code analysis in const-eval with compiling to Wasm first, but that is part of the [reflect-from-wasm](https://goals.rust-lang.org/2026/reflect-from-wasm.html) goal.

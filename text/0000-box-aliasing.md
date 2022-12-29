- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Explicitly permit unsafe aliasing with `Box<T>`.

# Motivation
[motivation]: #motivation

The aliasing rules for `Box` are not entirely clear.
Currently, rustc emits `noalias` on `Box` parameters, which implies some kind of aliasing requirement.
Attempts to formalize this generate examples which authors of unsafe code who are not intimately familiar with compiler internals find deeply disturbing.
For example, formalizing this rule in Stacked Borrows says that this program is UB:
```rust
let mut b = Box::new(0usize);
let ptr = &*b as *const usize;
let a = b;
unsafe {
    dbg!(*ptr);
}
```
To emphasize this point: The way users react to learning about this is different from learning about aliasing implications of `&mut`.
When confronted with this situation, users routinely have all their intuition about `Box<T>` invalidated, as opposed to being led to a greater understanding about the complexity of aliasing optimizations.
For `&mut`, users are sometimes frustrated by what can be derived from axioms they are familiar with.
But for `Box<T>`, such users are shocked by the axioms.

In safe Rust, our users cannot avoid learning the rules because the compiler issues hard errors.
But in the current state of unsafe Rust, we do not even have a single place that anyone can refer users to so that they can learn all the rules.
And on top of that we need to contend with the reality that while we are still figuring out what exactly the rules for unsafe Rust are, we are (at time of writing) 8 years past 1.0.
We have 8 years of legacy code and some level of stability obligation to all of that code.
Optimizing that code based on undocumented and unmentioned aliasing properties is likely to cause problems.

At the time of writing, we are aware of 149 crates published on crates.io which encounter UB due to the aliasing rules for `Box<T>`.
(Should we post this list somewhere?)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There is nothing to teach here.
The existing documentation for the type `Box` is sufficient, which is the point of this RFC.
Some documentation in `std::boxed` which was previously added in an effort to ameliorate the confusion discussed in this PR can now be elided.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

An implementation could be as simple as this, though the comment should probably be adjusted too:
```diff
diff --git a/compiler/rustc_ty_utils/src/abi.rs b/compiler/rustc_ty_utils/src/abi.rs
index 73c7eb6992f..ab63ee58d74 100644
--- a/compiler/rustc_ty_utils/src/abi.rs
+++ b/compiler/rustc_ty_utils/src/abi.rs
@@ -242,7 +242,7 @@ fn adjust_for_rust_scalar<'tcx>(
             // The aliasing rules for `Box<T>` are still not decided, but currently we emit
             // `noalias` for it. This can be turned off using an unstable flag.
             // See https://github.com/rust-lang/unsafe-code-guidelines/issues/326
-            let noalias_for_box = cx.tcx.sess.opts.unstable_opts.box_noalias.unwrap_or(true);
+            let noalias_for_box = cx.tcx.sess.opts.unstable_opts.box_noalias.unwrap_or(false);

             // `&mut` pointer parameters never alias other parameters,
             // or mutable global data
```

# Drawbacks
[drawbacks]: #drawbacks

This change will remove some aliasing optimizations from existing Rust programs, and by promising that `Box<T>` does not have any special aliasing properties, we will be for sure locked out of re-introducing something like LLVM `noalias` on `Box<T>` parameters and returns in the future.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Aliasing optimizations on `Box<T>` are predicated on introducing rules that surprise users of unsafe Rust.
In combination with the already-existing legacy Rust code in the wild, even if we were to effectively publicize and document the aliasing rules for `Box<T>`, optimizations based on those rules would be predicated on rules that existing code assumes don't exist.

If we do not remove aliasing optimizations based on `Box<T>`, we risk introducing surprise "miscompilations" into existing and future Rust code.

# Prior art
[prior-art]: #prior-art

The C++ standard library also provides an owned pointer type, `std::unique_ptr<T>`, and it is common to instruct newcomers from C++ that `Box` is like `std::unique_ptr`, and `Arc` is like `std::shared_ptr`.
But if `Box` has aliasing implications, then this is dangerously wrong in subtle ways and Rust does not have an equivalent of C++'s `std::unique_ptr`.
Of course we are not beholden to the practices of other languages or the way Rust is taught currently, but it is worthwhile to consider that maybe `Box` is not the place to spend some of our strangeness budget.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

What is the runtime cost of this change to Rust as it is written today? We currently have a flag `-Zbox-noalias=no` which can be used to turn off `noalias` on `Box<T>`, but in exercising the flag we have only gotten reports of "no change". This is absence of evidence for such aliasing optimizations being effective currently, but not evidence of absence.

# Future possibilities
[future-possibilities]: #future-possibilities

Instead of applying aliasing optimizations to existing types, we could add a new library type with some special aliasing properties.
This would permit users who really care about performance to add back in any aliasing optimizations which are lost by removing `noalias` from `Box<T>`.

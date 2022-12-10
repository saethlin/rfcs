- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Explicitly permit aliasing with `Box<T>`.

# Motivation
[motivation]: #motivation

The aliasing rules for `Box` are not entirely clear.
rustc emits `noalias` on `Box` parameters, which implies some kind of aliasing requirement.
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

This is distinct from how users respond when learning about the aliasing implications of `&mut`.
They seem to have some intuition already that `&mut` has something to do with uniqueness.
Perhaps the exact way that manifests is surprising, but with `Box`, users are shocked that there are _any_ aliasing implications.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There is nothing to teach here.
The existing documentation for the type `Box` is sufficient, which is the point of this RFC.

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

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

# Prior art
[prior-art]: #prior-art

The C++ standard library also provides an owned pointer type, `std::unique_ptr<T>`.
It is common to instruct newcomers from C++ that `Box` is like `std::unique_ptr`, and `Arc` is like `std::shared_ptr`.
If `Box` has aliasing implications, this is wrong, and Rust does not have an equivalent of C++'s `std::unique_ptr`.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities

Automatically applying aliasing properties to existing types is a very blunt instrument for producing optimizations.

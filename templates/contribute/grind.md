# `grind` Best Practices

`grind` is a powerful proof automation tactic recently added to core Lean. Best practices for using
grind` in Mathlib are still developing, but this page collects topics that seem to have reached
consensus on Zulip.

## Communication

## Usage in Fundamental Modules

One place where usage of `grind` is likely undesirable is in modules very high in the import graph
which establish the foundations of Mathlib's algebraic hierarchy hierarchy. Because `grind` uses
its own parallel theory of topics such as groups and rings, this results in proofs that are exceedingly
opaque and use choice in unexpected ways.

## Performance

Any pull requests that alter existing proofs to use `grind` should include the results performance
`trace.profiler`, with the current rule of thumb being that differences smaller than 30 ms are
considered with a margin of error. Below are a few suggestion for better `grind` performance.

## Explicit Unification

Like `simp`, `grind` has the ability to make arbitrary terms as parameters. This means that it is
possible to golf a usage of `apply` into a parameter passed to `grind`. While this may allow for a
more compact proof, this often comes with performance and readability issues.

## Parameter Golfing

When a proof by induction is using `grind` in each branch of the proof, it is possible to combine
these into a single call to `grind` that uses the union of any parameters. For instance, you could
write

```lean
example {α : Type} {xs ys : List α} : (xs ++ ys).length = xs.length + ys.length := by
  induction xs with grind only [= List.length_nil, = List.cons_append, = List.length_cons]
```
```
```

While this may be more compact, it obscures which rules are being used for each branch of a
proof and potentially creates performance issues. Unless the minimal parameters  that `grind`
requires are the same for each branch, it is better to separate these calls to `grind`.

## Maintainability

A key concern of using `grind`, which is still under active development, is the ability to repair
proofs that break across toolchains. Typically the first step in examining such breakage is to look
at the output of `grind?` from the successful proof of the previous toolchain, which can provide a
`grind only` that lists all of the theorems used. However, it is possible for the `grind?` tactic to
produce a valid proof. For this reason it is encouraged to always check for this when submitting a PR,
either manually or by enabling the `linter.tacticAnalysis.verifyGrindOnly` linter.

Restructuring a proof to have a successful `grind?` output can usually be done by being more
explicit with adding intermediate local hypothesis and cases splitting.

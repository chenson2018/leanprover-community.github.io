# `grind` Best Practices

`grind` is a powerful proof automation tactic recently added to core Lean. Best practices for usage in Mathlib are still developing, but this page collects topics that seem to have reached consensus on Zulip. These revolve roughly around the intertwined trade-offs concerning maintainability, readability, and performance that should be considered when using `grind`.

#### Proof Stability Across Toolchains

A key concern of using `grind`, which is still in active development, is the ability to repair proofs that break across toolchains. Typically the first step in examining such breakage is to look at the output of `grind?` from the successful proof on the previous toolchain. Usually this will provide a `grind only` that lists all of the theorems used, but it is possible for this tactic to fail. For this reason it is encouraged to always check for this when submitting a PR, either manually or by enabling the `linter.tacticAnalysis.verifyGrindOnly` linter. Without this step, it is easy for Mathlib to accrue proofs that are much more difficult to repair, especially for new proofs that have used `grind` from the onset.

Restructuring a proof to have a successful `grind?` output can usually be done by being more explicit with adding intermediate local hypothesis and case splitting. It is also acceptable to manually construct a `grind only` and leave a `set_option linter.tacticAnalysis.verifyGrindOnly false in` for the theorem so that it is excluded from the weekly linting report that collects these failures.

#### Usage in Fundamental Modules

One place where usage of `grind` is likely undesirable is in modules very high in the import graph which establish the foundations of Mathlib's algebraic hierarchy. Because `grind` uses its own parallel theory of topics such as groups and rings, this results in proofs that are exceedingly opaque and use choice in unexpected ways.

#### Golfing of Short Proofs

Using `grind` to golf proofs that are already very short usually makes a sacrifice in readability (and performance, see also *Parameter Golfing* below) that is undesirable. A common example of this is rewriting a proof that explicitly proves both directions of an `Iff` into a single call to `grind`. When each direction involves fairly involved proofs, this quickly becomes much less clear.

#### Parameter Golfing

When a proof by induction is using `grind` in each branch of the proof, it is possible to combine these into a single call to `grind` that uses the union of any parameters. For instance, you could write

```lean
example {α : Type} {xs ys : List α} : (xs ++ ys).length = xs.length + ys.length := by
  induction xs with grind only [= List.length_nil, = List.cons_append, = List.length_cons]
```

While this may be more compact, it obscures which rules are being used for each branch of a proof and potentially creates performance issues. Unless the minimal parameters  that `grind` requires are the same for each branch, it is better to separate these calls to `grind`.

#### `#grind_lint`

Mathlib contains a test [MathlibTest/grind/lint.lean](https://github.com/leanprover-community/mathlib4/blob/master/MathlibTest/grind/lint.lean) that checks for excessive instantiations caused by the interaction of `grind` annotations. When this test fails, it provides a code action for the corresponding `#grind_lint inspect` commands to provide additional debugging information.

#### Squeezing

Similar to `simp`, `grind only` is a "squeezed" output usually provided by `grind?`. While this can be used to address performance issues, it is similarly not preferred for the same reasons as `simp`.

#### Explicit Unification

Like `simp`, `grind` has the ability to make arbitrary terms as parameters. This means that it is possible to golf a usage of `apply` into a parameter passed to `grind`. While this may allow for a more compact proof, this often comes with performance and readability issues.

#### Using Interactive Mode to Minimize Calls to `grind`

In proofs with a complex local context, having many calls to `grind` in successive `have` blocks means repeatedly incurring a startup cost from `grind`. In these situations, it can be more performant to use `grind =>`, the interactive mode. Within these tactic blocks, `have` with only a signature indicates that a proof should be provided by `grind`. As an example, consider:

```lean
def x := 1
def y := 2

example : x + y = 3 := by
  have : 1 + 2 = 3 := by grind
  grind [x, y]

example : x + y = 3 := by
  grind =>
    have : 1 + 2 = 3 
    instantiate only [x, y]
```

While the proof of `1 + 2 = 3` here is trivial (`grind` would do this automatically) this demonstrates the general idea of reducing to a single call to `grind` with user-guided intermidate proofs.

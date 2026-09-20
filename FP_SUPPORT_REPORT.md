# Floating-Point Symbolic Execution Support in Java Ranger
## The `fpSupport` Branch: Motivation, Iterative Development, and Contributions

**Author:** Salmane Khalili
**Date:** September 2026
**Scope:** Work performed on the `fpSupport` branch of *Java Ranger* (a path-merging extension of Symbolic PathFinder, SPF) and its associated feature branches, as recorded in the git history of `SalmaneKhalili/java-ranger` and the pull-request (PR) history of `vaibhavbsharma/java-ranger`.

---

## Abstract

Symbolic execution of Java programs that use floating-point arithmetic is notoriously difficult: the IEEE 754 standard defines a rich landscape of special values (NaN, signed infinities, signed zero), and comparison and arithmetic instructions have non-trivial, case-dependent semantics for those values. This report documents the design, iterations, and contributions of the `fpSupport` branch of Java Ranger, whose goal is to make the symbolic execution engine's handling of floating-point programs spec-faithful to IEEE 754. The work comprises four threads: **(i)** a foundation of IEEE 754 infrastructure (special-value constants, floating-point-aware comparators, domain/bounds fixes, an `isNaN` predicate, and a Z3 translation layer); **(ii)** spec-faithful symbolic encodings of the JVM division instructions `FDIV`/`DDIV` and remainder instructions `FREM`/`DREM` as *outcome-choice* path conditions; **(iii)** symbolic support for the transcendental `Math.sin`, explored through three successive approaches (CEGIS concretization, Taylor-series peers, and a piecewise-linear ITE encoding); and **(iv)** cross-cutting integration with the string solver backend and a symcrete test methodology. Throughout, the PR history preserves not only the merged results but also a detailed record of every attempted approach, closed iteration, and reviewer-driven refactor, which is itself a central artifact of this work. Four contributions have been merged into the upstream `fpSupport`/`svcomp` branches, two PRs are currently open, one is drafted, and twelve closed PRs document the evolution of the ideas.

---

## 1. Introduction and Background

### 1.1 Symbolic Execution and Java Ranger

Symbolic execution (SE) executes a program with *symbolic* rather than concrete inputs, accumulating a **path condition (PC)** — a logical formula over the symbolic inputs — along the explored execution paths. For each path, a satisfiability-modulo-theories (SMT) solver decides whether the PC is feasible, and a bug property is checked along the way. When an assertion fails along a feasible path, the solver provides a concrete model of the inputs that reproduces the bug, i.e., a *witness*.

**Java Ranger** (Sharma et al.) is a path-merging extension of **Symbolic PathFinder (SPF)**, itself a symbolic execution engine built on the Java PathFinder (JPF) model checker. Java Ranger implements *veritesting*: it summarizes multi-path regions by merging paths into a single symbolic formula, dramatically reducing path explosion. Its development is coordinated on the repository `vaibhavbsharma/java-ranger`, with the bulk of the symbolic-concurrency work living in the `jpf-symbc` module.

### 1.2 Why Floating-Point Symbolic Execution Is Hard

The JVM's floating-point instructions and the `java.lang.Math` library follow the **IEEE 754** standard, whose semantics are inconvenient for the classical symbolic execution pipeline in at least five ways:

1. **Special values.** IEEE 754 defines values that are *not ordinary real numbers*: `NaN` (not a number), positive and negative infinity, and signed zero. Operations such as `0.0 / 0.0`, `1.0 / 0.0`, and `Inf - Inf` produce them. If the expression layer models every symbolic "real" as a genuine real number (as the `RealExpression` family did originally), those outcomes are unrepresentable and the analysis is **unsound**: paths that produce or consume special values are either missed entirely or reported incorrectly.
2. **Comparison semantics.** The JVM comparison instructions `FCMPG`/`FCMPL` (float) and `DCMPG`/`DCMPL` (double) do not follow the usual trichotomy. When an operand is `NaN`, the "unordered" case fires: `FCMPL`/`DCMPL` push `-1` while `FCMPG`/`DCMPG` push `+1`. Every comparison therefore needs an explicit *unordered* branch in addition to `<`, `==`, and `>`.
3. **Equality is not reflexive.** In IEEE 754, `NaN != NaN` is *true*. A solver that treats equality as syntactic identity, or a path condition that binds a result to its own symbolic variable before checking `res != res`, will silently miss the NaN branch.
4. **Division and remainder are total.** `x / 0.0` must not raise an exception: it yields `±Inf` (for a non-zero dividend) or `NaN` (for `0.0 / 0.0`). Similarly `x % 0.0` yields `NaN`, not a division-by-zero exception.
5. **Transcendental functions.** `Math.sin`, `Math.cos`, etc. are not part of the SMT theories supported by the underlying solvers (Z3's floating-point theory, the bit-vector backend, and the string backend), so symbolic calls to these methods previously ended in a crash or an "unsupported" exception.

The `fpSupport` branch addresses all five challenges. The remainder of this report describes, in the order they were tackled, the foundation work, the division/remainder encodings, the `Math.sin` support, and the cross-cutting integration, followed by a summary of the review process and the lessons it produced.

---

## 2. The `fpSupport` Branch and the PR Landscape

### 2.1 Branch Topology

The floating-point work is developed on short-lived feature branches (e.g., `fp-fdiv-inf-nan-support`, `fp-frem-support`, `fp-math-sin-support`, `fp-sin-curvature-weighted`, `frem-drem-rem`, `translationlayer`, `fp-isub-overflow`, `experimentalnan`) and integrated into the personal `fpSupport` branch. Contributions that are ready for the project are opened as PRs against the *same* `fpSupport` branch on the upstream repository `vaibhavbsharma/java-ranger`; merged contributions then land on `fpSupport` and propagate into the fork `SalmaneKhalili/java-ranger`. A secondary line of work targets the `svcomp` branch, which hosts Java Ranger's SV-COMP benchmark infrastructure.

Two staging mechanics are visible in the git history:

- *Fork-internal PRs.* Early iterations were opened and reviewed as PRs *within the fork* (PRs #1–#4 of `SalmaneKhalili/java-ranger`) before being promoted upstream. For example, the first-generation `FREM` support was merged into the fork's `fpSupport` via fork PR #2 (merged 2026-07-22), anchoring the fork's integration history; it was later *rewritten*, not merged upstream.
- *Base restarts.* When a PR accumulated too much review debt, its base was restarted. PR #45 was closed and reopened as PR #54 ("the other PR has gotten too messy for me to track your comments... I have reopened it here with a major refactor"). Similarly, PR #23 was closed and reopened as #24 after a merge-conflict/branch mix-up between the `sv-comp` branch of Java Ranger and the identically named branch of SPF.

### 2.2 The Pull-Request Inventory

Table 1 summarizes all PRs authored by the author of this report on `vaibhavbsharma/java-ranger`. (Fork-internal PRs #1–#4 on `SalmaneKhalili/java-ranger` are covered in later sections where they are the primary record.)

**Table 1 — PRs on `vaibhavbsharma/java-ranger` (author: Salmane Khalili)**

| PR | Status | Created → Closed / Merged | Base → Head | Net diff | One-line description |
|----|--------|---------------------------|-------------|----------|----------------------|
| #24 | **Merged** | 2026-01-21 → 2026-01-24 | `svcomp` → `svcomp` | +7/−0, 1 file | Java-based `Math.toRadians`/`toDegrees` for symbolic tracking (fixes Issue #22) |
| #31 | **Merged** | 2026-03-03 → 2026-03-16 | `fpSupport` → `feature/issue27-special-values` | +381/−0, 8 files | IEEE 754 special-value constants: `NaN`, `±Inf`, signed zero (Issue #27) |
| #46 | **Merged** | 2026-07-15 → 2026-08-13 | `fpSupport` → `fp-isub-overflow` | +78/−12, 3 files | Integer-overflow simulation for `ISUB` (mirrors `IADD`) |
| #53 | **Merged** | 2026-09-05 → 2026-09-13 | `fpSupport` → `translationlayer` | +57/−11, 2 files | Missing Z3 operations in `ProblemZ3`/`ProblemZ3BitVector` |
| #23 | Closed | 2026-01-20 → 2026-01-21 | `svcomp` → `default` | +34/−1, 3 files | Early (superseded) `toRadians`/`toDegrees` attempt |
| #36 | Closed | 2026-05-24 → 2026-06-19 | `fpSupport` → `feature/issue29-isNan-Predicate` | +231/−0, 3 files | `isNaN` predicate `RealIsNaN` (Issue #29) — absorbed into the unary-comparator work |
| #38 | Closed | 2026-06-27 → 2026-06-27 | `master` → `fp-fdiv-ddiv-fixes` | (344 commits, 2071 files — messy base) | Early NaN/infinity semantics for symbolic `FDIV`/`DDIV` (cherry-pick of `experimentalnan`) |
| #39 | Closed | 2026-06-27 → 2026-06-27 | `fpSupport` → `fp-fdiv-ddiv-fixes` | +331/−67, 15 files | Integration tests for symbolic `FDIV`/`DDIV` NaN/infinity semantics |
| #40 | Closed | 2026-06-27 → 2026-07-05 | `fpSupport` → `fp-fdiv-ddiv-fixes` | +310/−69, 14 files | Refined version of the integration tests |
| #41 | Closed | 2026-07-05 → 2026-07-05 | `fpSupport` → `fp-bounds-div-fixes` | (8 commits, 1734 files — messy base) | FP bounds + `FDIV`/`DDIV` domain fixes (strict → non-strict inequalities, no exception on ÷0) |
| #42 | Closed | 2026-07-05 → 2026-07-08 | `fpSupport` → `fp-bounds-div-fixes` | +229/−56, 10 files | Cleaned FP bounds/domain and `FDIV`/`DDIV` fixes |
| #43 | Closed | 2026-07-05 → 2026-07-22 | `svcomp` → `fp-fcmp-dcmp-fixes` | +533/−224, 14 files | NaN-aware `FCMP`/`DCMP` 5-way branches (vs `svcomp`) |
| #44 | Closed | 2026-07-05 → 2026-07-08 | `fpSupport` → `fp-frem-support` | +374/−21, 22 files | First-generation symbolic `FREM` (superseded by the FDIV-style rewrite) |
| #47 | Closed | 2026-07-22 → 2026-08-05 | `fpSupport` → `fpSupport` | +1195/−308, 39 files | NaN handling for `FCMPG`/`FCMPL`/`DCMPG`/`DCMPL` + FP comparator infrastructure (split, then committed directly) |
| #48 | Closed | 2026-07-25 → 2026-07-25 | `fpSupport` → `fp-math-sin-support` | +1518/−229, 50 files | `Math.sin()` via CEGIS concretization (superseded by the piecewise-linear encoding) |
| #45 | Closed | 2026-07-14 → 2026-09-16 | `fpSupport` → `fp-fdiv-inf-nan-support` | +1103/−153, 14 files | 6-arm `FDIV`/`DDIV` outcome constraints (superseded by the reopening in #54) |
| #54 | **Open** | 2026-09-16 → | `fpSupport` → `fp-fdiv-inf-nan-support` | +1094/−146, 14 files | Spec-faithful 6-arm `FDIV`/`DDIV` outcome constraints (disjunction-only) |
| #37 | **Open** | 2026-06-22 → | `fpSupport` → `fix/string-solver-realconstraint` | +7/−0, 2 files | Symbolic FP combined with the z3str3 string solver |
| #55 | **Draft** | 2026-09-16 → | `fpSupport` → `frem-drem-rem` | +580/−61, 8 files | Spec-faithful 3-arm `FREM`/`DREM` on top of PR #54 |

The distribution is telling: **4 merged, 2 open, 1 drafted, and 12 closed-but-not-merged.** The closed PRs are not wasted work; they are the raw history of *what was tried, why it was abandoned, and how the ideas evolved* into the open PRs. The following sections reconstruct that history thematically.

---

## 3. Foundation: IEEE 754 Infrastructure

### 3.1 Special-Value Constants (Issue #27, merged in #31)

The first and most fundamental problem was representability. The original expression layer treated every floating-point symbolic value as an ordinary real number. Issue #27 formalized the gap: division by zero, overflow, underflow, and invalid operations produce `NaN`, `±Inf`, and signed zero, and "without this fix, those paths are incorrectly modelled as normal arithmetic." The SV-COMP impact analysis in the issue lists concrete benchmarks (`CWE369_Divide_by_Zero`, `Adjustable-Inner-Count-Halving`) whose verdicts are wrong precisely because of this gap.

PR #31 introduced:

- `FpSort` — an enum distinguishing single (float) vs double (double) precision;
- `RealSpecialConstant` — the abstract base class holding the precision and (for `Inf`/zero) the sign;
- `RealNaN`, `RealInfinity`, and `RealZero` — concrete subclasses, including bit-level negative-zero storage.

These classes extend the existing `RealConstant` hierarchy, so constant folding and expression visitors handle them without restructuring the entire expression tree. The PR was reviewed and merged into `fpSupport` on 2026-03-16 after the reviewer's single procedural request — retarget the PR from `master` to `fpSupport` — was honored. This small PR is the load-bearing wall of everything that follows: every later encoding (`FDIV`/`DDIV`, `FREM`/`DREM`, comparisons, and even the `sin` ITE trees) reasons about NaN and infinities as first-class citizens.

### 3.2 The `isNaN` Predicate (Issue #29, PR #36)

Issue #29 asked for a way to *test* whether an expression is `NaN` — required both by the JVM comparison instructions (which dispatch on "unordered") and by user-level `Float.isNaN()`/`Double.isNaN()` calls. The proposal (PR #36, created 2026-05-24) implemented a new unary expression node `RealIsNaN` extending `IntegerExpression` (a boolean encoded as 0/1), with visitor hooks and solver-facing methods (`stringPC()`, `getVarsVals()`, `compareTo()`, `getSort()`), plus unit tests.

The PR was closed without merging on 2026-06-19. Its functionality was not lost, however: it was *absorbed* into the more general **unary FP comparator infrastructure** — `IS_NAN`, `NOT_IS_NAN`, `IS_INF`, `NOT_IS_INF` (and later `IS_ZERO`, `IS_INFINITY`, and their negations) in the `Comparator` layer — which a richer design needed anyway (Section 4). This is the first clear example in the record of the iterative pattern *"try a focused node, then generalize it into shared infrastructure."*

### 3.3 NaN-Aware Comparisons and the Comparator Infrastructure (PRs #43, #47, fork #1/#3)

The JVM's four comparison instructions were originally implemented as a **2-branch choice generator** (`<` vs `>=`, say), which is wrong for two reasons: it ignores `NaN` (which must yield the "unordered" result), and the branch conditions themselves could not be expressed correctly under IEEE 754 semantics.

The fix landed in stages:

- **PR #43** (2026-07-05, base `svcomp`): a 5-way branch (`LT`/`EQ`/`GT`/`NaN`/`inf`) with correct IEEE 754 result values and an explicit `NE` constraint on `NaN`, plus FP-aware `eq`/`neq` in `ProblemZ3BitVector` using Z3's `mkFPEq` so that `NaN != NaN` is satisfiable.
- **PR #47** (2026-07-22): the consolidated PR on `fpSupport`. It replaced the 2-branch CG with a **4-branch CG** (`<`, `==`, `>`, *unordered*), added the unary comparator infrastructure (`IS_NAN`/`NOT_IS_NAN`/`IS_INF`/`NOT_IS_INF`), FP-aware `div`/`eq`/`neq` in the Z3 solver layer, and a new `symbolic.inf` configuration flag that widens the FP domain to include infinities.

PR #47's review history is instructive and is analyzed in Section 8; in short, the reviewer (Dr. Soha Hussein) required (a) the PR to contain only new/modified code rather than delete-and-rewrite churn, (b) a split into smaller, reviewable PRs, and (c) an explanation of the pre-existing code that the split depended on. The author's response on 2026-07-26 — "it is added here because the other one has not been merged and the infrastructure is necessary for running the code as a whole; I will draft this PR until the other one is merged" — is the first explicit articulation of the **dependency-management and PR-splitting discipline** that the later, cleaner PRs (#54, #55) follow strictly (Section 4.4).

The working tree of the fork's `fpSupport` branch — and of `sin-pr`, which is built on it — carries this work as commit `6274df2` (2026-07-22, "Fix FCMPG/FCMPL/DCMPG/DCMPL: proper NaN handling with 4-branch CG + add FP comparator infrastructure"). Upstream `fpSupport` does **not** carry it: PR #47 was closed unmerged, and the 4-branch comparator work later mutated into the cleaner unary-predicate infrastructure (`IS_NAN`/`IS_ZERO`/`IS_INFINITY` and their negations) that PR #54 relies on. Fork PRs #1 (closed) and #3 (open) mirror the same code for the fork's own staging.

### 3.4 FP Bounds and Domain Fixes (PRs #38–#42)

A second silent unsoundness sat in the *variable-domain* encoding. When a symbolic FP variable was declared, `ProblemZ3BitVector.makeRealVar` encoded its domain with **strict** inequalities (`mkFPGt`/`mkFPLt`). Strict inequalities exclude both `NaN` and `±Inf` from the domain value sets — precisely the values that many FP operations legitimately produce. The solver would therefore fail to satisfy any set of constraints that forced a NaN or infinite result.

PR #38 (2026-06-27) cherry-picked the exploratory commit `9a662f9` from the `experimentalnan` branch and, together with PRs #39/#40 (integration tests) and #41/#42 (cleaned fixes), established several corrections:

- FP variable bounds switched to **non-strict** `mkFPGEq`/`mkFPLEq`, with an `isNaN` predicate and an optional `isInf` predicate OR-ed into the domain (the infinity part gated by the new `symbolic.inf` flag);
- `FDIV`/`DDIV` no longer throw `ArithmeticException` on division by zero — they push the IEEE 754 result (`±Inf` or `NaN`);
- `NumericConstraintTranslator` drops `RealConstraint` instances in the z3str3 transform chain to avoid a `ClassCastException` (a precursor of the string-solver work in Section 6);
- the tests were relocated into a dedicated `fp` test package under `examples/`.

PR #42's review (2026-07-08) contains three directives that directly shape the later encodings: (1) move FP tests under `examples/.../fp`; (2) **make the NaN, infinity, and zero outcomes of `FDIV`/`DDIV` separate choices**, with the last choice being the ordinary non-NaN/non-infinity/non-zero case; and (3) verify that infinity cases are added to *every* creation of a `double`/`float` domain. Directives (1) and (2) are implemented verbatim in PR #54.

### 3.5 Supporting Fixes: ISUB Overflow (#46) and `Math.toRadians`/`toDegrees` (#23/#24)

Two smaller merged contributions round out the foundation:

- **`ISUB` overflow (PR #46, merged 2026-08-13).** With the bit-vector backend configured at 64 bits (`bvlength=64`), integer subtraction overflow silently wraps differently than the JVM's 32-bit `int` semantics. The fix mirrors the existing `IADD` pattern: wrap the subtraction result in `_shiftL(32)._shiftR(32)` to simulate 32-bit overflow inside the 64-bit domain, and replace a `NanoXML` workaround (`sf.push(0, false)`) with the correct concrete result. The review thread (2026-07-19 → 2026-08-12) pushed for regression tests, which were added as symcrete examples under `examples/overflow/`, later consolidated into a single `SubtractionOverflowTest` with boundary-forcing guards (`MIN_VALUE - 1` must wrap to `MAX_VALUE`). The PR went through several test refactors before merging — a pattern repeated throughout this work (see Section 7).
- **`Math.toRadians`/`Math.toDegrees` (Issue #22; #23 closed, #24 merged 2026-01-24).** SV-COMP reported an incorrect result on the `radians` benchmark (Issue #22). The root cause was that the native `toRadians`/`toDegrees` methods escaped symbolic tracking. The fix replaced the native implementations with Java-side implementations (`degrees * PI / 180` style) so the symbolic engine follows the arithmetic. The two-PR history of this small fix also documents the cross-repository confusion between the identically named `sv-comp` branches of Java Ranger and SPF, resolved by targeting the Java Ranger `sv-comp` branch.

### 3.6 The Z3 Translation Layer (PR #53, merged)

PR #53 (merged 2026-09-13) closed a gap in the solver-facing layer: `ProblemZ3` and `ProblemZ3BitVector` were missing several operations needed by the FP work — real-valued multiplication/division branches (with `div` handling `FPExpr` via `mkFPDiv`), integer `rem`/`mod` via `mkRem`/`mkMod`, int-to-real and int-to-FP conversions, `getRealValueInf/Sup` delegating to `getRealValue`, and a `postLogicalOR` that posts a constraint disjunction to the solver. The reviewer merged it with the pragmatic note that it "needed test cases to check that this works fine" but would "probably show up in later PRs if there are issues" — and indeed the FP comparator and division PRs are precisely where those operations are exercised.

---

## 4. Spec-Faithful Division and Remainder

### 4.1 The Design Problematique

The JVM specifies `float`/`double` division (`FDIV`, `DDIV`) and remainder (`FREM`, `DREM`) completely in terms of IEEE 754. For a *symbolic* operand pair, a faithful encoding must partition the entire space of operand values into outcome classes and, for each class, produce (a) a path condition that characterizes it and (b) the result value that the instruction pushes. Early attempts conflated these two concerns, and the review history shows the reviewer repeatedly steering the design:

- **The result must be part of the choices (2026-07-31, on DDIV):** *"You cannot push the results like that... if we are exploring the NaN or the infinity cases then we know that the result can be NaN or Infinity depending on the operation. The result needs to be part of the choices; it should not be globally placed here."* This comment, together with PR #42's "separate choices" directive, is the design seed of the 6-arm encoding.
- **Unreachable arms (2026-09-13):** In the ITE-based version, the NaN arm was *structurally unreachable* because the result arm was bound to its own symbolic variable — a `res != res` NaN test could never be satisfied, so the NaN branch was skipped. The reviewer's instruction was to bind the NaN outcome with *only the conditions that make the result NaN* — a set of disjunctions on the operand classes — and to push a concrete `NaN` on the stack for that arm.
- **Pattern discipline (2026-09-08):** *"I flattened the constraints to use regular disjunctions instead of ITEs; this should be easier to deal with / easier to read."* The final design deliberately has **flat, disjunction-only guard formulas** — no nested `ITE` trees — so every arm is a conjunction `(operand-class predicates) ∧ (result == class-representative)`, and the path condition stays in the same form as the rest of veritesting.

### 4.2 The Six-Arm Division Encoding (PRs #45 → #54)

The current `FDIV`/`DDIV` implementation (branch `fp-fdiv-inf-nan-support`, PR #54) splits symbolic division into **six outcome classes**, each derived solely from the classes of the two operands:

| # | Outcome | Operand-class guard | Result binding |
|---|---------|--------------------|----------------|
| 1 | `NaN` | `isNaN(A) ∨ isNaN(B) ∨ (isZero(A) ∧ isZero(B)) ∨ (isInf(A) ∧ isInf(B))` | concrete `NaN` pushed |
| 2 | `+Inf` | `(isZero(B) ∧ ¬isZero(A) ∧ ¬isInf(A)) ∨ (isInf(A) ∧ ¬isInf(B) ∧ ¬isZero(B)) ∨ (isInf(A) ∧ isZero(B))` | classified result pushed |
| 3 | `−Inf` | mirror of arm 2 with signs | classified result pushed |
| 4 | `+0` | `(isZero(B) ∧ ¬isZero(A) ∧ ¬isInf(A))` (sign-resolved) | classified result pushed |
| 5 | `−0` | sign-resolved zero | classified result pushed |
| 6 | normal quotient | `¬isNaN(A) ∧ ¬isNaN(B) ∧ ¬isZero(B) ∧ ¬isInf(A) ∧ ¬isInf(B)` | `result == A / B` (Z3 `fp.div`) |

Mechanically, `FDIV.execute()`:

1. creates a `PCChoiceGenerator` (6-way) when either operand is symbolic — its `select(choice)` mirrors the concrete replay so Veritesting's deja-vu states see the right operand stack;
2. computes the operand-class predicates lazily per arm through the new `FPClassExpr` Green leaf (Section 4.3);
3. conjoins the chosen arm's guard with `result == <class representative>` and posts it via `pc._addDet(new GreenConstraint(identity))` — with the *identity constraint* being a problem-name binding that keeps every arm's formula flat;
4. pushes the concrete class representative on the operand stack so downstream concrete execution proceeds; and
5. falls back to ordinary concrete division when *both* operands are concrete.

The observable effect, verified with a both-operands-symbolic example: the run terminates in **7 paths with no dead arms** — the five special outcomes, plus the ordinary class, which the program's own `res == 0.0f` check splits into zero and non-zero. (The "9 paths" the reviewer reported earlier were reproduced as 7 by the author; the discrepancy, documented in the PR thread, comes from the engine replaying a concrete arm in addition to the symbolic exploration, which is not another outcome class.)

The design also required the **`symbolic.inf`** flag (default off): when disabled, the FP domain excludes infinities/`NaN` exactly as before, so existing non-FP behavior is byte-for-byte unchanged; when enabled, the widened domain lets the outcome arms actually be decided.

### 4.3 `FPClassExpr`: a First-Class Predicate Leaf

To express "which class is this operand in?" *inside* a Green formula, PR #54 adds a new expression node, `FPClassExpr`, to the Green expression layer (the underlying constraint library). It pairs a Green operand (typically a `RealVariable`) with a `Comparator` from the SPF numeric layer (`IS_NAN`, `IS_ZERO`, `IS_INFINITY`, and their negations). Because the Green `Visitor` interface is a compiled, closed hierarchy, the node dispatches at `accept(Visitor)` time to the SPF's own `GreenPbTranslator` via an `instanceof` check, calling `postVisitFPClass` to emit the Z3 FP predicate (`mkFPIsNaN`, `mkFPIsZero`, `mkFPIsInfinite`, etc.); for any other visitor it falls through to the no-op base chain, keeping the node safe with the rest of the codebase. This single leaf unifies the operand-class vocabulary used by both `FDIV`/`DDIV` and the `FREM`/`DREM` draft.

### 4.4 The Three-Arm Remainder Encoding (Draft PR #55)

With division settled, remainder was next. First-generation `FREM` (PR #44, merged into the fork's `fpSupport` via fork PR #2) followed the older `FDIV` pattern (choice generator for the division-by-zero guard, a created symbolic `_rem` expression) — a design that the division experience had already shown to be superseded. The draft PR #55 (`frem-drem-rem`) re-implements remainder on the *division* infrastructure, with three arms that partition every pair of operands:

| # | Outcome | Guard | Result binding |
|---|---------|-------|----------------|
| 1 | `NaN` | `isNaN(A) ∨ isNaN(B) ∨ isZero(B) ∨ isInf(A)` | concrete `NaN` pushed |
| 2 | `A` (dividend) | `isInf(B) ∧ ¬isNaN(A) ∧ ¬isInf(A)` | `result == A` |
| 3 | `A % B` | `¬isNaN(A) ∧ ¬isNaN(B) ∧ ¬isZero(B) ∧ ¬isInf(A) ∧ ¬isInf(B)` | `result == fp.rem(A, B)` |

This matches the JVM exactly: `x % 0`, `x % NaN`, `NaN % y`, and `Inf % y` give `NaN`; an infinite divisor preserves the finite dividend; everything else is the exact IEEE 754 remainder (quotient rounded toward zero, sign of the dividend) computed by Z3's `fp.rem`.

On the solver side, `ProblemZ3BitVector.rem` gains FP dispatch to `ctx.mkFPRem`, plus mixed concrete/symbolic overloads `rem(double, Object)` and `rem(Object, double)` mirroring the FP `div` methods; `GreenPbTranslator` now maps the Green `MOD` operator to `fp.rem` for floating-point operands (previously `MOD` only had the bit-vector power-of-two shortcut). PR #55 also fixes the **concrete path**: the old stubs threw `ArithmeticException` on a zero divisor even for fully concrete operands, whereas the JVM yields `NaN`; the concrete path now delegates to the base instruction.

PR #55 is a **draft** for a principled reason: it is built on infrastructure that only exists inside PR #54 (`FPClassExpr`, the unary FP class comparators, `postVisitFPClass`) and is therefore un-compilable until #54 merges. By writing the FREM/DREM files *against* the division encoding, the diff stays limited to the remainder-specific work — the PR-splitting discipline from Section 3.3 applied strictly.

### 4.5 What the Closed-PR History Adds

The sequence #38 → #39 → #40 → #41 → #42 → #44 → #43 → #47 → #45 → (#54, #55) is a textbook illustration of iterative design under review:

- #38/#39/#40: establish *that* a bug exists and *how to test* the fix (integration tests with the concrete/symbolic matrix `CC`/`CS`/`SC`/`SS`);
- #41/#42: fix the underlying domain/bounds unsoundness and introduce the "separate choices" principle;
- #44: first-generation remainder — correct in spirit (symbolic `_rem`), wrong in mechanism (global result placement);
- #45: the 6-arm ITE version — correct structure, wrong control flow (`NaN` unreachable, ITE-heavy formulas);
- #54/#55: the disjunction-only, concrete-replay, dependency-annotated final shape.

Each closed PR is a documented counterexample that made the next design decision defensible. This is the value of preserving closed PRs, and one of the report's central methodological points.

---

## 5. Symbolic Support for `Math.sin()`

### 5.1 Why `sin` Is Different

Unlike division/remainder, `Math.sin` is not an SMT-theory operation at all: neither Z3's FP theory nor the bit-vector backend has it. The naive fallback — execute `sin` concretely whenever the argument is concrete and crash otherwise — was the original behavior ("Math.sin not supported"). Three approaches were tried, in order.

### 5.2 Approach 1: CEGIS Concretization (PR #48, `PLAN.md`)

The first design (branch `fp-math-sin-support`, PR #48, closed 2026-07-25; fully documented in `PLAN.md`) implemented a **counterexample-guided inductive synthesis** (CEGIS) loop:

- **Phase 1 (abstract):** `pb.sin(arg)` creates a fresh FP variable `_sin_N` constrained to `[-1, 1]` — a sound over-approximation of the range — and records `(_sin_N, arg, sortBits)` in a pending list.
- **Phase 2 (concretize):** inside `ProblemZ3BitVector.solve()`, after the first SAT, each pending sin is resolved: the argument is evaluated from the model as a concrete double (via IEEE 754 bit reconstruction, `evalArgAsDouble`/`fpNumToDouble`), `Math.sin(concreteArg)` is computed, the assignment `_sin_N == sin(concreteArg)` is added, and the query is re-solved. If the strengthened query is SAT the model is genuine; if UNSAT the offending input is excluded by predicate and the loop retries (up to 15 attempts). A second phase tries 23 hard-coded sample arguments.

The CEGIS approach worked on 14/14 constructed tests, but four bugs/fixes documented in `PLAN.md` show its fragility:

1. **NaN handling:** Z3 returns `NaN` for unconstrained FP variables; `mkFPEq(x, NaN)` is *always false* under IEEE 754, so value-exclusion fails — one must use the `mkFPIsNaN(x)` *predicate*;
2. **Subnormal trap:** Z3 prefers subnormal values where `sin(x) ≈ 0`; excluding individual values is hopeless against ~2^50 subnormals — fixed with the sample-seeding phase;
3. **Sample x-constraint:** the sample phase must post `argExpr == sample` (not merely `_sin_N == sin(sample)`) or the solver produces models where `x` contradicts the path condition;
4. **Attempt count:** `MAX_CEGIS_ATTEMPTS` raised from 5 to 15.

The design was *sound for bug-finding* (a spurious model may prune a real path, never introduce a fake one) but slow and solver-fragile. It was superseded rather than merged.

### 5.3 Approach 2: Taylor-Series Peer (branch `fp-taylor-sin`)

A single-commit experiment (`5235320`, "feat(fp): Math.sin() via Taylor series peer approximation") replaced the CEGIS loop with the closed-form approximation `sin(x) ≈ x − x³/3! + x⁵/5! − …` rendered as a symbolic polynomial. This yields a purely *solver-native* formula (no meta-level loop), but a truncated Taylor series is only accurate near 0 and diverges sharply approaching `±π/2`, so the approximation error pollutes the path condition precisely where decisions matter. The branch was abandoned in favor of a piecewise construction.

### 5.4 Approach 3: Piecewise-Linear ITE Encoding (current work)

The current approach (branches `fp-sin-curvature-weighted` and the local `sin-pr`) models `sin` on its principal arc `[−π/2, +π/2]` as a **piecewise-linear approximation** — 16 secant segments, each turned into a single affine leaf `y = m·x + b` (one `fp.mul` + one `fp.add`, minimal circuit depth for Z3's bit-vector FP backend), organized into a **nested ITE tree**. The design, captured in the javadoc of the new `numeric/PiecewiseLinearSin` class, makes four engineering choices:

1. **Curvature-weighted breakpoints.** Segments are denser near `±π/2`, where `|sin″(x)| = |sin(x)|` (curvature) is maximal, and sparser near 0 where `sin` is nearly linear. Breakpoints come from the inverse CDF of the `|sin|` density (`cumulative(x) = ∫|sin| dt`), so every segment carries an equal share of total curvature.
2. **PC-bound pruning.** Before building the tree, the path condition is scanned for interval constraints on the symbolic argument; only segments overlapping the PC-implied interval are emitted. Unreachable segments vanish, dramatically cutting Z3 solve time (the `fp.eq` checks on the ITE tree were the known bottleneck — at 16 segments the unpruned tree timed out).
3. **Polarity split.** Negative and positive half-segments become separate ITE sub-trees combined under a single `x < 0` guard, so Z3 can discard a whole half as soon as the PC constrains the sign.
4. **Structural clamp.** The result is clamped into `[-1, 1]`, so secant extrapolation near the boundaries cannot produce out-of-codomain values that would act as spurious counterexamples.

The class's own documentation is scrupulous about the **limitations**: it is a pointwise approximation, not an enclosure (per-segment error `O(width²/8)` reaches the solver only through the segment count, so a guard very close to a segment boundary can be decided the wrong way); arguments outside `[−π/2, +π/2]` are not reduced (no periodicity handling); and NaN/Inf symbolic arguments are not modeled.

The branch history shows the incremental experiments: `7f00821` (piecewise-linear peer via symbolic ITE) → `d5ca6d5`/`85f2fad`/`e922666` (a `symbolic.mantissa` flag defining a custom **reduced-significand FP sort**, with a warning when it is actually used and auto-registration from `FpSortUtil`) → `9a8b430` (polarity-split ITE trees) → `282e8db` (curvature-weighted breakpoints, "Branch B experiment") → `cfe130e` (structural clamp). The reduced-mantissa experiment — shrinking the significand so Z3's FP engine handles the ITE tree faster — was ultimately **reverted**, along with the FP-comparator infrastructure and the test-package moves, in the six `Revert …` commits at the head of `sin-pr`. The local working tree then refactors the implementation into its current minimal shape: the ~200-line inline encoding is extracted from the `JPF_java_lang_Math` peer into the standalone `PiecewiseLinearSin` class, and the ad-hoc `CMP` operator used as an ITE guard is replaced by a dedicated, semantically correct `LT` operator, wired through `Operator`, `BinaryRealExpression`, and `PCParser`. Fourteen end-to-end tests (`TestSin*`) exercise feasible/infeasible branches, guard decisions, positivity/negativity, range bounds, nesting, and imposed-branch behaviour.

This trajectory — *meta-level CEGIS → closed-form Taylor → solver-native piecewise ITE; and within piecewise: uniformity → curvature weighting, one tree → polarity split, no pruning → PC-bound pruning, full precision → reduced mantissa (reverted)* — is the report's clearest single narrative of hypothesis-and-revision.

---

## 6. Cross-Cutting Integration

### 6.1 Symbolic FP × String Solver (PR #37, open)

PR #37 (open since 2026-06-22) documents a genuine incident: combining `symbolic.fp=true` with the z3str3 string solver crashed with a `ClassCastException` (followed, once fixed, by a `NumberFormatException`). The post-mortem in the PR body isolates the root cause precisely:

- The string solver aliases the *entire* numeric path condition (`StringPathCondition → Translator.translate → manager.numCons.collect(npc)`), not just string-relevant constraints; `IntermediateConstraint.transform` unconditionally casts both operands to `(IntegerExpression)`, so a `SymbolicReal`/`RealConstant` operand under the string tree throws.
- The first attempt guarded by constraint *class* (`instanceof RealConstraint`) — the wrong axis (`MixedConstraint` with real operands still crashed). The minimal fix drops any node whose operand is a `RealExpression` — 5 lines, no imports, subsumes the hack.
- A secondary fix in `ProblemZ3BitVector.getFPValue`: Z3 prints negative FP model values as `(- 2052)`; `Double.parseDouble` rejects `(-`, so a `(- N) → -Double.parseDouble(inner)` branch was added (commit `d6ab2b4`, "parse negative FP numerals rendered as `(- N)`").
- Verification: four SV-COMP benchmarks (`float`, `double2long`, `GraphFragment_false`, `ExSymExeFNEG_false`) reach their expected violations at comparable cost.

The PR's value is dual: it fixes a concrete integration bug and it enforces architectural hygiene (the string and numeric solver trees are separate worlds; reals must never be routed through the z3str3 translation).

### 6.2 Configuration Surface and Test Methodology

The work introduces a growing, orthogonal set of configuration flags, each gating a distinct soundness improvement so that default behavior is byte-for-byte unchanged:

| Flag | Introduced by | Meaning |
|------|---------------|---------|
| `symbolic.fp` | (pre-existing in fpSupport history) | Master switch for FP symbolic execution |
| `symbolic.inf` | comparator work / #42 / #54 | Widen FP domains with `NaN`/`±Inf` (default off) |
| `symbolic.mantissa` | `sin` experiment (`sin-pr`) | Reduced-significand FP sort for faster FP solves (reverted) |
| `symbolic.strings`, `symbolic.string_dp=z3str3` | (pre-existing) | String solver; must not see real operands (PR #37) |

The **symcrete test pattern** — a concrete/symbolic operand matrix (`CC`, `CS`, `SC`, `SS`) with expected IEEE 754 results per outcome class, run under `RunJPF` with a `.jpf` config — appears consistently across `TestDiv`, `TestDDiv`, `TestFREM`, `TestDREM`, `TestSin*`, and the overflow tests. It is the community's answer to the reviewer's recurring demand, "please include tests for these changes," and it is what makes regressions in the IEEE 754 edge cases (`NaN` operand, infinite divisor/dividend, signed zero) observable at all.

---

## 7. Review Process and Working Practices

The PR threads give a rare, first-person record of how this work was steered by its reviewer (Dr. Soha Hussein, co-author of Java Ranger). Recurring directives, all of which the author adopted over time:

1. **Minimal diffs.** *"The PR should only include new/modified code. Please avoid deleting and rewriting the same code... Also, please avoid making space-editing changes to existing code."* (PR #47, 2026-07-26; repeated on #45, e.g., "why adding spaces, changing format?"). The later PRs (#53, #54 statement "no unrelated files are touched", #55 "the diff is the FREM/DREM encoding, the two solver touch points, and the tests") are the proof of adoption.
2. **Split large PRs.** PR #47 was explicitly required to be split (41 files); its successor was drafted until its dependency merged. #45 was closed and reopened as #54 because the accumulation of comments made the thread unmanageable; #55 is written *against* #54 by construction so the remainder diff stays small.
3. **Tests for every change.** "please include the test case for checking these changes" (#46) → the symcrete matrix became standard.
4. **Sound semantics, not syntactic patches.** The reviewer repeatedly pushed from "push a result" toward "the result is part of the choices," from ITE-tree formulas toward flat disjunctions, and from "conditions making the result NaN" toward "the class of the operands." The final encodings are exactly these.
5. **Reproducibility.** A test `.jpf` config "dependent on your machine" was flagged and generalized (2026-09-13); the author "generalized" (2026-09-15) — the witness/path-numbering discussion ("9 paths" vs "7 paths") was similarly grounded in run logs rather than claims.
6. **Explanation of pre-existing code.** Comments asked "this is not new code, right?" or "why is this here?" — answered by the author by locating the origin (e.g., the lexicographic `compareTo` "originally added by Corinus") and, in the cleanest cases, by *reverting* the file to byte-identical base state (the `style(fp): realign … to original commit bytes` commits) and keeping only the true delta.

These practices are not trivia; they are the operational half of the thesis argument this report documents: *spec-faithful symbolic FP semantics is achievable incrementally, provided each step is a minimal, reviewable, test-carrying diff that preserves default behavior.*

---

## 8. Current Status and Future Work

**Merged into `fpSupport`/`svcomp` (4):** IEEE 754 special values (#31), Java-based `toRadians`/`toDegrees` (#24), `ISUB` overflow (#46), Z3 translation layer (#53). The fork's working `fpSupport` branch additionally carries, via direct commits and fork-internal merges **not yet promoted upstream**: the 4-branch FP comparator infrastructure with NaN-aware `FCMPG`/`FCMPL`/`DCMPG`/`DCMPL`, non-strict FP bounds with the `isNaN`/`isInf` domain predicates (`symbolic.inf`), and the first-generation `FREM`.

**Open (2):** PR #54 (`FDIV`/`DDIV`, 6-arm disjunction-only, ready for review) and PR #37 (symbolic FP × string solver).

**Drafted (1):** PR #55 (`FREM`/`DREM`, 3-arm, on top of #54).

**In progress:** the `sin-pr` branch refactor of the piecewise-linear `Math.sin` encoding (extraction into `PiecewiseLinearSin`, `LT` operator) — the eventual sin PR will be the minimal, self-contained continuation of this line.

Immediate next steps, in dependency order:

1. Merge #54; then promote #55 out of draft (it compiles only against #54's `FPClassExpr` infra).
2. Land #37 (string-solver integration).
3. Complete the `Math.sin` PR from `sin-pr`, with the documented limitations (no argument reduction, no NaN/Inf modeling, pointwise-not-enclosure error) stated as known scope rather than silent gaps.
4. Consider the open extensions recorded across the work: argument reduction for `sin` beyond `[−π/2, π/2]`, a formal error-enclosure (interval) variant, FP-aware handling for the remaining transcendental/library methods, and additional SV-COMP seed benchmarks for the FP domain (e.g., the `radians` family of Issue #22).

---

## Appendix A — Key Files

**Expression/numeric layer** (`jpf-symbc/src/main/gov/nasa/jpf/symbc/numeric/`): `RealExpression`, `RealConstant`, `SymbolicReal`, `BinaryRealExpression`, `Operator` (incl. new `LT`), `PCParser`, `PathCondition`, `Constraint`, `Comparator` (unary FP class predicates), `RealSpecialConstant`, `RealNaN`, `RealInfinity`, `RealZero`, `FpSort`, `FPClassExpr` (PR #54), `PiecewiseLinearSin` (sin PR).

**Solver layer** (`numeric/solvers/`): `ProblemGeneral`, `ProblemZ3`, `ProblemZ3BitVector`, `ProblemZ3BitVectorIncremental` (FP-aware `div`/`eq`/`neq`/`rem`, non-strict bounds, `getFPValue` negative-numeral fix).

**Bytecode** (`bytecode/`): `FDIV`, `DDIV`, `FREM`, `DREM`, `FCMPG`, `FCMPL`, `DCMPG`, `DCMPL`, `ISUB`; **peer** `peers/.../JPF_java_lang_Math`; **factory** `SymbolicInstructionFactory` (`symbolic.inf`).

**Translation:** `GreenPbTranslator` (`postVisitFPClass`), `NumericConstraintTranslator` (skip reals for z3str3).

**Tests** (`examples/gov/nasa/jpf/symbc/fp/`): `TestFDiv`/`TestDDiv`/`TestFREM`/`TestDREM`, `TestSin*` (14), overflow examples under `examples/overflow/`.

## Appendix B — Methodological Summary (one paragraph)

This chapter has reconstructed a year of floating-point work on the `fpSupport` branch of Java Ranger from three complementary records: the merged pull requests (what landed), the closed pull requests (what was tried and why it was superseded), and the working-tree git state (what is in flight). Taken together, they document a consistent method: *make default behavior untouched; state soundness and approximation; separate outcome classes into explicit choices; prove each idea with a small, reviewable, test-carrying diff; and preserve the closed attempts as the history of the design.* The result is a spec-faithful, incrementally developed foundation for floating-point symbolic execution that future contributors can extend commit-by-commit.
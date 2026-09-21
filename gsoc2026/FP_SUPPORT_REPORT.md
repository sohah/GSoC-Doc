# Floating-Point Symbolic Execution Support in Java Ranger

**Author:** Salmane Khalili
**Date:** September 2026
**Scope:** the `fpSupport` line of work in `SalmaneKhalili/java-ranger` (fork of `vaibhavbsharma/java-ranger`), upstreamed as pull requests against `vaibhavbsharma/java-ranger`. The record is the git history and PRs of both repositories, including merged, open, draft, and closed PRs.

This report covers the motivation, the work done, the approaches that were tried and superseded, and the current status of floating-point (FP) symbolic execution support in Java Ranger, the path-merging extension of Symbolic PathFinder (SPF).

---

## 1. Motivation

### 1.1 The unsoundness and the crashes

The JVM's floating-point instructions and `java.lang.Math` follow IEEE 754. The symbolic engine did not. Four concrete problems drove the work:

1. **Special values were unrepresentable.** The expression layer (`RealExpression`, `RealConstant`, `SymbolicReal`) modeled every floating-point value as an ordinary real. NaN, positive/negative infinity, and signed zero, which IEEE 754 operations legitimately produce (for example `0.0 / 0.0`, `1.0 / 0.0`, overflow, underflow), could not be represented or distinguished, so the resulting constraints were unsound.
2. **Comparisons ignored the unordered case.** `FCMPG`/`FCMPL`/`DCMPG`/`DCMPL` push -1 or +1 when an operand is NaN, depending on the instruction. The comparison choice generator handled only two branches (`<` versus `>=`), so NaN paths were missed. Equality is not reflexive in IEEE 754 (`NaN != NaN`), which a solver treating equality as syntactic identity gets wrong.
3. **Division and remainder by zero were modeled as exceptions.** The JVM defines `x / 0.0` as `+/-Inf` (or `NaN` for `0.0 / 0.0`) and `x % 0.0` as `NaN`; no exception is thrown. `FDIV`/`DDIV` threw `ArithmeticException` on a zero divisor, and `FREM`/`DREM` failed with `SYMBOLIC FREM not supported` for a symbolic operand.
4. **`Math.sin` was unsupported.** `sin` is not an operation of any SMT theory used by the backends (Z3 FP, bit-vector, string). A symbolic call to `Math.sin` crashed instead of producing a constraint.

### 1.2 The concrete drivers

- [Issue #22](https://github.com/vaibhavbsharma/java-ranger/issues/22), "Incorrect Result - radians": Java Ranger reported a wrong verdict on the SV-COMP `radians` benchmark. Root cause: the native `Math.toRadians`/`Math.toDegrees` methods escaped symbolic tracking, so the arithmetic was not part of the path condition.
- [Issue #27](https://github.com/vaibhavbsharma/java-ranger/issues/27), "Add IEEE-754 special values": without special-value constants, paths that produce NaN/infinity are modeled as normal arithmetic. The issue names the affected SV-COMP benchmarks: `CWE369_Divide_by_Zero` (division by a symbolic value that can be zero yields infinity or NaN) and `Adjustable-Inner-Count-Halving` (repeated halving can underflow to zero).
- [Issue #29](https://github.com/vaibhavbsharma/java-ranger/issues/29), "Introduce isNaN predicate": comparisons need a way to test for NaN (the `fcmpg`/`fcmpl` unordered case, and user-level `Float.isNaN`/`Double.isNaN`); benchmarks such as `float1` get wrongly constrained path conditions without it.
- `Math.sin` crash: any SV-COMP-style benchmark calling `Math.sin` on a symbolic argument terminated in an unsupported-operation error.

---

## 2. What I Did

Grouped factual items. Each links its PR and states its status. Closed PRs are kept to a brief history; their rationale and why each was abandoned sit in Section 3.

### 2.1 IEEE-754 special value constants

- [PR #31](https://github.com/vaibhavbsharma/java-ranger/pull/31), merged 2026-03-16 into `fpSupport` (commit `b1ae02c`, `+381/-0`, 8 files). Added `FpSort` (float/double precision), `RealSpecialConstant` base class, and `RealNaN`, `RealInfinity`, `RealZero` with sign flags and bit-level negative-zero storage, extending the `RealConstant` hierarchy. This makes special values first-class in the expression layer so the later encodings can reason about them. The reviewer requested retargeting the PR from `master` to `fpSupport`, which was done.

### 2.2 isNaN predicate

- [PR #36](https://github.com/vaibhavbsharma/java-ranger/pull/36), closed. Implemented `RealIsNaN` for Issue #29; the functionality was absorbed into the unary FP comparator infrastructure of the comparison work (Section 3.2).

### 2.3 FP variable bounds and division-by-zero semantics

- [PR #38](https://github.com/vaibhavbsharma/java-ranger/pull/38), closed, and [PRs #39](https://github.com/vaibhavbsharma/java-ranger/pull/39)/[#40](https://github.com/vaibhavbsharma/java-ranger/pull/40), closed. Cherry-picked the NaN/infinity division-by-zero fix from the `experimentalnan` branch (commit `9a662f9`) and added FDIV/DDIV integration tests under `symbolic.fp=true`. #38 closed because its `master` base produced a huge diff.
- [PR #41](https://github.com/vaibhavbsharma/java-ranger/pull/41) (closed, dirty base) and [PR #42](https://github.com/vaibhavbsharma/java-ranger/pull/42) (closed, cleaned). FP variable bounds became non-strict with `isNaN`/`isInf` domain predicates OR-ed in, `FDIV`/`DDIV` stopped throwing on division by zero, the `symbolic.inf` flag was added, and `RealConstraint` was dropped from the z3str3 chain (a precursor to #37). #42's review established that division outcomes must be separate choices (Section 3.1).

### 2.4 NaN-aware comparisons

- [PR #43](https://github.com/vaibhavbsharma/java-ranger/pull/43), closed. NaN-aware 5-way branch (LT/EQ/GT/NaN/inf) with IEEE 754 result values and FP-aware `eq`/`neq` via Z3 `mkFPEq`, so `NaN != NaN` is satisfiable. Superseded by the consolidated comparison PR.
- [PR #47](https://github.com/vaibhavbsharma/java-ranger/pull/47), closed. Consolidated 4-branch comparison (`<`, `==`, `>`, unordered) plus the unary FP comparator vocabulary (`IS_NAN`/`IS_INF` family). Closed because the reviewer required a split, minimal-diff PR; the infrastructure was committed to the fork's `fpSupport` (fork PR #1, merged) and staged again as [fork PR #3](https://github.com/SalmaneKhalili/java-ranger/pull/3) (open). This vocabulary is what PRs #54/#55 build on (Section 3.2).

### 2.5 Math.toRadians / Math.toDegrees

- [PR #23](https://github.com/vaibhavbsharma/java-ranger/pull/23), closed (merge conflicts; superseded by #24), and [PR #24](https://github.com/vaibhavbsharma/java-ranger/pull/24), merged 2026-01-24 into `svcomp`. Replaced the native implementations with Java-side `toRadians`/`toDegrees` so the symbolic engine tracks the arithmetic, fixing Issue #22's `radians` benchmark.

### 2.6 Division: spec-faithful FDIV/DDIV

- [PR #45](https://github.com/vaibhavbsharma/java-ranger/pull/45), closed 2026-09-16. The six-arm `FDIV`/`DDIV` outcome encoding (NaN, `+Inf`, `-Inf`, `+0`, `-0`, normal quotient), evolved from ITE to flat disjunctions; closed over accumulated review debt and reopened as #54 (Section 3.1).
- [PR #54](https://github.com/vaibhavbsharma/java-ranger/pull/54), open. Current, review-ready six-arm encoding (`fp-fdiv-inf-nan-support`, `+1094/-146`, 14 files), each arm a flat conjunction of operand-class predicates and one result equality; also introduces the `FPClassExpr` Green leaf and the `symbolic.inf` flag (Section 3.1).

### 2.7 Remainder: symbolic FREM/DREM

- [PR #44](https://github.com/vaibhavbsharma/java-ranger/pull/44), closed. First-generation `FREM` following the old `FDIV` pattern (symbolic `_rem` expression); superseded by the division-infrastructure rewrite. Staged in [fork PR #2](https://github.com/SalmaneKhalili/java-ranger/pull/2), merged.
- [fork PR #4](https://github.com/SalmaneKhalili/java-ranger/pull/4), open: the split, upstream-ready remainder PR.
- [PR #55](https://github.com/vaibhavbsharma/java-ranger/pull/55), draft. The three-arm `FREM`/`DREM` encoding on top of PR #54's infrastructure (see Section 3.1), plus Z3 `fp.rem` support in the solver layer.

### 2.8 Math.sin support

- [PR #48](https://github.com/vaibhavbsharma/java-ranger/pull/48), closed. `Math.sin` via CEGIS concretization (14/14 tests passing, documented in `PLAN.md`); superseded by the piecewise-linear encoding now in development (Section 3.3).
- Current piecewise-linear encoding in development on the local `sin-pr` branch (no upstream PR yet).

### 2.9 Supporting fixes

- [PR #46](https://github.com/vaibhavbsharma/java-ranger/pull/46), merged 2026-08-13 into `fpSupport`. Added 32-bit overflow simulation to `ISUB` when the bit-vector backend runs at `bvlength=64` (wrap the subtraction in `_shiftL(32)._shiftR(32)`, mirroring `IADD`), and replaced a `NanoXML` workaround (`sf.push(0, false)`) with the correct concrete result. Regression tests were consolidated into `SubtractionOverflowTest` with boundary-forcing guards.
- [PR #53](https://github.com/vaibhavbsharma/java-ranger/pull/53), merged 2026-09-13 into `fpSupport` (commit `66a1e17`). Implemented missing Z3 operations in `ProblemZ3`/`ProblemZ3BitVector`: real multiply/divide (with FP `div` via `mkFPDiv`), integer `rem`/`mod` via `mkRem`/`mkMod`, int-to-real and int-to-FP conversions, `getRealValueInf`/`Sup`, and `postLogicalOR` for posting constraint disjunctions. The FP comparator and division PRs depend on these.
- [PR #37](https://github.com/vaibhavbsharma/java-ranger/pull/37), open since 2026-06-22. Fixed the `ClassCastException`/`NumberFormatException` when `symbolic.fp=true` is combined with the z3str3 string solver: the string solver aliases the entire numeric path condition, and the translation cast real operands to `IntegerExpression`. The minimal fix (commit `eaa207c`) drops any constraint node whose operand is a `RealExpression`; commit `d6ab2b4` parses Z3 negative FP model numerals rendered as `(- N)`.

---

## 3. Approaches

What was actually tried, in order, and why each step was abandoned or superseded.

### 3.1 Division and remainder outcome encodings

1. **Exploratory start.** `experimentalnan` branch, commit `9a662f9` ("push and solve correct JVM operand-stack/expression for NaN and division per 0 semantics"). Correct results for `0/0` and `x/0`, but the result was pushed globally rather than chosen per branch.
2. **Bug + tests (PRs #38-#40).** Established that a bug exists and how to test it: integration tests with concrete/symbolic operand matrices (CC/CS/SC/SS) under `symbolic.fp=true`.
3. **Domain fix (PRs #41-#42).** Non-strict FP bounds with `isNaN`/`isInf` domain predicates, no `ArithmeticException` on division by zero. The review of #42 pushed the principle that the NaN/infinity/zero outcomes of division must be separate choices, with the ordinary case last.
4. **Six-arm encoding (PR #45, then #54).** `FDIV`/`DDIV` split into six outcome classes derived from operand classes: NaN (either operand NaN, zero/zero, inf/inf), `+Inf`/`-Inf`, `+0`/`-0`, and the ordinary quotient `result == A / B`.
   The encoding evolved through three forms on the #45 branch: the initial 6-arm choice generator (`90b00c6`), an ITE-based result encoding using new `FPClassExpr` predicates (`0b0614c`), and finally a flat "Phi" encoding with no nested ITEs (`129cf84`). Reviewer feedback drove the shape: "the result needs to be part of the choices; it should not be globally placed here", the NaN arm must be bound by "only the conditions that will make the result a NaN ... a set of disjunctions", and (from the author) "i flattened the constraints to use regular disjunctions instead of ITEs; this should be easier to deal with/easier to read".
   A real bug surfaced in review: in the ITE version, the NaN arm was unreachable when both operands were symbolic, because the result was bound to its own symbolic variable, so `res != res` was unsatisfiable ("I tested this where both arguments are symbolic and it is unsat"). The fix (commit `1b76aae`) binds the NaN arm with operand-class disjunctions only and pushes a concrete NaN, making the branch genuinely explored.
   PR #45 accumulated enough review debt (spacing churn, a machine-dependent test config, the unsat NaN arm) that it was closed and reopened as PR #54 on 2026-09-16 ("the other PR has gotten too messy for me to track your comments. I've reopened it here with a major refactor"). On the both-symbolic example, the reviewer reported 9 paths; the author could not reproduce that and reports 7 paths with no dead arms (five special outcomes, plus the ordinary class split by the program's own `res == 0.0f` check).
5. **Remainder: first generation (PR #44), then three-arm (PR #55).** First-generation `FREM` followed the old `FDIV` pattern (symbolic `_rem` expression, div-by-zero choice generator) and was superseded once division had its outcome-class design. Draft PR #55 (`frem-drem-rem`) re-implements remainder on the division infrastructure with three arms that partition every operand pair: NaN (`isNaN(A)` or `isNaN(B)` or `isZero(B)` or `isInf(A)`), dividend (`isInf(B)` with finite non-NaN `A`), and `A % B` via Z3 `fp.rem` otherwise. This matches the JVM: `x % 0`, `x % NaN`, `NaN % y`, and `Inf % y` give NaN; an infinite divisor preserves the finite dividend. The PR is a draft because it compiles only against PR #54's `FPClassExpr` infrastructure, which has not merged yet.

### 3.2 Comparison handling

1. **Original:** a 2-branch choice generator, which ignores the IEEE 754 unordered case.
2. **PR #43:** a 5-way branch (LT/EQ/GT/NaN/inf) with correct result values, plus `mkFPEq`-based `eq`/`neq` so `NaN != NaN` is expressible.
3. **PR #47:** the consolidated design: a 4-branch choice generator (`<`, `==`, `>`, unordered) and a unary FP comparator vocabulary (`IS_NAN`/`NOT_IS_NAN`/`IS_INF`/`NOT_IS_INF`) that later grew into `IS_NAN`, `NOT_IS_NAN`, `IS_INFINITY`, `NOT_IS_INFINITY`, `IS_ZERO`, `NOT_IS_ZERO`. The reviewer required a smaller, split PR and minimal diffs; the PR was closed, and the infrastructure was committed to the fork's `fpSupport` (commit `6274df2`, also fork PR #1 merged 2026-07-22) and staged as the split fork PR #3 (open).
4. **Current use:** the unary FP class predicates are the guard vocabulary of the division/remainder encodings in PRs #54 and #55 (via `FPClassExpr` and `GreenPbTranslator.postVisitFPClass` mapping to Z3 `mkFPIsNaN`/`mkFPIsZero`/`mkFPIsInfinite`).

### 3.3 Math.sin

1. **CEGIS concretization (PR #48, branch `fp-math-sin-support`, commit `dc5f994`).** `sin(arg)` creates a fresh FP variable `_sin_N` constrained to `[-1, 1]` (a sound over-approximation), records it as pending, and after the first SAT evaluates the argument from the model as a concrete double, posts `_sin_N == Math.sin(concreteArg)`, and re-solves; if UNSAT, an exclusion predicate prunes that value and the loop retries (up to 15 attempts), with a fallback phase that tries 23 hard-coded sample arguments. Four failures found during development, documented in `PLAN.md`: Z3 returns NaN for unconstrained FP variables and `mkFPEq(x, NaN)` is always false, so exclusion must use the `mkFPIsNaN` predicate; Z3 prefers subnormal values where `sin(x)` is near 0, and excluding individual values is hopeless against many subnormals (fixed by sample seeding); the sample phase must constrain `argExpr == sample` or the model contradicts the path condition; the attempt count had to be raised from 5 to 15. The approach was sound for bug finding but slow and solver-fragile, and it worked at the solver meta level rather than in the expression layer. Closed unmerged.
2. **Taylor series (branch `fp-taylor-sin`, single commit `5235320`).** Replaced the CEGIS loop with the closed-form polynomial `sin(x) approx x - x^3/3! + x^5/5! - ...` as a symbolic expression, which is solver-native. A truncated series is only accurate near 0 and diverges approaching `+/-pi/2`, polluting the path condition where decisions matter. Abandoned.
3. **Piecewise-linear ITE (branch `fp-sin-curvature-weighted`, current work on local `sin-pr`).** Models `sin` on `[-pi/2, +pi/2]` as 16 secant segments, each a single affine leaf `m*x + b` (one `fp.mul` plus one `fp.add`, minimal circuit depth for the Z3 FP backend), organized as a nested ITE tree. Design choices, in commit order: uniformly spaced segments gave way to curvature-weighted breakpoints generated by the inverse CDF of the `|sin|` density, so every segment carries an equal share of curvature (`282e8db`); the path condition is scanned for interval constraints on the argument and only overlapping segments are emitted (PC-bound pruning), because at 16 segments the unpruned `fp.eq` tree timed out; negative and positive halves become separate ITE sub-trees under a single sign guard (polarity split, `9a8b430`); the result is clamped to `[-1, 1]` so secant extrapolation cannot produce out-of-codomain values (structural clamp, `cfe130e`). An experiment with a reduced-significand FP sort gated by a `symbolic.mantissa` flag (`d5ca6d5`, `85f2fad`, `e922666`) was reverted. Documented limitations: the encoding is a pointwise approximation, not an enclosure (per-segment error bounded by `width^2 / 8`), arguments outside `[-pi/2, +pi/2]` are not reduced, and NaN/Inf symbolic arguments are not modeled.
4. **Current state (`sin-pr`).** Six revert commits at the head of `sin-pr` slim the branch back to the sin change only (reverting the comparator infra, the test-subpackage move, and the mantissa experiment). The working tree refactor extracts the inline peer encoding into a standalone `numeric/PiecewiseLinearSin.encode()` class and replaces the ad-hoc `CMP` operator used as an ITE guard with a dedicated, semantically correct `LT` operator wired through `Operator`, `BinaryRealExpression`, and `PCParser`. Fourteen end-to-end tests (`TestSin*`) with `.jpf` configs exercise range, sign, branch-pruning, constrained, and nested cases.

---

## 4. Results / Current Status

### 4.1 Merged (4)

- [PR #24](https://github.com/vaibhavbsharma/java-ranger/pull/24) (svcomp): Java-based `Math.toRadians`/`toDegrees` for symbolic tracking.
- [PR #31](https://github.com/vaibhavbsharma/java-ranger/pull/31) (fpSupport): IEEE-754 special value constants.
- [PR #46](https://github.com/vaibhavbsharma/java-ranger/pull/46) (fpSupport): `ISUB` integer overflow simulation.
- [PR #53](https://github.com/vaibhavbsharma/java-ranger/pull/53) (fpSupport): missing Z3 operations.

### 4.2 Open (2)

- [PR #54](https://github.com/vaibhavbsharma/java-ranger/pull/54): spec-faithful 6-arm `FDIV`/`DDIV`, ready for review.
- [PR #37](https://github.com/vaibhavbsharma/java-ranger/pull/37): symbolic FP with the z3str3 string solver.

### 4.3 Draft (1)

- [PR #55](https://github.com/vaibhavbsharma/java-ranger/pull/55): three-arm `FREM`/`DREM` on PR #54's infrastructure; uncompilable until #54 merges.

### 4.4 Closed (12)

- [#23](https://github.com/vaibhavbsharma/java-ranger/pull/23): superseded by #24.
- [#36](https://github.com/vaibhavbsharma/java-ranger/pull/36): `isNaN` predicate absorbed into the unary comparator infrastructure.
- [#38](https://github.com/vaibhavbsharma/java-ranger/pull/38), [#39](https://github.com/vaibhavbsharma/java-ranger/pull/39), [#40](https://github.com/vaibhavbsharma/java-ranger/pull/40): established the FDIV/DDIV NaN/infinity bug and its integration tests.
- [#41](https://github.com/vaibhavbsharma/java-ranger/pull/41), [#42](https://github.com/vaibhavbsharma/java-ranger/pull/42): FP bounds/domain and division-by-zero fixes; #42 is the cleaned version.
- [#43](https://github.com/vaibhavbsharma/java-ranger/pull/43): superseded by the consolidated comparison work (#47 and descendants).
- [#44](https://github.com/vaibhavbsharma/java-ranger/pull/44): first-generation `FREM`, superseded by #55.
- [#45](https://github.com/vaibhavbsharma/java-ranger/pull/45): the 6-arm division encoding, superseded by its reopening as #54.
- [#47](https://github.com/vaibhavbsharma/java-ranger/pull/47): NaN-aware comparisons and comparator infra, closed for splitting; the infra lives on in the fork and in #54.
- [#48](https://github.com/vaibhavbsharma/java-ranger/pull/48): `Math.sin` via CEGIS, superseded by the piecewise-linear encoding.

### 4.5 Fork staging and branch state

- [fork PR #1](https://github.com/SalmaneKhalili/java-ranger/pull/1) and [fork PR #2](https://github.com/SalmaneKhalili/java-ranger/pull/2) merged 2026-07-22 into the fork's `fpSupport`: the comparison/comparator infrastructure (commit `6274df2`) and the first-generation `FREM`. [fork PR #3](https://github.com/SalmaneKhalili/java-ranger/pull/3) (comparison split) and [fork PR #4](https://github.com/SalmaneKhalili/java-ranger/pull/4) (remainder split) are open as the upstream-ready split versions.
- The fork's `fpSupport` branch (tip `a847598`) carries the comparator infra and first-generation `FREM` that are not yet upstream; the upstream `fpSupport` branch (tip `50b5678`) is the base of PRs #54 and #55. Head branches: `fp-fdiv-inf-nan-support` (tip `0f29781`) for #54, `frem-drem-rem` (tip `9d809fe`) for #55.
- `sin-pr` is a local branch; the piecewise `Math.sin` encoding is in the working tree, with no upstream PR yet.

### 4.6 Next steps, in dependency order

1. Merge #54, then promote #55 out of draft (it compiles only against #54's `FPClassExpr` infra).
2. Land #37 (string solver integration).
3. Finish the `sin-pr` refactor and open the `Math.sin` PR, with the documented limitations (pointwise approximation, no argument reduction, no NaN/Inf modeling) stated as known scope.
4. Consider the recorded extensions: argument reduction for `sin` beyond `[-pi/2, pi/2]`, an interval/enclosure variant, FP-aware handling for other transcendental methods, and more SV-COMP seed benchmarks for the FP domain (for example the `radians` family of Issue #22).

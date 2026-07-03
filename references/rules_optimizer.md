# StarRocks FE Optimizer Bug Rules

These rules apply to StarRocks **FE optimizer** Java code — primarily the packages
`com.starrocks.sql.optimizer` (operators, rules, tree rewrites, validation) and
`com.starrocks.qe.feedback` (plan tuning guides). Load these in addition to
`rules_common.md` and `rules_java.md` when scanning `.java` files under those packages.

Rule IDs use the `OPT-NNN` prefix.

---

## OPT-001 — Asymmetric Copy of Paired Operator State
**Severity**: HIGH
**Pattern** (StarRocks-specific): When an optimizer `Operator`/`LogicalOperator`/`PhysicalOperator`
is rebuilt, cloned, or reconstructed (e.g. in a tuning guide, rewrite rule, copy constructor, or
`Builder`), one field of a semantically-coupled pair is carried over but its partner field is
dropped. The classic case is copying `predicate` (via `setPredicate`/`withPredicate`) without also
copying `predicateCommonOperators` (`setPredicateCommonOperators`) — the common sub-expressions that
`ScalarOperatorsReuseRule` extracted and that the predicate references. The rebuilt operator then
references columns produced only by the missing common operators, and the plan fails
`InputDependenciesChecker` with *"The required cols cannot obtain from input cols"*. The failure is
intermittent: the first run has no recorded guide and succeeds, but a later run that applies the
recorded feedback (e.g. `JoinTuningGuide.buildJoinOperator()`) reconstructs the operator and fails.

**Common unsafe patterns to flag**:
1. `newOp.setPredicate(op.getPredicate())` (or a `Builder`/copy that carries `predicate`) **without**
   a matching `newOp.setPredicateCommonOperators(op.getPredicateCommonOperators())`.
2. Any operator reconstruction (`build*Operator()`, `copy`, `duplicate`, rewrite in a
   `TransformationRule`/`TreeRewriteRule`, tuning guide `apply`) that copies a subset of these
   coupled field groups but not all members:
   - `predicate` ↔ `predicateCommonOperators`
   - `projection` (common sub-expressions inside `Projection.commonSubOperatorMap`) when the
     projection is rebuilt from parts
   - `limit`, `predicate`, `projection`, `rowOutputInfo` when only some are propagated
3. A `setPredicate(...)` call whose predicate expression uses `ColumnRefOperator`s that are defined
   in `getPredicateCommonOperators()`, where the surrounding rebuild does not re-attach those common
   operators.

**Natural language**: In StarRocks FE optimizer code (`com.starrocks.sql.optimizer`), whenever an
operator is reconstructed and its `predicate` is copied, verify `predicateCommonOperators` is copied
too — the two form an inseparable pair (the predicate references columns defined only by the common
operators). More generally, flag any operator rebuild/clone/tuning-guide that copies one field of a
coupled state pair while silently omitting its partner, since the resulting plan can fail
`InputDependenciesChecker` ("required cols cannot obtain from input cols").

**Reference fix — StarRocks PR #75773** (https://github.com/StarRocks/starrocks/pull/75773):
`JoinTuningGuide.buildJoinOperator()` rebuilt a `PhysicalHashJoinOperator` but dropped the common
operators. The failure was intermittent — the first run had no recorded guide and succeeded; a
later run that applied the recorded tuning guide reconstructed the join and failed
`InputDependenciesChecker` with *"The required cols cannot obtain from input cols"*.
```java
// BEFORE (buggy) — predicate copied, predicateCommonOperators lost
return new PhysicalHashJoinOperator(
        joinType, joinOperator.getOnPredicate(), joinOperator.getJoinHint(),
        joinOperator.getLimit(),
        joinOperator.getPredicate(),        // <-- predicate carried over
        joinOperator.getProjection(),
        joinOperator.getSkewColumn(), joinOperator.getSkewValues());

// AFTER (fixed) — re-attach the paired field before returning
PhysicalHashJoinOperator newJoinOperator = new PhysicalHashJoinOperator(...);
newJoinOperator.setPredicateCommonOperators(joinOperator.getPredicateCommonOperators());
return newJoinOperator;
```
Note the systemic sibling of this bug: `Operator.Builder.withOperator()` copies
`limit`/`predicate`/`projection` but not `predicateCommonOperators`, so any Builder-based
reconstruction that runs after `ScalarOperatorsReuseRule` can reintroduce the same failure.
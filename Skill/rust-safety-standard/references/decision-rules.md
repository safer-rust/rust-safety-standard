# Decision Rules

This reference expands the rules in `SKILL.md` without repeating its workflow.

## Obligation tracing

For each unsafe operation `U`:

1. List the preconditions `P(U)` imposed by the unsafe operation or callee.
   For example, a raw-pointer read may require validity for reads,
   initialization, and alignment.
2. Identify which facts the current function establishes from checks, safe
   types, or documented invariants.
3. The unestablished facts are the residual caller obligations. They must be
   represented in the enclosing unsafe API's contract, possibly as a conditional
   obligation for a reachable branch.
4. At an unsafe call site, check the same facts in the current state. A local
   `SAFETY` comment is recommended to record that justification.

Do not infer safety from the presence or absence of the `unsafe` keyword alone.
The keyword marks a contract boundary; it does not prove a body is unsafe or
that a body without an unsafe expression preserves a type invariant.

## Compact matrix

| Item | Safe when | Must be unsafe when |
| --- | --- | --- |
| Free function | It discharges all reachable unsafe obligations and does not create an invalid invariant | Caller must provide a residual condition, or it can create an invalid invariant |
| Constructor | It establishes the complete type invariant | Caller must establish any part of the invariant |
| Method | It preserves the invariant and discharges internal unsafe obligations | It may invalidate the invariant or leaves an unsafe obligation to the caller |
| Unsafe trait | Not applicable | An incorrect implementation could make safe use undefined |
| Trait method | The trait's implementation contract guarantees safe use for all callers | Caller must provide a precondition not guaranteed by the trait |

## Recommendations versus requirements

The following are recommendations, not automatic defects:

- document invariants even when a type currently has no unsafe constructor;
- prefer an invariant-establishing constructor over repeating unsafe contracts
  on every accessor;
- avoid introducing new unsafe APIs when a safe design can discharge the same
  obligations;
- justify every unsafe block at its call site.

Missing documentation for an unsafe API, an unverifiable contract, or an
undischarged obligation is a defect even if the current implementation appears
to work.

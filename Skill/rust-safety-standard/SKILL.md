---
name: rust-safety-standard
description: Audit unsafe Rust functions, structs, and traits against the Rust Safety Standard. Use to trace safety requirements, verify type invariants and safety documentation, and find undefined behavior reachable from safe APIs.
metadata:
  short-description: Review Rust unsafe safety contracts
---

# Rust Safety Standard

Use this skill to audit Rust code involving unsafe functions, unsafe traits,
unsafe blocks, raw-pointer dereferences, mutable statics, union field access,
or safe APIs that depend on unsafe code. Determine whether a public safe API can
reach undefined behavior and whether every unsafe boundary has a sufficient,
caller-usable contract.

## Scope

Default to an audit: inspect code and report findings without modifying files.
If the user requests a patch or documentation change, make only changes that
address an identified finding.

Do not add `unsafe` merely to silence a compiler. Do not weaken a safety
contract to make a call compile. The absence of an unsafe block alone does not
show that a function preserves a type invariant.

## Audit workflow

1. Identify the unsafe functions, unsafe traits, unsafe blocks, and unsafe
   operations in the requested scope.
2. For every unsafe operation or unsafe callee, list the conditions required for
   safe use. For example, a raw-pointer read may require valid readable memory,
   initialized bytes, and suitable alignment.
3. Determine which conditions the enclosing function establishes. Every
   condition it cannot establish is a residual obligation that must be
   propagated to its caller.
4. Reconstruct any type invariant on which an associated function relies.
   Check that constructors establish it and methods preserve it.
5. Classify each function, constructor, method, trait, and trait method using
   the rules below. A safe API must discharge all applicable obligations;
   an unsafe API must document the obligations it leaves to its user.
6. Check documentation and call-site evidence. Safety requirements must be
   sufficient, externally verifiable, and independent of internal details.
7. Report any public safe path that can violate an unsafe requirement or type
   invariant, with file/line evidence and a focused correction. Do not edit
   unless the user requests a change.

## Decision rules

### Functions

- A function containing unsafe operations may be safe only when it establishes
  every condition required by those operations and preserves all relevant type
  invariants.
- If any condition cannot be established, the function must be `unsafe` and its
  public contract must state the residual condition in terms visible to the
  caller.
- A function without a direct unsafe operation has no automatic unsafe
  requirement. When it constructs or can invalidate a type with an invariant,
  apply the struct rules instead of relying on this fact alone.
- A safe wrapper may call an unsafe function only after discharging its complete
  contract. Do not propagate a callee's wording mechanically when the wrapper
  proves the condition in a different, externally verifiable form.
- Conditional calls produce conditional contracts: express the caller's
  obligation as the condition under which the unsafe path is reachable.

### Structs and type invariants

- Document the type invariant when unsafe constructors establish it or when
  associated functions rely on it.
- A safe constructor must establish the complete invariant. A constructor that
  requires the caller to establish it must be `unsafe`.
- A safe method must preserve the invariant. A method that may break it must be
  `unsafe`.
- An associated function without a receiver follows the function rules, with
  the additional check for indirect construction or mutation of the type.

### Traits

- A trait is `unsafe` when an incorrect implementation could make undefined
  behavior reachable through safe trait use.
- An unsafe trait must state the guarantees that every implementation must
  uphold, and associate each guarantee with the relevant trait item when
  applicable.
- A trait method is `unsafe` when its caller must provide conditions that the
  trait's implementation obligations cannot guarantee. Document those caller
  conditions in the trait definition.
- Keep implementation obligations and method-call obligations separate. An
  implementation must not silently impose stricter safety requirements than
  the trait contract.

## Safety documentation

For each unsafe function, document a `# Safety` section containing sufficient
preconditions stated using parameters, receiver state, documented invariants,
or other facts the caller can verify. Do not describe only internal temporaries
or implementation steps. It is acceptable for a contract to be sufficient
without being minimal, but distinguish mandatory preconditions from discouraged
usage.

For each unsafe trait, document the guarantees every implementation must uphold
in the trait definition. For each unsafe trait method, document caller safety
requirements there; implementations should refer to the trait contract rather
than redefining it.

At unsafe call sites, it is recommended to add a concise `SAFETY` comment
explaining why the conditions hold. A comment is evidence, not a replacement
for proving the conditions from code or an established invariant.

## Reporting

For review or audit, report:

- a concise soundness verdict and scope;
- findings ordered by severity, each with path/line, reachable API path,
  violated rule, concrete safety condition, and recommended fix;
- missing or unverifiable safety documentation;
- type invariants and safety requirements relevant to the finding;
- documentation omissions and recommended documentation improvements.

For requested fixes, include the condition that the patch establishes or
propagates. Preserve unrelated behavior and follow the crate's existing style.

Read the supporting references only when the task needs detailed examples or a
rule lookup:

- [Decision rules](references/decision-rules.md) for the compact rule matrix
  and obligation-tracing method.
- [Documentation guidance](references/safety-documentation.md) for contract
  and call-site templates.
- [Examples](references/examples.md) for small, compile-oriented patterns.

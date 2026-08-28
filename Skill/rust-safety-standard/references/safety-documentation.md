# Safety Documentation

Safety requirements must be sufficient to prevent undefined behavior when the
caller satisfies them. They should be phrased in terms visible outside the
function's implementation.

## Unsafe function template

```rust
/// # Safety
/// - `p` must be valid for reads of one `u32`.
/// - The pointed-to bytes must be initialized.
/// - `p` must be properly aligned for `u32`.
pub unsafe fn read_u32(p: *const u32) -> u32 {
    // SAFETY: the caller guarantees the documented requirements for `p`.
    unsafe { *p }
}
```

State the conditions relevant to the operation. Use public parameters,
receiver state, or documented invariants rather than internal aliases such as
`tmp` or `raw`.

## Unsafe trait template

```rust
/// # Safety
/// Implementations must ensure that `as_bytes` returns a slice valid for reads
/// for the lifetime of the returned reference.
pub unsafe trait Buffer {
    fn as_bytes(&self) -> &[u8];

    /// # Safety
    /// `index` must be less than `self.as_bytes().len()`.
    unsafe fn get_unchecked(&self, index: usize) -> u8;
}
```

The implementation contract and the method caller contract are separate. An
implementation may explain why it satisfies the trait contract, but it must
not add stricter preconditions to the method.

## Call-site comments

The standard recommends a short, local justification:

```rust
// SAFETY: `p` points to the live `value`, is aligned for `u32`, and the read
// covers exactly one initialized value.
unsafe { read_u32(p) }
```

Do not use a bare `// SAFETY: this is safe` comment. If a condition is supplied
by an invariant, name that invariant and the constructor or operation that
establishes it.

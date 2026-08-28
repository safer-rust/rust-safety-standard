# Examples

These examples illustrate the standard's rules; adapt them to the target
crate rather than copying them blindly.

## Propagating a residual obligation

```rust
/// # Safety
/// `p` must be aligned for `u32` and point to four initialized readable bytes.
pub unsafe fn read_u32(p: *const u32) -> u32 {
    // SAFETY: the caller supplies the documented pointer conditions.
    unsafe { *p }
}

pub fn read_local() -> u32 {
    let value = 42_u32;
    let p = &value as *const u32;
    // SAFETY: `p` points to the live, initialized, suitably aligned `value`.
    unsafe { read_u32(p) }
}
```

If a wrapper cannot establish alignment, initialization, or lifetime, it must
remain unsafe and document that residual requirement.

## Invariant-establishing constructor

```rust
/// # Safety
/// The field is always even for every valid `EvenNumber`.
pub struct EvenNumber(u32);

impl EvenNumber {
    /// # Safety
    /// `value` must be even.
    pub unsafe fn new_unchecked(value: u32) -> Self {
        Self(value)
    }

    pub fn get(&self) -> u32 {
        self.0
    }
}
```

If the type has no such invariant, `new_unchecked` can instead be a safe
function; the name alone does not require `unsafe`.

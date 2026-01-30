# Rust Safety Standard
## 1. Background
Rust’s safety guarantees rely heavily on the correct use of unsafe code. 
However, maintaining safety becomes increasingly challenging as real-world crates grow in size, particularly when attempting to enforce a clear separation between safe and unsafe components. 
Rust currently provides only the following high-level [soundness requirement](https://rust-lang.github.io/unsafe-code-guidelines/glossary.html#soundness-of-code--of-a-library):
> We say that a library (or an individual function) is sound if it is impossible for safe code to cause Undefined Behavior using its public API.

This definition does not provide actionable criteria for determining which functions or struct methods should be declared safe or unsafe. 
The design space becomes even more complex when visibility boundaries are taken into account.

Furthermore, when a function is declared unsafe, developers must clearly document the conditions required for its correct use, and users must fully understand them.
This raises another important question: what information is necessary to specify the safety requirements of an unsafe function?

This document addresses these challenges by presenting actionable guidance derived from extensive reviews of unsafe code in several Rust projects, particularly the Rust standard library.
Our recommendations are grounded in the theoretical foundation presented in [A Trace-based Approach for Code Safety Analysis](https://arxiv.org/pdf/2510.10410). 

## 2 Free Functions
A free function is a function defined at the module level that can be called directly by its path rather than through an instance or type.

### 2.1 Safety Rules
The soundness of free functions is relatively straightforward to justify.

- **Function Safety Rule 1**: If a function contains no unsafe code, it cannot cause undefined behavior and is therefore safe.
- **Function Safety Rule 2**: If a function contains unsafe code, it may be declared safe only if all conditions required for the safe use of that unsafe code are met.

## 2.2 Safety Comments
There are two mandatory rules and one recommended practice for documenting the safety requirements of free functions:
- **Function Comments Rule 1**: All unsafe functions must document their safety requirements.
- **Function Comments Rule 2**: Safety requirements must be externally verifiable and must not depend on the function’s internal implementation.
- **Function Comments Rule 3** (Recommended):  At the callsite, users are encouraged to justify why the safety requirements are satisfied.

### 2.3 Example Cases

The following function foo performs a raw pointer dereference, which is an unsafe operation. 
The raw pointer `r` is derived from the function parameter `p`, and the function does not check whether it is valid before dereferencing it. 
According to Function Safety Rule 2, foo must be declared unsafe.

```rust
/// # Safety:
/// - `p` must be valid for reads.
/// - `p` must be properly aligned for `u32`.
pub unsafe fn foo(p: *const u32) -> u32 {
    let r = p;
    unsafe { *r }
}

```

Additionally, according to Function Comments Rule 1, developers must provide the safety requirements of the function.
The safety requirements reference only the function parameter `p`, which is visible to the caller, rather than the internal pointer `r`. 
Therefore, they satisfy Function Comments Rule 2.

When calling the unsafe function `foo`, the caller must check if all safety requirements are satisfied. 
In the following example, the caller `bar` justifies why each of the safety requirements of `foo` is satisfied at the callsite in accordance with Function Comments Rule 3. 
Since all the requirements are satisfied, `bar` can be declared as safe according to Function Safety Rule 1. 

```rust
pub  fn bar() {
    let mut x: u32 = 42;
    let p: *const u32 = &x;
    /// Safety: 
    /// - `p` is valid for reads because it points to `x`.
    /// - `p` is properly aligned for `u32`.
    unsafe { foo(p); }
}
```

If one of the safety requirements cannot be satisfied by the caller, the caller cannot be declared safe.
Moreover, the caller must propagate any unsatisfied safety requirements.
Consider the example below: `bar` satisfies one safety requirement (valid for reads), but cannot ensure the second requirement (alignment).
Therefore, `bar` must be declared unsafe due to the unsatisfied alignment requirement.

```rust
/// # Safety:
/// - `x` must be properly aligned for `u32`.
pub unsafe fn bar<T>(x: T) {
    let p: *const u32 = &x as *const T as *const u32;

    /// Safety: 
    /// - `p` is valid for reads because it points to `x`.
    unsafe { foo(p as *mut u32); }
}
```

In some cases, the safety requirements of unsafe callees are not directly propagated to the caller. It can be transformed to other safety requirements.
```rust
/// # Safety: 
/// - `x` must be properly aligned for `u32`, or
/// - `x` <= 0
pub unsafe fn bar<T>(x: T) {
    let p: *const u32 = &x as *const T as *const u32;

    if x > 0 {
        /// Safety: 
        /// - `p` is valid for reads because it points to `x`.
        unsafe { foo(p as *mut u32); }
    }
}
```

## 3 Stucts

Consider the following struct:
```rust
struct Foo<'a> {
    ptr: *mut u8,
    len: usize 
}
```

One possible design is to declare the constructor as safe while marking other methods as unsafe:
```rust
impl Foo {
    pub fn from(p: *mut u8, l: usize) -> Foo {
        Foo { ptr: p, len: l }
    }
    pub unsafe fn get(&self) -> &[u8] { 
        slice::from_raw_parts(self.ptr, self.len)
    }
    pub unsafe fn set_len(&mut self, l: usize) {
        self.len = l;
    }
}
```

Alternatively, the constructor can be declared unsafe while the accessor method remains safe:
```rust
impl Foo {
    pub unsafe fn from(p: *mut u8, l: usize) -> Foo {
        Foo { ptr: p, len: l }
    }
    pub fn get(&self) -> &[u8] { 
        unsafe { slice::from_raw_parts(self.ptr, self.len) }
    }
    pub unsafe fn set_len(&mut self, l: usize) {
        self.len = l;
    }
}
```
Both implementations satisfy Rust’s soundness requirement. 


## 4 Traits

## 5 Visibility

## 6 More

### 6.1 Static Mut

## 7 Summary


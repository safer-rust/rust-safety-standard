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
- **Function Comments Rule 3** (Recommended):  At the callsite of unsafe code, users are encouraged to justify why the safety requirements are satisfied.

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
    // Safety: 
    // - `p` is valid for reads because it points to `x`.
    // - `p` is properly aligned for `u32`.
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

    // Safety: 
    // - `p` is valid for reads because it points to `x`.
    unsafe { foo(p as *mut u32); }
}
```

In some cases, the safety requirements of unsafe callees are not directly propagated to the caller; instead, they may manifest as different safety obligations.
```rust
/// # Safety: 
/// - `x` must be properly aligned for `u32`, or
/// - `x` <= 0
pub unsafe fn bar<T>(x: T) {
    let p: *const u32 = &x as *const T as *const u32;

    if x > 0 {
        // Safety: 
        // - `p` is valid for reads because it points to `x`.
        unsafe { foo(p as *mut u32); }
    }
}
```

## 3 Stucts

A struct is a user-defined data type composed of named fields. Its behavior is defined through associated functions within `impl` blocks.
- A method is a special associated function that takes self as the first parameter.
- Associated functions without a receiver are commonly used for constructors or type-level operations.
- Rust does not provide built-in constructors; by convention, static associated functions such as `new` are used.
- Struct instances can also be created using struct literals, e.g., `Foo { field1: value, ... }`.

### 3.1 Safety Rules
- **Struct Safety Rule 1**: If none associated functions of a struct contain unsafe code, the struct cannot cause undefined behavior and all its associated functions can be declared as safe.
- **Struct Safety Rule 2**: For associated functions without a receiver, their safety rules are generally the same with the safety rules defined for [free functions](#2-free-functions).

Before discussing methods involving unsafe code, we first define the type invariant of a struct.
A type invariant specifies the conditions that all instances of the type must satisfy, regardless of which constructor is used to create them.
If a constructor is unsafe, its usage must satisfy the corresponding safety requirements, which forms the basis of Struct Safety Rule 3.
In practice, the domain of the invariant can be defined with respect to the potential undefined behaviors that might be triggered by other methods.

- **Struct Safety Rule 3**: A constructor can be declared safe if it guarantees that the type invariant of the struct is satisfied; otherwise, it must be declared unsafe.

The safety of a struct’s methods depends on whether they contain unsafe code:
- **Struct Safety Rule 4**: A method that contains no unsafe code can be declared safe if it does not violate the type invariant of the struct.
- **Struct Safety Rule 5**: If a method contains unsafe code, it can be declared safe only if both of the following conditions are met:
  - 1) It does not violate the type invariant of the struct.
  - 2) All safety requirements of its internal unsafe code are satisfied.
   
Note that type invariants play a key role in preventing the safety of methods from depending on the behavior of constructors, and vice versa.
 
### 3.2 Safety Comments
- **Struct Comments Rule 1** (Recommended): Each struct with either unsafe constructors or unsafe methods should document the type invariant.
- **Struct Comments Rule 2**: Each unsafe associated function should document its safety requirements:
  - 1) The requirements should not depend on other functions of the struct.
  - 2) They must be externally verifiable and must not depend on the function’s internal implementation.
- **Struct Comments Rule 3** (Recommended):  At the callsite of unsafe code, users are encouraged to justify why the safety requirements are satisfied.

### 3.3 Example Cases

Consider the following struct, there are different ways to declare the safety of its associated functions.
```rust
struct Foo<'a> {
    ptr: *mut u8,
    len: usize 
}
```

One possible design is to declare the constructor as safe while marking other methods as unsafe. 
In this case, this type does not maintain a validity invariant.
Instead, each unsafe method defines the conditions required for safe use.
```rust
impl Foo {
    pub fn from(p: *mut u8, l: usize) -> Foo {
        Foo { ptr: p, len: l }
    }

    /// # Safety:
    /// - `self.ptr` must be valid for reads of `self.len` bytes.
    /// - The memory must be properly aligned.
    /// - The memory must not be mutated for the duration of the returned slice.
    pub unsafe fn get(&self) -> &[u8] {
        // Safety:
        // - The caller guarantees that `ptr` and `len` satisfy the requirements of `slice::from_raw_parts`.
        slice::from_raw_parts(self.ptr, self.len)
    }

    /// # Safety:
    /// - This method may break the type invariant of the struct
    /// - `l` must not exceed the size of the allocation referenced by `self.ptr`.
    pub unsafe fn set_len(&mut self, l: usize) {
        self.len = l;
    }
}
```

Alternatively, the constructor can be declared unsafe while the accessor method remains safe. In this design, the constructor establishes a type invariant that other methods can rely on.

```rust
/// # Safety:
/// ## Type invariant
/// - `ptr` is valid for reads of `len` bytes.
/// - `ptr` is properly aligned.
/// - The referenced memory remains valid for the lifetime of `Foo`.
struct Foo<'a> {
    ptr: *mut u8,
    len: usize 
}

impl Foo {
    /// # Safety
    /// - `p` must be valid for reads of `l` bytes.
    /// - `p` must be properly aligned.
    /// - The memory must remain valid for the lifetime of the constructed `Foo`.
    pub unsafe fn from(p: *mut u8, l: usize) -> Foo {
        Foo { ptr: p, len: l }
    }

    pub fn get(&self) -> &[u8] {
        // Safety:
        // - The invariant is guaranteed by `from`.
        unsafe { slice::from_raw_parts(self.ptr, self.len) }
    }

    /// # Safety:
    /// - This method may break the type invariant of the struct
    /// - `l` must not exceed the size of the allocation referenced by `self.ptr`.
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


# Rust Safety Standard (Request for Comments)
This standard is a draft and welcomes feedback.

## 1. Background
Rust’s safety guarantees rely heavily on the correct use of unsafe code. 
However, maintaining safety becomes increasingly challenging as real-world crates grow in size, particularly when attempting to enforce a clear separation between safe and unsafe components. 
Rust currently provides only the following high-level [soundness requirement](https://rust-lang.github.io/unsafe-code-guidelines/glossary.html#soundness-of-code--of-a-library):
> We say that a library (or an individual function) is sound if it is impossible for safe code to cause Undefined Behavior using its public API.

This definition does not provide actionable guidance for determining which functions or struct methods should be declared safe or unsafe. 
The design space becomes even more complex when visibility boundaries are taken into account.

Furthermore, when a function is declared unsafe, developers must clearly document the conditions required for its correct use, and users must fully understand them.
This raises another important question: what information is necessary to specify the safety requirements of an unsafe function?

This document addresses these challenges by presenting actionable guidance derived from extensive reviews of unsafe code in several Rust projects, particularly the Rust standard library and Rust-for-Linux.
Our recommendations are grounded in the theoretical foundation presented in [A Trace-based Approach for Code Safety Analysis](https://arxiv.org/pdf/2510.10410). 

## 2 Rationale
### 2.1 Definition of Unsafe
In general, unsafe indicates that there are additional requirements imposed on the user. These requirements should be clearly documented to avoid misuse.
- **Unsafe function**: a function is unsafe if it requires the caller to satisfy certain requirements.
- **Unsafe trait**: a trait is unsafe if it requires the implementer to satisfy certain requirements.

These two concepts are orthogonal. 

### 2.2 Safety Documentation
- **Define unsafe code**: Safety requirements should be provided alongside each unsafe definition as comments or documents. They should state sufficient conditions to prevent undefined behavior when using the unsafe code, and may also describe discouraged usage patterns that are known to lead to undefined behavior.
- **Use unsafe code**: When using unsafe code, developers must provide comments justifying why the relevant safety requirements are upheld.

### 2.3 Design Choices
In general, there are two scenarios in which a developer declares a function unsafe.
- **Dependent unsafe (mandatory)**:
  -  A free function invokes unsafe code from its dependencies, and the safety requirements of that unsafe code cannot be fully discharged by the function itself. As a result, the function must be declared unsafe to propagate the remaining safety requirements to its caller.
  -  An associated function may violate a struct’s type invariant, which can in turn affect other methods of the struct that rely on unsafe code. Regardless of whether it contains unsafe code, such a function should be declared unsafe, with safety requirements specifying how the invariant must be upheld.
  
- **New unsafe (by design)**:  
  - Besides dependent unsafe, developers may deliberately introduces additional safety contracts that must be upheld by the caller in order to use the function safely.
  - When there is confusion about whether a function should be declared safe or unsafe, remember that safety invariants always go before `unsafe`.
Using `unsafe` means that a function may violate some safety invariants if it is misused.

In the following example, whether `new_unchecked` should be safe or unsafe depends on whether the type explicitly declares a safety invariant that `x` must be even.
If such an invariant is part of the type’s contract, then `new_unchecked` must be declared `unsafe`, since incorrect usage can violate the invariant.
If such invariant does not exist, then `new_unchecked` is safe, as misuse does not violate any safety invariant.

Therefore, both versions below are sound, but they correspond to different design choices regarding the type’s invariants.

```rust
/// # Safety
/// ## Struct invariant
/// - `x` must be even.
pub struct EvenNumber {
  val: u32,
}

impl EvenNumber {
    /// # Safety
    /// - `x` must be even.
    pub unsafe fn new_unchecked(x: u32) -> Self {
        EvenNumber{x}
    }
}
```

```rust
pub struct EvenNumber {
  val: u32,
}

impl EvenNumber {
    pub fn new_unchecked(x: u32) -> Self {
        EvenNumber{x}
    }
}
```

For example, in the Rust standard library, [`RawWaker`](https://doc.rust-lang.org/core/task/struct.RawWaker.html) and [`RawWakerVTable`](https://doc.rust-lang.org/core/task/struct.RawWakerVTable.html) are types without type invariants and currently contain no internal unsafe code. 
Therefore, their constructors `new` are both declared safe. These structs are then used by [`LocalWaker`](https://doc.rust-lang.org/core/task/struct.LocalWaker.html), which enforces the type invariants. As a result, its constructors [`new`](https://doc.rust-lang.org/core/task/struct.LocalWaker.html#method.new) and [`from_raw`](https://doc.rust-lang.org/core/task/struct.LocalWaker.html#method.from_raw) are declared unsafe.

Note that the introduction of new unsafe code is discouraged unless necessary.
Sometimes, introducing new unsafe functions can bring benefits and help avoid exposing additional unsafe functions.
This is essential to prevent the proliferation of unsafe code and the degradation of overall safety in Rust projects.

### 2.4 Decoupling
Sometimes, it can be unclear how safety duties should be allocated among related program components. Decoupling these responsibilities is therefore essential. 
A common source of confusion involves free functions that return a struct instance. In such cases, we recommend treating the free function as indirectly constructing the instance via the struct’s own constructors, including the literal constructor.
For example, in the following code, we can treat `new_unchecked` as a free function that returns a an `EvenNumber` instance, instead of a constructor; the real constructor is the literal constructor of the tuple struct `EvenNumber()`.

```rust
pub struct EvenNumber(u32);

/// # Safety
/// - `x` must be even.
pub unsafe fn new_unchecked(x: u32) -> EvenNumber {
    EvenNumber(x)
}
```

A better design is to place the unsafe function inside the `impl` block of the struct:
```rust
/// # Safety
/// ## Struct invariant
/// - `x` must be even.
pub struct EvenNumber(u32);

impl EvenNumber {
    /// # Safety
    /// - `x` must be even.
    pub unsafe fn new_unchecked(x: u32) -> Self {
        EvenNumber(x)
    }
}
```

Similarly, one may create objects of other types that enclose the struct, such as `Box<EvenNumber>`. These should also rely on the constructor of `EvenNumber`.
```rust
pub unsafe fn new_unchecked(x: u32) -> Box<EvenNumber> {
    Box::new(EvenNumber::new_unchecked(x))
}
```

### 2.5 Visibility and Soundness
In Rust, soundness is tied to [Visibility](https://doc.rust-lang.org/reference/visibility-and-privacy.html), because it determines which and how APIs are accessible.
By default, a module is the smallest visibility boundary: all functions, structs, and struct fields are private to the module unless explicitly made public.
Developer can rely on the visibility restrictions to maintain sound abstractions, i.e., all uses of a module’s public safe items (or unsafe items, provided their safety requirements are met) from outside the module must not lead to undefined behavior.
This is the default soundness criterion adopted by the Rust standard library. 

#### 2.5.1 Struct Literals and Field Projections
In the following example, `Vec` is a struct with defined invariants, and `new` is a safe constructor that ensures all safety invariants hold. Although it is possible to break these invariants within the module using struct literals or field projections (e.g., `v.len = usize::MAX`), this is still considered sound. Preventing these fields from being publicly accessible is key to ensuring soundness.

```rust
/// # Safety Invariants
/// - If `cap == 0`, `ptr` must be dangling and never dereferenced.
/// - If `cap > 0`, `ptr` points to a heap allocation for `cap` elements of `T`.
/// - The range `[0, len)` is initialized.
/// - `len <= cap`.
/// - This `Vec` uniquely owns the allocation.
mod vec {
    pub struct Vec<T> {
        ptr: NonNull<T>, // pointer to heap allocation
        len: usize,      // number of initialized elements
        cap: usize,      // total allocated capacity
        _marker: PhantomData<T>,
    }

    pub fn new() -> Self {
        Self {
            ptr: NonNull::dangling(),
            len: 0,
            cap: 0,
            _marker: PhantomData,
        }
    }
    ...
}
```

#### 2.5.2 Static Variables
A function may rely on the states of private static variables to ensure soundness. 
For example, the function `do_critical_task` in the following code can be declared safe. 
It requires the static variable `PTR` to be either `null` or to point to `CONFIG`. 
This requirement is always satisfied because, within this module, `PTR` can only have these two states. 
Although future modifications to the module could introduce additional states or allow `PTR` to be modified in new ways, it is unnecessary to declare `do_critical_task` as unsafe.
In that case, declaring the new function as unsafe should be preferred in order to avoid introducing a semantically breaking change to the function `do_critical_task`.

```rust
mod FooSys {
    use std::{ptr, sync::atomic::{AtomicPtr, Ordering}};

    /// Static atomic pointer pointing to system configuration
    static PTR: AtomicPtr<u32> = AtomicPtr::new(ptr::null_mut());

    /// Initialize the system and set PTR to point to a valid static resource
    pub fn initialize_system() {
        static CONFIG: u32 = 42;
        PTR.store(&CONFIG as *const u32 as *mut u32, Ordering::SeqCst);
    }

    pub fn do_critical_task() {
        let ptr = PTR.load(Ordering::SeqCst);
        if !ptr.is_null() {
            unsafe { read_config(ptr); }
        }
    }

    // Safety: `ptr` must point to a valid configuration.
    unsafe fn read_config(ptr: *mut u32) {
       ...
    }
}
```
Sometimes, in a large crate, a static variable may be `pub` across several modules but not exposed outside the crate. 
As long as it cannot enter invalid states and downstream crate users cannot introduce invalid states to it, using the static variable in this way is considered sound.

#### 2.5.3 Sealed Traits
Making a trait sealed is necessary if implementing it outside the module could introduce undefined behavior. 
In the example below, the module FooSys defines a private trait `Sealed` and a public trait `Foo` that requires implementers to also implement Sealed. 
This design ensures that only types defined within the module, such as `Bar`, can implement `Foo`.
This prevents external types that have a `len` field from misimplementing `Foo`.
```rust
mod FooSys {
    // Private module to seal the trait
    pub mod private {
        pub trait Sealed {}
    }

    /// Public trait `Foo`, only implementable within this module
    /// because it requires `Sealed`.
    pub trait Foo: private::Sealed {
        fn set_len(&mut self, len: usize);
        fn get_len(&self) -> usize;
    }

    /// Internal struct with private field
    pub struct Bar {
        len: usize, 
    }

    impl private::Sealed for Bar {}

    impl Foo for Bar {
        ...
    }
}
```

## 3 Rules for Free Functions
A free function is a function defined at the module level that can be called directly by its path rather than through an instance or type.

### 3.1 Safety Rules
The soundness of free functions is relatively straightforward to justify.

- **Function Safety Rule 1**: If a free function contains unsafe code, it can be declared safe only if all conditions required for the safe use of that unsafe code are met.
- **Function Safety Rule 2**: If a free function contains no unsafe code, there is no mandatory contract to be upheld, and it can be declared safe.

Note that Function Safety Rule 2 continues to hold even when the function calls other functions that contain unsafe code, provided that those callees themselves satisfy Function Safety Rule 1 and do not impose additional safety contracts.

## 3.2 Safety Comments
There are two mandatory rules and one recommended practice for documenting the safety requirements of free functions:
- **Function Comments Rule 1**: All unsafe functions must document their safety requirements.
- **Function Comments Rule 2**: Safety requirements must be externally verifiable and must not depend on the function’s internal implementation.
- **Function Comments Rule 3** (Recommended):  At the callsite of unsafe code, users are encouraged to justify why the safety requirements are satisfied.

### 3.3 Example Cases

The following function `foo` performs a raw pointer dereference, which is an unsafe operation. 
The raw pointer `r` is derived from the function parameter `p`, and the function does not check whether it is valid before dereferencing it. 
According to Function Safety Rule 1, foo must be declared unsafe.

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
Since all the requirements are satisfied, `bar` can be declared safe according to Function Safety Rule 1. 

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
/// - `T` must have size at least 4.
/// - `T` must not have padding in its first 4 bytes.
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
/// - `x` <= 0 or 
/// - the following three properties should be upheld:
///   * `x` must be properly aligned for `u32`.
///   * `T` must have size at least 4.
///   * `T` must not have padding in its first 4 bytes.
pub unsafe fn bar<T>(x: T) {
    let p: *const u32 = &x as *const T as *const u32;

    if x > 0 {
        // Safety: 
        // - `p` is valid for reads because it points to `x`.
        unsafe { foo(p as *mut u32); }
    }
}
```

Besides function calls, operations such as raw pointer dereferencing, accessing static mut values, and reading or writing union fields can be treated in the same way as unsafe callees with specific safety requirements, and the same rules apply.

## 4 Rules for Stucts

A struct is a user-defined data type composed of fields. Its behavior is defined through associated functions within `impl` blocks.
- A method is a special associated function that takes self as the first parameter.
- Associated functions without a receiver are commonly used for constructors or type-level operations.
- Struct instances can also be created using struct literals, e.g., `Foo { field1: value, ... }`.

A struct can have a type invariant that specifies the conditions that all instances of the type must satisfy, regardless of which constructor is used to create them.
Type invariants are crucial to the safety of associated functions and should be clearly documented. 
Similar to unsafe, there are two types of invariants:
- **Dependent invariants (mandatory)**: Associated functions contain unsafe code and rely on these invariants to avoid undefined behavior.
- **New invariants (by design)**: The invariants are unrelated to unsafe code of the struct and are introduced intentionally by design.

Type invariants are widely used in Rust-for-Linux, e.g., [IovIterSource](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/iov.rs#L38-L49), [List](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/list.rs#L31-L266), [ListLinks](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/list.rs#L367-L375), [Cursor](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/list.rs#L945-L952), [CursorPeek](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/list.rs#L1100-L1107).
They play a key role in preventing the safety of methods from depending on the behavior of constructors, and vice versa.

### 4.1 Safety Rules

- **Struct Safety Rule 1**: A constructor can be declared safe if it guarantees that the type invariant of the struct is satisfied; otherwise, it must be declared unsafe.
- **Struct Safety Rule 2**: For associated functions without a receiver, their safety rules are generally the same with the safety rules defined for [free functions](#3-free-functions).
- **Struct Safety Rule 3**: A method that contains no unsafe code can be declared safe if it does not violate the type invariant of the struct.
- **Struct Safety Rule 4**: If a method contains unsafe code, it can be declared safe only if both of the following conditions are met:
  - 1) It does not violate the type invariant of the struct.
  - 2) All safety requirements of its internal unsafe code are satisfied.
 
### 4.2 Safety Comments
- **Struct Comments Rule 1** (Recommended): Each struct with unsafe constructors should document the type invariant.
- **Struct Comments Rule 2**: Each unsafe associated function should document its safety requirements:
  - 1) The requirements should not depend on other functions of the struct.
  - 2) They must be externally verifiable and must not depend on the function’s internal implementation.
- **Struct Comments Rule 3** (Recommended):  At the callsite of unsafe code, users are encouraged to justify why the safety requirements are satisfied.

### 4.3 Example Cases

Consider the following struct, there are different ways to declare the safety of its associated functions.
```rust
struct Foo<'a> {
    ptr: *mut u8,
    len: usize 
}
```

One possible design is to declare the constructor safe while marking other methods unsafe. 
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

Both implementations satisfy Rust’s soundness requirement. For the first type, examples can be found in the Rust standard library, such as [DwarfReader](https://github.com/rust-lang/rust/blob/7d8ebe3128fc87f3da1ad64240e63ccf07b8f0bd/library/std/src/sys/personality/dwarf/mod.rs#L15-L69) and [Unique](https://github.com/rust-lang/rust/blob/7d8ebe3128fc87f3da1ad64240e63ccf07b8f0bd/library/core/src/ptr/unique.rs#L94-L156); the second type is commonly used in Rust-for-Linux, with examples including [IovIterSource](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/iov.rs#L38-L49), [Resource](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/io/resource.rs#L75-L169), and [Bitmap](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/bitmap.rs#L17-L90).

## 5 Rules for Traits
A trait defines a collection of associated items (typically functions) that can be shared across multiple types. Traits may provide default implementations for some items; these implementations are automatically available to implementing types unless explicitly overridden.
- When a type implements a trait, the trait’s functions behave like associated functions of the type.
- Similar to those introduced in structs, these associated functions may or may not have a receiver.

### 5.1 Safety Rules
- **Trait Safety Rule 1**: Declare a trait unsafe when the correctness of its implementations is required to prevent undefined behavior in safe code.
- **Trait Safety Rule 2**: A trait method should be unsafe if its correct use depends on safety guarantees that must be enforced by the caller.

### 5.2 Safety Comments
- **Trait Comments Rule 1**: An unsafe trait must document the safety invariants that all implementations are required to uphold.
- **Trait Comments Rule 2**: Each unsafe method of a trait must clearly document the safety requirements that callers must satisfy.
    - The safety requirements of an unsafe method are distinct from the trait invariants.
    - The safety requirements of an unsafe method must be specified in the trait definition, where they form part of the trait’s contract.
    - Implementations should not introduce stricter safety requirements than those declared by the trait.

- **Trait Comments Rule 3**  (Recommended): Implementations of unsafe traits are encouraged to justify why the required invariants are satisfied.


### 5.3 Example Cases

The following example introduces a trait `Buffer` that should be declared unsafe, because incorrectly implementing it may introduce undefined behavior. 
The trait defines an unsafe method `get_unchecked`. It cannot be declared safe because its safety requirements cannot be guaranteed solely by the trait invariant.
```rust
/// # Safety:
/// ## Trait invariant: 
/// - Implementors must guarantee that `as_bytes()` always returns a slice pointing to valid memory for reads
/// - and that the slice remains valid for the lifetime of the returned reference. 
unsafe trait Buffer {
    fn as_bytes(&self) -> &[u8];

    /// # Safety:
    /// - The caller must ensure that `index < self.as_bytes().len()`.
    unsafe fn get_unchecked(&self, index: usize) -> u8;
}
```

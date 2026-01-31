# Rust Safety Standard
This standard is a draft and welcomes feedback.

## 1. Background
Rust’s safety guarantees rely heavily on the correct use of unsafe code. 
However, maintaining safety becomes increasingly challenging as real-world crates grow in size, particularly when attempting to enforce a clear separation between safe and unsafe components. 
Rust currently provides only the following high-level [soundness requirement](https://rust-lang.github.io/unsafe-code-guidelines/glossary.html#soundness-of-code--of-a-library):
> We say that a library (or an individual function) is sound if it is impossible for safe code to cause Undefined Behavior using its public API.

This definition does not provide actionable criteria for determining which functions or struct methods should be declared safe or unsafe. 
The design space becomes even more complex when visibility boundaries are taken into account.

Furthermore, when a function is declared unsafe, developers must clearly document the conditions required for its correct use, and users must fully understand them.
This raises another important question: what information is necessary to specify the safety requirements of an unsafe function?

This document addresses these challenges by presenting actionable guidance derived from extensive reviews of unsafe code in several Rust projects, particularly the Rust standard library and Rust-for-Linux.
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

The following function `foo` performs a raw pointer dereference, which is an unsafe operation. 
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
Since all the requirements are satisfied, `bar` can be declared as safe according to Function Safety Rule 2. 

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

Besides function calls, operations such as raw pointer dereferencing, accessing static mut values, and reading or writing union fields can be treated in the same way as unsafe callees with specific safety requirements, and the same rules apply.


## 3 Stucts

A struct is a user-defined data type composed of fields. Its behavior is defined through associated functions within `impl` blocks.
- A method is a special associated function that takes self as the first parameter.
- Associated functions without a receiver are commonly used for constructors or type-level operations.
- Struct instances can also be created using struct literals, e.g., `Foo { field1: value, ... }`.

### 3.1 Safety Rules
- **Struct Safety Rule 1**: If none associated functions of a struct contain unsafe code, the struct cannot cause undefined behavior and all its associated functions can be declared as safe.
- **Struct Safety Rule 2**: For associated functions without a receiver, their safety rules are generally the same with the safety rules defined for [free functions](#2-free-functions).

Before discussing methods involving unsafe code, we first define the type invariant of a struct.
A type invariant specifies the conditions that all instances of the type must satisfy, regardless of which constructor is used to create them.
Type invariants are widely used in Rust-for-Linux, e.g., [IovIterSource](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/iov.rs#L38-L49), [List](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/list.rs#L31-L266), [ListLinks](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/list.rs#L367-L375), [Cursor](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/list.rs#L945-L952), [CursorPeek](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/list.rs#L1100-L1107).

If a constructor is safe, its use should always guarantee that the type invariants are upheld, forming the basis of Struct Safety Rule 3.
In practice, the domain of the invariant can be defined with respect to the potential undefined behaviors that might be triggered by other methods.

- **Struct Safety Rule 3**: A constructor can be declared safe if it guarantees that the type invariant of the struct is satisfied; otherwise, it must be declared unsafe.

The safety of a struct’s methods depends on whether they contain unsafe code:
- **Struct Safety Rule 4**: A method that contains no unsafe code can be declared safe if it does not violate the type invariant of the struct.
- **Struct Safety Rule 5**: If a method contains unsafe code, it can be declared safe only if both of the following conditions are met:
  - 1) It does not violate the type invariant of the struct.
  - 2) All safety requirements of its internal unsafe code are satisfied.
   
Note that type invariants play a key role in preventing the safety of methods from depending on the behavior of constructors, and vice versa.
 
### 3.2 Safety Comments
- **Struct Comments Rule 1** (Recommended): Each struct with unsafe constructors should document the type invariant.
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

Both implementations satisfy Rust’s soundness requirement. For the first type, examples can be found in the Rust standard library, such as [DwarfReader](https://github.com/rust-lang/rust/blob/7d8ebe3128fc87f3da1ad64240e63ccf07b8f0bd/library/std/src/sys/personality/dwarf/mod.rs#L15-L69) and [Unique](https://github.com/rust-lang/rust/blob/7d8ebe3128fc87f3da1ad64240e63ccf07b8f0bd/library/core/src/ptr/unique.rs#L94-L156); the second type is commonly used in Rust-for-Linux, with examples including [IovIterSource](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/iov.rs#L38-L49), [Resource](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/io/resource.rs#L75-L169), and [Bitmap](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/bitmap.rs#L17-L90).

## 4 Traits
A trait defines a collection of associated items (typically functions) that can be shared across multiple types. Traits may provide default implementations for some items; these implementations are automatically available to implementing types unless explicitly overridden.
- When a type implements a trait, the trait’s functions behave like associated functions of the type.
- Similar to those introduced in structs, these associated functions may or may not have a receiver.

### 4.1 Safety Rules
- **Trait Safety Rule 1**: Declare a trait as unsafe when the correctness of its implementations is required to prevent undefined behavior in safe code.
- **Trait Safety Rule 2**: A trait method should be unsafe if its correct use depends on safety guarantees that must be enforced by the caller.

### 4.2 Safety Comments
- **Trait Comments Rule 1**: An unsafe trait must document the safety invariants that all implementations are required to uphold.
- **Trait Comments Rule 2**: Each unsafe method of a trait must clearly document the safety requirements that callers must satisfy.
    - The safety requirements of an unsafe method are distinct from the trait invariants.
    - The safety requirements of an unsafe method must be specified in the trait definition, where they form part of the trait’s contract.
    - Implementations should not introduce stricter safety requirements than those declared by the trait.

- **Trait Comments Rule 3**  (Recommended): Implementations of unsafe traits are encouraged to justify why the required invariants are satisfied.


### 4.3 Example Cases

The following example introduces a trait `Buffer` that should be declared as unsafe, because incorrectly implementing it may introduce undefined behavior. 
The trait defines an unsafe method `get_unchecked`. It cannot be declared as safe because its safety requirements cannot be guaranteed solely by the trait invariant.
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

## 5 Visibility
The basic unit of Rust software is a crate. Each crate can contain one or more modules, which in turn can contain submodules, forming a tree-like structure.
[Visibility](https://doc.rust-lang.org/reference/visibility-and-privacy.html) plays a critical role in ensuring safety, because it determines which and how APIs are accessible.
- Module-based visibility:
  - By default, functions, structs, and struct fields are only visible within the module in which they are defined (i.e., private).
  - Exception: Items of a public trait are automatically public, regardless of whether they are explicitly marked `pub`; variants of enumeration types are public if the enum itself is public.
- Visibility granularity: Rust supports different scopes:
  -  `pub`: the item is visible outside the crate.
  -  `pub(crate)`: the item is visible anywhere within the current crate.
  -  `pub(in path)`: the item is visible only within the specified module path.
  -  `pub(super)`: the item is visible to the parent module.

A program component can rely on visibility restrictions to maintain sound abstractions. Real-world Rust projects, however, may adopt different levels of soundness guarantees.
- **Struct-level Soundness Criterion**: All uses of a struct’s safe items (or unsafe items when their safety requirements are satisfied) must not cause undefined behavior, even within the same module.

This is the strongest criterion because all private methods and fields are accessible within the module.
The struct cannot rely on visibility to enforce soundness.
Ensuring this can be challenging, particularly when considering struct literals, which allow creation of instances without going through constructors that uphold the type invariant.

- **Weak Struct-level Soundness Criterion**: This criterion ignores the risk that a type instance could be created or modified via a struct literal that violates the struct’s invariant.

This criterion is the one most commonly employed by the Rust standard library. 
For example, [Vec](https://github.com/rust-lang/rust/blob/7d8ebe3128fc87f3da1ad64240e63ccf07b8f0bd/library/alloc/src/vec/mod.rs#L440) is defined as follows: 
```rust
pub struct Vec<T, #[unstable(feature = "allocator_api", issue = "32838")] A: Allocator = Global> {
    buf: RawVec<T, A>,
    len: usize,
}
```
Developers within the same module can easily create a `Vec` instance or modify `len` in a way that violates the type invariant. 
However, this risk may hopefully be mitigated in the near future through the use of [unsafe fields](https://rust-lang.github.io/rust-project-goals/2025h1/unsafe-fields.html) in the nearby future. 

Such examples are also common in Rust-for-Linux, for instance [List](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/list.rs#L31-L266), as shown below. The type invariant can be enforced via the new constructor. However, developers working within the same module can bypass the invariant by using a struct literal.
```rust
/// # Invariants
///
/// * If the list is empty, then `first` is null. Otherwise, `first` points at the `ListLinks` field of the first element in the list.
/// * All prev/next pointers in `ListLinks` fields of items in the list are valid and form a cycle.
/// * For every item in the list, the list owns the associated [`ListArc`] reference and has exclusive access to the `ListLinks` field.
pub struct List<T: ?Sized + ListItem<ID>, const ID: u64 = 0> {
    first: *mut ListLinksFields,
    _ty: PhantomData<ListArc<T, ID>>,
}

impl<T: ?Sized + ListItem<ID>, const ID: u64> List<T, ID> {
    /// Creates a new empty list.
    pub const fn new() -> Self {
        Self {
            first: ptr::null_mut(),
            _ty: PhantomData,
        }
    }
}
```

Note that all structs and associated functions must be sound regardless of whether they are public or private. In other words, a struct cannot rely on visibility to maintain soundness.
For example, in Rust-for-Linux’s `List`, the private method [insert_inner](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/list.rs#L489-L531) is declared as unsafe. 
This reflects the fact that, even though the function is not part of the public API, if its correct use depends on invariants that cannot be enforced by the type system alone, it must be declared as unsafe.

- **Module-level Soundness Criterion**: All uses of the module’s public safe items (or unsafe items when their safety requirements are satisfied) from outside the module must not cause undefined behavior.
- **Crate-level Soundness Criterion**: All uses of the crate’s public safe items (or unsafe items when their safety requirements are satisfied) from outside the crate must not cause undefined behavior.

The two criteria differ only in scope. 
Some systems may nevertheless prefer such safety criteria because certain APIs are designed for specific usage contexts and can rely on global or system state to ensure safety. 

This design pattern is common in the operating system [Asterinas](https://github.com/asterinas/asterinas). 
Consider the function [`call_ostd_main`](https://github.com/asterinas/asterinas/blob/6d2ff13a639b9fff72842869cfd78f78ff21bbf2/ostd/src/boot/mod.rs#L122-L137), shown below.
The function directly calls two unsafe functions, yet itself is declared as safe.
This is because `call_ostd_main` is only callable within the crate, and its safety can be guaranteed at all of its current callsites, which are restricted to architecture-specific boot code, e.g., [__linux_boot](https://github.com/asterinas/asterinas/blob/6d2ff13a639b9fff72842869cfd78f78ff21bbf2/ostd/src/arch/x86/boot/linux_boot/mod.rs#L203-L222).
```rust
pub(crate) fn call_ostd_main() -> ! {
    // The entry point of kernel code, which should be defined by the package that
    // uses OSTD.
    unsafe extern "Rust" {
        fn __ostd_main() -> !;
    }

    // SAFETY: The function is called only once on the BSP.
    unsafe { crate::init() };

    // SAFETY: This external function is defined by the package that uses OSTD,
    // which should be generated by the `ostd::main` macro. So it is safe.
    unsafe {
        __ostd_main();
    }
}
```


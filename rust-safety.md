# Rust Safety Standard
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

### 2.2 Design Choices
In general, there are two scenarios in which a developer declares a function unsafe.
- **Dependent unsafe (mandatory)**:
  -  A free function invokes unsafe code from its dependencies, and the safety requirements of that unsafe code cannot be fully discharged by the function itself. As a result, the function must be declared unsafe to propagate the remaining safety requirements to its caller.
  -  An associated function may violate a struct’s type invariant, which can in turn affect other methods of the struct that rely on unsafe code. Regardless of whether it contains unsafe code, such a function should be declared unsafe, with safety requirements specifying how the invariant must be upheld.
  
- **New unsafe (by design)**:  
  - Besides dependent unsafe, developers may deliberately introduces additional safety contracts that must be upheld by the caller in order to use the function safely.
  - When there is confusion about whether a function should be declared safe or unsafe, remember that safety invariants come before `unsafe`.
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
Therefore, their constructors `new` are both declared safe.  

These structs are then used by [`LocalWaker`](https://doc.rust-lang.org/core/task/struct.LocalWaker.html), which enforces the type invariants. As a result, its constructors [`new`](https://doc.rust-lang.org/core/task/struct.LocalWaker.html#method.new) and [`from_raw`](https://doc.rust-lang.org/core/task/struct.LocalWaker.html#method.from_raw) are declared unsafe.


Note that the introduction of new unsafe code is discouraged unless necessary.
Sometimes, introducing new unsafe functions can bring benefits and help avoid exposing additional unsafe functions.
This is essential to prevent the proliferation of unsafe code and the degradation of overall safety in Rust projects.

### 2.3 Decoupling
To provide clearer and more actionable guidance, decoupling the relationships between safety responsibilities is very important. 
One key decoupling strategy is to separate the safety of free functions and structs.
In particular, we conceptually treat struct construction and field access as follows: a free function creates a struct instance only via the struct’s constructors (including struct literals), and any direct field access is modeled as an invocation of the struct’s implicit methods.

For example, in the following code, we can treat `new_unchecked` as a free function that returns a an `EvenNumber` instance, instead of a constructor; the real constructor is the literal constructor of the tuple struct `EvenNumber()`:

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

Similarly, one may create objects of other types that enclose the struct, such as `Box<EvenNumber>`. These should also rely on the constructor of `EvenNumber`:
```rust
pub unsafe fn new_unchecked(x: u32) -> Box<EvenNumber> {
    Box::new(EvenNumber::new_unchecked(x))
}
```

In the same way, direct field assignments (e.g., `even_number.0 = ...`) can be treated as invoking the literal methods of the struct.

## 3 Establish A Soundness Criterion
Existing soundness criteria require only that safe code cannot cause undefined behavior, but they do not clearly specify the scope in which such code usage is considered. 
[Visibility](https://doc.rust-lang.org/reference/visibility-and-privacy.html) plays a critical role in soundness, because it determines which and how APIs are accessible.

Therefore, the first step for a Rust project to follow this standard is to establish a visibility-aware soundness criterion that enables systematic soundness auditing.

### 3.1 Visibility
The basic unit of Rust software is a crate. Each crate can contain one or more modules, which in turn can contain submodules, forming a tree-like structure.
- Module-based visibility:
  - By default, functions, structs, and struct fields are only visible within the module in which they are defined (i.e., private).
  - Exception: Items of a public trait are automatically public, regardless of whether they are explicitly marked `pub`; variants of enumeration types are public if the enum itself is public.
- Visibility granularity: Rust supports different scopes:
  -  `pub`: the item is visible outside the crate.
  -  `pub(crate)`: the item is visible anywhere within the current crate.
  -  `pub(in path)`: the item is visible only within the specified module path.
  -  `pub(super)`: the item is visible to the parent module.

A program component can rely on visibility restrictions to maintain sound abstractions. 

### 3.2 Visibility-based Soundness Criteria
Before determining whether each function should be declared safe or unsafe, the project owner should be explicit about which criterion is being adopted, as it guides the subsequent rules for soundness checking. 
The project owner may select the soundness guarantees that best suit their project.

- **Module-level Soundness Criterion** (default): All uses of a module’s public safe items (or unsafe items, provided their safety requirements are met) from outside the module must not lead to undefined behavior.

This is the default soundness criterion adopted by the Rust standard library, as a module represents the smallest unit of accessibility control.

For example, the following code is considered sound even if it is possible to store an arbitrary raw pointer into HOOK using only safe code. 
This criterion has been confirmed by the library team in [issues/152078](https://github.com/rust-lang/rust/issues/152078).
```rust
static HOOK: AtomicPtr<()> = AtomicPtr::new(ptr::null_mut());

pub fn rust_oom(layout: Layout) -> ! { 
     crate::sys::backtrace::__rust_end_short_backtrace(|| { 
         let hook = HOOK.load(Ordering::Acquire); 
         let hook: fn(Layout) = 
             if hook.is_null() { default_alloc_error_hook } else { unsafe { mem::transmute(hook) } }; 
         hook(layout); 
         crate::process::abort() 
     }) 
 } 
```

- **Crate-level Soundness Criterion** (relaxed): All uses of the crate’s public safe items (or unsafe items when their safety requirements are satisfied) from outside the crate must not cause undefined behavior.

This is a more relaxed criterion while still providing soundness guarantees.
Some systems may nevertheless prefer such safety criteria because certain APIs are designed for specific usage contexts and can rely on global or system state to ensure safety.

- **Struct-level Soundness Criterion** (stricter): All uses of a struct’s safe items (or unsafe items when their safety requirements are satisfied) must not cause undefined behavior, even within the same module.

This is the most strict criterion. 
Since all private methods and fields are accessible within a module, a struct cannot rely on visibility to prevent misuse from code in the same module.
It is equivelent to the module-level soundness criterion if the module has only one struct.

Ensuring struct-level soundness can be challenging, particularly when considering struct literals, which allow creation of instances without going through constructors that uphold the type invariant.
This might be achievable in the future through the use of [unsafe fields](https://rust-lang.github.io/rust-project-goals/2025h1/unsafe-fields.html). 
For example, [Vec](https://github.com/rust-lang/rust/blob/7d8ebe3128fc87f3da1ad64240e63ccf07b8f0bd/library/alloc/src/vec/mod.rs#L440) is defined as follows. 
Developers within the same module can easily create a `Vec` instance or modify `len` via struct literals in ways that violate the type invariant.
```rust
pub struct Vec<T, #[unstable(feature = "allocator_api", issue = "32838")] A: Allocator = Global> {
    buf: RawVec<T, A>,
    len: usize,
}
```

### 3.3 Principle of Least Unsound Scope 

The soundness criterion serves as the last line of defense against undefined behavior. 
Whenever possible, safety should be ensured earlier by restricting the scope of unsafe code or enforcing invariants locally.

For example, in Rust-for-Linux’s `List`, the private method [insert_inner](https://github.com/Rust-for-Linux/linux/blob/08afcc38a64cec3d6065b90391afebfde686a69a/rust/kernel/list.rs#L489-L531) is declared unsafe with clearly documented safety requirements to prevent misuse. 
This demonstrates that even if a function is not part of the public API, it is recommended to mark it unsafe when its correct use depends on invariants that cannot be enforced by the type system alone.


## 4 Rules for Free Functions
A free function is a function defined at the module level that can be called directly by its path rather than through an instance or type.

### 4.1 Safety Rules
The soundness of free functions is relatively straightforward to justify.

- **Function Safety Rule 1**: If a free function contains unsafe code, it can be declared safe only if all conditions required for the safe use of that unsafe code are met.
- **Function Safety Rule 2**: If a free function contains no unsafe code, there is no mandatory contract to be upheld, and it can be declared safe.

Note that Function Safety Rule 2 continues to hold even when the function calls other functions that contain unsafe code, provided that those callees themselves satisfy Function Safety Rule 1 and do not impose additional safety contracts.

## 4.2 Safety Comments
There are two mandatory rules and one recommended practice for documenting the safety requirements of free functions:
- **Function Comments Rule 1**: All unsafe functions must document their safety requirements.
- **Function Comments Rule 2**: Safety requirements must be externally verifiable and must not depend on the function’s internal implementation.
- **Function Comments Rule 3** (Recommended):  At the callsite of unsafe code, users are encouraged to justify why the safety requirements are satisfied.

### 4.3 Example Cases

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


## 5 Rules for Stucts

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

### 5.1 Safety Rules

- **Struct Safety Rule 1**: A constructor can be declared safe if it guarantees that the type invariant of the struct is satisfied; otherwise, it must be declared unsafe.
- **Struct Safety Rule 2**: For associated functions without a receiver, their safety rules are generally the same with the safety rules defined for [free functions](#3-free-functions).
- **Struct Safety Rule 3**: A method that contains no unsafe code can be declared safe if it does not violate the type invariant of the struct.
- **Struct Safety Rule 4**: If a method contains unsafe code, it can be declared safe only if both of the following conditions are met:
  - 1) It does not violate the type invariant of the struct.
  - 2) All safety requirements of its internal unsafe code are satisfied.
 
### 5.2 Safety Comments
- **Struct Comments Rule 1** (Recommended): Each struct with unsafe constructors should document the type invariant.
- **Struct Comments Rule 2**: Each unsafe associated function should document its safety requirements:
  - 1) The requirements should not depend on other functions of the struct.
  - 2) They must be externally verifiable and must not depend on the function’s internal implementation.
- **Struct Comments Rule 3** (Recommended):  At the callsite of unsafe code, users are encouraged to justify why the safety requirements are satisfied.

### 5.3 Example Cases

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

## 6 Rules for Traits
A trait defines a collection of associated items (typically functions) that can be shared across multiple types. Traits may provide default implementations for some items; these implementations are automatically available to implementing types unless explicitly overridden.
- When a type implements a trait, the trait’s functions behave like associated functions of the type.
- Similar to those introduced in structs, these associated functions may or may not have a receiver.

### 6.1 Safety Rules
- **Trait Safety Rule 1**: Declare a trait unsafe when the correctness of its implementations is required to prevent undefined behavior in safe code.
- **Trait Safety Rule 2**: A trait method should be unsafe if its correct use depends on safety guarantees that must be enforced by the caller.

### 5.2 Safety Comments
- **Trait Comments Rule 1**: An unsafe trait must document the safety invariants that all implementations are required to uphold.
- **Trait Comments Rule 2**: Each unsafe method of a trait must clearly document the safety requirements that callers must satisfy.
    - The safety requirements of an unsafe method are distinct from the trait invariants.
    - The safety requirements of an unsafe method must be specified in the trait definition, where they form part of the trait’s contract.
    - Implementations should not introduce stricter safety requirements than those declared by the trait.

- **Trait Comments Rule 3**  (Recommended): Implementations of unsafe traits are encouraged to justify why the required invariants are satisfied.


### 6.3 Example Cases

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


# Rust Safety Standard
## 1. Background
Rust’s safety guarantees rely heavily on the correct use of unsafe code. 
However, maintaining safety becomes increasingly challenging as real-world crates grow in size, particularly when attempting to enforce a clear separation between safe and unsafe components. 
Rust currently provides only the following high-level [soundness requirement](https://rust-lang.github.io/unsafe-code-guidelines/glossary.html#soundness-of-code--of-a-library):
> We say that a library (or an individual function) is sound if it is impossible for safe code to cause Undefined Behavior using its public API.

On the one hand, this definition does not provide actionable criteria for determining which functions should be declared safe or unsafe.

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
    pub fn unsafe get(&self) -> &[u8] { 
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
    pub fn unsafe from(p: *mut u8, l: usize) -> Foo {
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
The design space becomes even more complex when visibility boundaries and trait implementations are taken into account.

On the other hand, when a function is declared unsafe, developers must clearly understand the conditions required for its correct use. 
This raises another important question: what information should be documented to properly specify the safety requirements of an unsafe function?

This document aims to provide actionable guidance derived from extensive reviews of unsafe code in several Rust projects, particularly the Rust standard library. 
Our recommendations are grounded in the theoretical foundation presented in [A Trace-based Approach for Code Safety Analysis](https://arxiv.org/pdf/2510.10410). 

## 2. Determining Function Safety

## 3. Writing Safety Comments

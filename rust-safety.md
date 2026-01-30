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


## 4 Traits

## 5 Visibility

## 6 More

### 6.1 Static Mut

## 7 Summary


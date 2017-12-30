- Feature Name: total_fn
- Start Date: 2017-12-29
- RFC PR: 
- Rust Issue: 

# Summary
[summary]: #summary

Allow functions to be defined as "total", which promises that the function will always return some value and will not, e.g. panic, regardless of input. This promise is validated by the compiler and is sound but incomplete: all functions explicitly defined as total actually are, but it will be possible to express some functions which are total in reality but cannot be validated as such. 

# Motivation
[motivation]: #motivation

In contexts such as high-uptime web services, industrial control systems or even [trains], it is business-critical or even safety-critical that software continues running, even in failure cases. While Rust goes much further than some other languages in encouraging the developer to handle gracefully errors with the likes of `Option`, `Result`, `try!` and a lack of exceptions, function calls are nonetheless somewhat opaque in their stability guarantees -- while `unwrap` and the like are immediately recognisable as dangerous, it is not possible in general to tell from a given function's signature or call site whether it might result in ending the thread in a panic.

The industry-standard solution to this is testing, but as [the aphorism](https://www.cs.utexas.edu/users/EWD/ewd02xx/EWD249.PDF) goes, tests can only prove the presence of bugs, rather than their absence. The compiler, however, *can* prove the absence of bugs, and being able to prove the absence of unexpected aborts would be worthwhile in the safety-critical contexts Rust is targeting, being more exhaustive and reliable than any amount of testing.

(TBC?)

[trains]: https://users.rust-lang.org/t/using-rust-for-railway-software-no-core/13848

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation
In order to use a function as total, it must be explicitly annotated as such.

```rust
total fn is_nonnegative(x: i32) -> bool {
    if (x < 0) { false }
    else       { true  }
}
```

(TODO: total closures?)

This annotation promises to the compiler that the function will not fail to return a value under any circumstances or combination of input values. The annotation is *strictly optional* - a function does not have to be annotated as total, whether it actually is or not, if it is only called from non-total callers. The annotation implies the following restrictions:

- total functions cannot contain `loop` or `while`, even if those loops also include `break` statements. 
- any function called from a `total` function must itself be total, including functions called implicitly in syntax desugaring, e.g. `Deref`
- as a special case of the above, `for` loops inside total functions require that the iterator implement a more restrictive subtrait of `Iterator` which is guaranteed to have a pessimistic size bound and a `total` implementation of `next`. (TODO: elaborate on this)
- any function "returning" an unhabited type such as `!` is not total by definition; marking it as such is an error in itself.
- FFI functions are not total by definition. (They are not visible to the compiler and so cannot be proven to halt.)
- a total function cannot call itself. (TBD: or any of its callers in the same crate?)

TBC

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TBD

# Drawbacks
[drawbacks]: #drawbacks

TBD

# Rationale and alternatives
[alternatives]: #alternatives

TBD

# Unresolved questions
[unresolved]: #unresolved-questions
- Is this the most understandable/teachable syntax/naming? Would "halting fn" be easier to understand?
- Do we want total blocks too?
- Discussion on #rust-lang suggested that unbounded loops be allowed inside `unsafe` blocks within a `total fn`; is this a good idea? What else might be allowed?
- How does totalness interact with traits?
- What is the proper relationship between `total fn` and `const fn`? Is one a subset of the other? Is it coherent to write a total function that cannot be evaluated at compile time?

TBD

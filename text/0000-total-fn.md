# Total Fn
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

# Fundamentals
[fundamentals]: #fundamentals
In order to use a function as total, it must be explicitly annotated as such.

```rust
total fn is_nonnegative(x: i32) -> bool {
    if (x < 0) { false }
    else       { true  }
}
```

(TODO: how do we do total closures? Do we have total blocks?)

This annotation promises to the compiler that the function will not fail to return a value under any circumstances (except possibly stack overflow, see below) or combination of input values. The annotation is *strictly optional* - a function does not have to be annotated as total, whether it actually is or not, if it is only called from non-total callers. The annotation implies the following restrictions:

- total functions cannot contain `loop` or `while`, even if those loops also include `break` statements. 
- any function called from a `total` function must itself be total, including functions called implicitly in syntax desugaring, e.g. `Deref`
- as a special case of the above, `for` loops inside total functions require that the iterator implement a new trait, `BoundedIterator`, defined below, as opposed to the standard `Iterator`. 
- any function "returning" an uninhabitable type such as `!` is not total by definition; marking it as such is an error.
- FFI functions are not total by definition. (They are not visible to the compiler and so cannot be proven to halt.)
- A `total` function cannot call a function accepting a function parameter if the value being passed is not known to be `total`, even if the callee function is `total`. (Calling a total function with a non-total function as a parameter "decays" into a non-total call, ala `const`.)

It should be relatively obvious that all of the above conditions are required for a function to always halt regardless of what subcalls it may make to other functions and more generally, regardless of the specific contents of its body. 

However, the following questions do not have answers as obvious and natural. The following proposals should be taken as suggestions and prompts for further discussion rather than the strong implications of the previous list.

1. Should a `total` function be allowed to recurse, and if so under what circumstances? The most conservative answer, and the one this RFC proposes pending discussion otherwise, is that `total` functions are not allowed to recurse, even indirectly. This would add the following condition to the above  list: a `total` function cannot call itself or any of its callers - it's call graph must be completely acyclic. 
2. Should there be an "escape hatch" of some form allowing code that cannot be proven to halt to be included inside a `total` function? Would this take the form of `unsafe` allowing unbounded loops, or an `untotal` block? The position here is that such a mechanism is outside the scope of this RFC and is assumed to not exist. 
3. Is a `total` function allowed to fail because it runs out of stack space? While it is relatively easy to determine the maximum amount of stack space a non-recursive function will use, determining how much stack space is actually availiable is complex, varies wildly by architecture, and is not necessarily a meaningful question to ask at compile-time. In the interests of conservatism, this RFC avoids trying to answer this problem and instead proposes an exception to the rule that a `total` function always halts: **a `total` function is allowed to overflow the stack**.

TBC

## `BoundedIterator`

In order to make writing total functions ergonomic, it should be possible to express loops that are known to be bounded. `BoundedIterator` is a new trait introduced to allow this, with conceptually the following implementation:

```rust 
trait BoundedIterator : Iterator {
    total fn size_bounds(&self) -> (usize, usize);
    total fn next(&mut self) -> Option<Self::Item>;
}
```

- The `next` method has exactly the same semantics as the corresponding method on `Iterator`, except that it is also required to be total. 
- `size_bounds` is required to return a *pessmistic* bound on the number of th remaining elements, with the tuple elements being the lower and upper bounds respectively. These elements should be equal if the number of elements remaining is exactly determinable. Inside a total function, a `for` loop will iterate `size_bounds().1` elements or up until the first `None`, whichever happens sooner. This implies it will iterate at most `usize::MAX` elements before breaking, even if the iterator is still returning `Some`.

# Total fns and Traits

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

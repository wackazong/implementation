# implementation

A crate for targeting and accessing _actual implementation_.

Take an example trait:

```rust
trait ScrapeTheInternet {
    fn scrape_the_internet(&self) -> Vec<Website>;
}
```

The trait represents some abstract computation. The trait exports a method signature that
can be implemented by types. In this case, we can imagine what a true implementation of the
trait will do: _Actually scrape the internet_.

`implementation` provides _impl target_ types for traits having the following semantics:

* There is only one actual, true implementation.
* Other implementations may exist, but they are interpreted as _fake_, _mocked_ in some way.

`implementation` enables a standardized way of writing these actual implementations in a way
that allows the actual `Self`-receiver type to be unknown.

* For `self` receivers, implement for [`Impl<T>`](Impl).
* For `&self` receivers, implement for [`Impl<&T>`](Impl).
* For `&mut self` receivers, implement for [`Impl<&mut T>`](Impl).

## Usage
To define the actual, generic implementation of `ScrapeTheInternet`, we can write the following impl:

```rust
impl<'a, T> ScrapeTheInternet for implementation::Impl<&'a T> {
    fn scrape_the_internet(&self) -> Vec<Website> {
        todo!("find all the web pages, etc")
    }
}
```

This code implements the trait for the pointer-like type [Impl], and by doing that we have asserted
that it is the actual, true implementation.

The implementation is fully generic, and works for any `T`. This implementation can be invoked
by converting `T` into a `Ref<'_, T>` by calling [BorrowImpl::borrow_impl]:

```rust
use implementation::BorrowImpl;

struct MyType;

let my_type = MyType;
my_type
    .borrow_impl()
    .scrape_the_internet();
```

### Trait bounds
The advantage of keeping trait implementations generic, is that the self type might
live in a downstream crate. Let's say we need to access a configuration parameter
from `scrape_the_internet`. E.g. the maximum number of pages to scrape:

```rust
use implementation::Impl;

trait GetMaxNumberOfPages {
    fn get_max_number_of_pages(&self) -> Option<usize>;
}

impl<'a, T> ScrapeTheInternet for Impl<&'a T>
    where Impl<&'a T>: GetMaxNumberOfPages
{
    fn scrape_the_internet(&self) -> Vec<Website> {
        let max_number_of_pages = self.get_max_number_of_pages();
        todo!("find all the web pages, etc")
    }
}
```

Now, `Impl<&T>` also need to provide an actual implementation of `GetMaxNumberOfPages`.

## Explanation

This crate is the solution to a trait coherence problem.

Given the trait above, we would like to provide an actual and a mocked implementation.
We might know what its actual implementation looks like as an algorithm, but
_not what type it should be implemented for_. There could be several reasons
to have a generic Self:

* The `Self` type might live in a downstream crate
* It is actually designed to work generically

If we had used a generic Self type (`impl<T> DoSomething for T`), the trait
would be unable to also have distinct fake implementations, because that would break
the coherence rules: A generic ("blanket") impl and a specialized
impl are not allowed to exist at the same time, because that would lead to ambiguity.

To solve that, a concrete type is needed as implementation target. But that
type is allowed to be generic _internally_. It's just the root level that
needs to be a concretely named type.

That type is the [Impl] type.

When we use this implementation, we can create as many fake implementations as we want.

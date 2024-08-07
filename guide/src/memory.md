# Memory management

<div class="warning">

⚠️ Warning: API update in progress 🛠️

PyO3 0.21 has introduced a significant new API, termed the "Bound" API after the new smart pointer `Bound<T>`.

This section on memory management is heavily weighted towards the now-deprecated "GIL Refs" API, which suffered from the drawbacks detailed here as well as CPU overheads.

See [the smart pointer types](./types.md#pyo3s-smart-pointers) for description on the new, simplified, memory model of the Bound API, which is built as a thin wrapper on Python reference counting.
</div>

Rust and Python have very different notions of memory management.  Rust has
a strict memory model with concepts of ownership, borrowing, and lifetimes,
where memory is freed at predictable points in program execution.  Python has
a looser memory model in which variables are reference-counted with shared,
mutable state by default. A global interpreter lock (GIL) is needed to prevent
race conditions, and a garbage collector is needed to break reference cycles.
Memory in Python is freed eventually by the garbage collector, but not usually
in a predictable way.

PyO3 bridges the Rust and Python memory models with two different strategies for
accessing memory allocated on Python's heap from inside Rust. These are
GIL Refs such as `&'py PyAny`, and GIL-independent `Py<Any>` smart pointers.

## GIL-bound memory

PyO3's GIL Refs such as `&'py PyAny` make PyO3 more ergonomic to
use by ensuring that their lifetime can never be longer than the duration the
Python GIL is held.  This means that most of PyO3's API can assume the GIL is
held. (If PyO3 could not assume this, every PyO3 API would need to take a
`Python` GIL token to prove that the GIL is held.)  This allows us to write
very simple and easy-to-understand programs like this:

```rust,ignore
# #![allow(unused_imports)]
# use pyo3::prelude::*;
# use pyo3::types::PyString;
# fn main() -> PyResult<()> {
# #[cfg(feature = "gil-refs")]
Python::with_gil(|py| -> PyResult<()> {
    #[allow(deprecated)] // py.eval() is part of the GIL Refs API
    let hello = py
        .eval("\"Hello World!\"", None, None)?
        .downcast::<PyString>()?;
    println!("Python says: {}", hello);
    Ok(())
})?;
# Ok(())
# }
```

Internally, calling `Python::with_gil()` creates a `GILPool` which owns the
memory pointed to by the reference.  In the example above, the lifetime of the
reference `hello` is bound to the `GILPool`.  When the `with_gil()` closure ends
the `GILPool` is also dropped and the Python reference counts of the variables
it owns are decreased, releasing them to the Python garbage collector.  Most
of the time we don't have to think about this, but consider the following:

```rust,ignore
# #![allow(unused_imports)]
# use pyo3::prelude::*;
# use pyo3::types::PyString;
# fn main() -> PyResult<()> {
# #[cfg(feature = "gil-refs")]
Python::with_gil(|py| -> PyResult<()> {
    for _ in 0..10 {
        #[allow(deprecated)] // py.eval() is part of the GIL Refs API
        let hello = py
            .eval("\"Hello World!\"", None, None)?
            .downcast::<PyString>()?;
        println!("Python says: {}", hello);
    }
    // There are 10 copies of `hello` on Python's heap here.
    Ok(())
})?;
# Ok(())
# }
```

We might assume that the `hello` variable's memory is freed at the end of each
loop iteration, but in fact we create 10 copies of `hello` on Python's heap.
This may seem surprising at first, but it is completely consistent with Rust's
memory model.  The `hello` variable is dropped at the end of each loop, but it
is only a reference to the memory owned by the `GILPool`, and its lifetime is
bound to the `GILPool`, not the for loop.  The `GILPool` isn't dropped until
the end of the `with_gil()` closure, at which point the 10 copies of `hello`
are finally released to the Python garbage collector.

<div class="warning">

⚠️ Warning: `GILPool` is no longer the preferred way to manage memory with PyO3 🛠️

PyO3 0.21 has introduced a new API known as the Bound API, which doesn't have the same surprising results. Instead, each `Bound<T>` smart pointer releases the Python reference immediately on drop. See [the smart pointer types](./types.md#pyo3s-smart-pointers) for more details.
</div>


In general we don't want unbounded memory growth during loops!  One workaround
is to acquire and release the GIL with each iteration of the loop.

```rust,ignore
# #![allow(unused_imports)]
# use pyo3::prelude::*;
# use pyo3::types::PyString;
# fn main() -> PyResult<()> {
# #[cfg(feature = "gil-refs")]
for _ in 0..10 {
    Python::with_gil(|py| -> PyResult<()> {
        #[allow(deprecated)] // py.eval() is part of the GIL Refs API
        let hello = py
            .eval("\"Hello World!\"", None, None)?
            .downcast::<PyString>()?;
        println!("Python says: {}", hello);
        Ok(())
    })?; // only one copy of `hello` at a time
}
# Ok(())
# }
```

It might not be practical or performant to acquire and release the GIL so many
times.  Another workaround is to work with the `GILPool` object directly, but
this is unsafe.

```rust,ignore
# #![allow(unused_imports)]
# use pyo3::prelude::*;
# use pyo3::types::PyString;
# fn main() -> PyResult<()> {
# #[cfg(feature = "gil-refs")]
Python::with_gil(|py| -> PyResult<()> {
    for _ in 0..10 {
        #[allow(deprecated)] // `new_pool` is not needed in code not using the GIL Refs API
        let pool = unsafe { py.new_pool() };
        let py = pool.python();
        #[allow(deprecated)] // py.eval() is part of the GIL Refs API
        let hello = py
            .eval("\"Hello World!\"", None, None)?
            .downcast::<PyString>()?;
        println!("Python says: {}", hello);
    }
    Ok(())
})?;
# Ok(())
# }
```

The unsafe method `Python::new_pool` allows you to create a nested `GILPool`
from which you can retrieve a new `py: Python` GIL token.  Variables created
with this new GIL token are bound to the nested `GILPool` and will be released
when the nested `GILPool` is dropped.  Here, the nested `GILPool` is dropped
at the end of each loop iteration, before the `with_gil()` closure ends.

When doing this, you must be very careful to ensure that once the `GILPool` is
dropped you do not retain access to any owned references created after the
`GILPool` was created.  Read the documentation for `Python::new_pool()`
for more information on safety.

This memory management can also be applicable when writing extension modules.
`#[pyfunction]` and `#[pymethods]` will create a `GILPool` which lasts the entire
function call, releasing objects when the function returns. Most functions only create
a few objects, meaning this doesn't have a significant impact. Occasionally functions
with long complex loops may need to use `Python::new_pool` as shown above.

<div class="warning">

⚠️ Warning: `GILPool` is no longer the preferred way to manage memory with PyO3 🛠️

PyO3 0.21 has introduced a new API known as the Bound API, which doesn't have the same surprising results. Instead, each `Bound<T>` smart pointer releases the Python reference immediately on drop. See [the smart pointer types](./types.md#pyo3s-smart-pointers) for more details.
</div>

## GIL-independent memory

Sometimes we need a reference to memory on Python's heap that can outlive the
GIL.  Python's `Py<PyAny>` is analogous to `Arc<T>`, but for variables whose
memory is allocated on Python's heap.  Cloning a `Py<PyAny>` increases its
internal reference count just like cloning `Arc<T>`.  The smart pointer can
outlive the "GIL is held" period in which it was created.  It isn't magic,
though.  We need to reacquire the GIL to access the memory pointed to by the
`Py<PyAny>`.

What happens to the memory when the last `Py<PyAny>` is dropped and its
reference count reaches zero?  It depends whether or not we are holding the GIL.

```rust
# #![allow(unused_imports)]
# use pyo3::prelude::*;
# use pyo3::types::PyString;
# fn main() -> PyResult<()> {
# #[cfg(feature = "gil-refs")]
Python::with_gil(|py| -> PyResult<()> {
    #[allow(deprecated)] // py.eval() is part of the GIL Refs API
    let hello: Py<PyString> = py.eval("\"Hello World!\"", None, None)?.extract()?;
    #[allow(deprecated)] // as_ref is part of the GIL Refs API
    {
        println!("Python says: {}", hello.as_ref(py));
    }
    Ok(())
})?;
# Ok(())
# }
```

At the end of the `Python::with_gil()` closure `hello` is dropped, and then the
GIL is dropped.  Since `hello` is dropped while the GIL is still held by the
current thread, its memory is released to the Python garbage collector
immediately.

This example wasn't very interesting.  We could have just used a GIL-bound
`&PyString` reference.  What happens when the last `Py<Any>` is dropped while
we are *not* holding the GIL?

```rust
# #![allow(unused_imports, dead_code)]
# #[cfg(not(pyo3_disable_reference_pool))] {
# use pyo3::prelude::*;
# use pyo3::types::PyString;
# fn main() -> PyResult<()> {
# #[cfg(feature = "gil-refs")]
# {
let hello: Py<PyString> = Python::with_gil(|py| {
    #[allow(deprecated)] // py.eval() is part of the GIL Refs API
    py.eval("\"Hello World!\"", None, None)?.extract()
})?;
// Do some stuff...
// Now sometime later in the program we want to access `hello`.
Python::with_gil(|py| {
    #[allow(deprecated)]  // as_ref is part of the deprecated "GIL Refs" API.
    let hello = hello.as_ref(py);
    println!("Python says: {}", hello);
});
// Now we're done with `hello`.
drop(hello); // Memory *not* released here.
// Sometime later we need the GIL again for something...
Python::with_gil(|py|
    // Memory for `hello` is released here.
# ()
);
# }
# Ok(())
# }
# }
```

When `hello` is dropped *nothing* happens to the pointed-to memory on Python's
heap because nothing _can_ happen if we're not holding the GIL.  Fortunately,
the memory isn't leaked. If the `pyo3_disable_reference_pool` conditional compilation flag
is not enabled, PyO3 keeps track of the memory internally and will release it
the next time we acquire the GIL.

We can avoid the delay in releasing memory if we are careful to drop the
`Py<Any>` while the GIL is held.

```rust
# #![allow(unused_imports)]
# use pyo3::prelude::*;
# use pyo3::types::PyString;
# fn main() -> PyResult<()> {
# #[cfg(feature = "gil-refs")]
# {
#[allow(deprecated)] // py.eval() is part of the GIL Refs API
let hello: Py<PyString> =
    Python::with_gil(|py| py.eval("\"Hello World!\"", None, None)?.extract())?;
// Do some stuff...
// Now sometime later in the program:
Python::with_gil(|py| {
    #[allow(deprecated)] // as_ref is part of the GIL Refs API
    {
        println!("Python says: {}", hello.as_ref(py));
    }
    drop(hello); // Memory released here.
});
# }
# Ok(())
# }
```

We could also have used `Py::into_ref()`, which consumes `self`, instead of
`Py::as_ref()`.  But note that in addition to being slower than `as_ref()`,
`into_ref()` binds the memory to the lifetime of the `GILPool`, which means
that rather than being released immediately, the memory will not be released
until the GIL is dropped.

```rust
# #![allow(unused_imports)]
# use pyo3::prelude::*;
# use pyo3::types::PyString;
# fn main() -> PyResult<()> {
# #[cfg(feature = "gil-refs")]
# {
#[allow(deprecated)] // py.eval() is part of the GIL Refs API
let hello: Py<PyString> =
    Python::with_gil(|py| py.eval("\"Hello World!\"", None, None)?.extract())?;
// Do some stuff...
// Now sometime later in the program:
Python::with_gil(|py| {
    #[allow(deprecated)] // into_ref is part of the GIL Refs API
    {
        println!("Python says: {}", hello.into_ref(py));
    }
    // Memory not released yet.
    // Do more stuff...
    // Memory released here at end of `with_gil()` closure.
});
# }
# Ok(())
# }
```

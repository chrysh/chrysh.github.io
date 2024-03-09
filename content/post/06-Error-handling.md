---
title: Results and Errors
date: '2024-03-09'
categories:
  - Rust
  - C
  - Kernel Driver
  - Linux Kernel
tags:
  - Rust
  - Kernel Driver
  - Linux Kernel
  - Error handling
  - Result type
---

[<img src="/static/img/rusty_penguin_7.jpeg" style="max-width:40%;min-width:40px;float:right;padding:40px" alt="Rusty penguin. Created by DALL·E 3." />](https://github.com/Rust-for-Linux/)

# Rusty error handling

In C, it is common that functions return a integer indicating the success or
failure of the function execution. Extra data can only be modified or returned
through non-`const` pointer parameters. Rust on the other hand uses explicit
error types.  The Result type encapsulates error and success.  In the context of
Rust code in the Linux Kernel, functions that would return an error code in C
would return a `core::result::Result` with `Error` as its error type instead. In
the following subsections, we will mainly look at the file
`rust/kernel/error.rs`, and see how it interacts with the kernel error handling.

## `declare_err` macro
The file `rust/kernel/error.rs` couples Rust errors to Linux Kernel Code. It
starts off by declaring a macro `declare_err`, which expands to the given name
and error code.

```
macro_rules! declare_err {
    ($err:tt $(,)? $($doc:expr),+) => {
        $(
        #[doc = $doc]
        )*
        pub const $err: super::Error = super::Error(-(crate::bindings::$err as i32));
    };
}
```

This macro is used for all the errors used in the Linux kernel, and connects the
error code with its documentation.

```
declare_err!(EPERM, "Operation not permitted.");
declare_err!(ENOENT, "No such file or directory.");
```
The first line expands to the following two lines:

```
#[doc =  "Operation not permitted."]
pub const EPERM: super::Error = super::Error(-(crate::bindings::EPERM as i32));
```

The error code is negated and cast to a i32 to match the type of `super::Error`.
This code declares a constant, `EPERM`, with the respective documentation. The
documentation can be generated with rustdoc, but the documentation String is not
accessible at runtime, because Rust does not have a reflection feature like some
other languages like Python, Java and C# do.

The `$(,)?` designator allows for an optional trailing comma. This means that
both those invocations are valid, the second line having a trailing comma:

```
declare_err!(EPERM, "Operation not permitted.");
declare_err!(ENOENT, "No such file or directory",);
```

Furthermore, the designator `$($doc:expr),+` means that it is possible to pass
several comma separated string arguments to the macro. This will result in one
additional doc line for each additional string passed.

Let's take the following line as an example:

```
declare_err!(EPERM, "Operation not permitted.", "This is usually due to insufficient permissions.");
```

This would be expanded to the following code:

```
#[doc = "Operation not permitted."]
#[doc = "This is usually due to insufficient permissions."]
pub const EPERM: super::Error = super::Error(-(crate::bindings::EPERM as i32));
```

## `from_errno` and back

After declaring the constants, the implementation for the Error struct follows.

```
#[derive(Clone, Copy, PartialEq, Eq)]
pub struct Error(core::ffi::c_int);

impl Error {
    pub(crate) fn from_errno(errno: core::ffi::c_int) -> Error {
        if errno < -(bindings::MAX_ERRNO as i32) || errno >= 0 {
            crate::pr_warn!(
                "attempted to create `Error` with out of range `errno`: {}",
                errno
            );
            return code::EINVAL;
        }
        Error(errno)
    }
}
```

A call to the function `from_errno` returns an Error type in case the integer
`errno` passed to it was in the valid error range, or the Error `EINVAL` otherwise.

Other Rust code uses the `from_errno` function as follows:

```
pub fn read(&mut self, regnum: u16) -> Result<u16> {
    let ret = unsafe { bindings::some_function() };
    if ret < 0 {
        Err(Error::from_errno(ret))
    } else {
        Ok(ret as u16)
    }
}
```
The return value of read is therefore of a `Result` type.

For the transformation back from an Error type to a C integer, the function `to_errno`
is implemented:

```
impl Error {
    /// Returns the kernel error code.
    pub fn to_errno(self) -> core::ffi::c_int {
        self.0
    }
}
```

Since the code line `pub struct Error(core::ffi::c_int);` defines that the struct
has only one element, we can access this element by means of writing `self.0`.

The following code block implements the `From` trait for the `Error` type from
the `AllocError` type.
```
impl From<AllocError> for Error {
    fn from(_: AllocError) -> Error {
        code::ENOMEM
    }
}
```
The `from` function takes an `AllocError` as an argument, but ignores the
argument (indicated by `_` as the argument name), and always returns
`code::ENOMEM`. Similar code block exist to convert a `TryFromIntError` into `EINVAL`, a
`LayoutError` into `ENOMEM`, etc. Because of this implementation, you can
explicitly use the `into()` method to convert an `AllocError` into an `Error`,
or the compiler performs the conversion for you where the context is
unambiguous.

Here is a code block demonstrating how the conversion can be used:
```
let alloc_error = AllocError::new();
let error: Error = alloc_error.into(); // This works because of the `From<AllocError> for Error` implementation
```

The next noteworthy code block implements the From trait for the type
Infallible. However, since the type `Infallible` can not exist because no value
can be an `Infallible`, the function body is a match with no arms, to signal
that "this situation is impossible". This is a Rust idiom used for dealing with
types that are theoretically possible due to generic type parameters, but
practically will never occur because the specific types are not used.

```
impl From<core::convert::Infallible> for Error {
    fn from(e: core::convert::Infallible) -> Error {
        match e {}
    }
}
```

Next, we will look at the implementation of the Result type. In the first code
line the default type for the generic parameter `T` is set. This means if you
use `Result` without specifying a type, it will default to the unit type `()`.

```
pub type Result<T = (), E = Error> = core::result::Result<T, E>;
```

The function `to_result` converts an integer as returned by a C kernel function
to an error if it's negative, and `Ok(())` otherwise.

```
pub fn to_result(err: core::ffi::c_int) -> Result {
    if err < 0 {
        Err(Error::from_errno(err))
    } else {
        Ok(())
    }
}
```

Since `to_result` returns `Result`, it is the equivalent of writing the
following:

```
pub fn to_result(err: core::ffi::c_int) -> Result<(), Error> {
    // ...
}
```

This default Result type is used for functions which you are calling for their
side effects rather than their return value, where the only useful information
is whether an error occurred or not.

An example usage of the `to_result` function is taken from
`rust/kernel/net/phy.rs`. A call to the C kernel function `phy_init_hw` using an
FFI interface is converted to a result type.
```
    /// Initializes the PHY.
    pub fn init_hw(&mut self) -> Result {
        let phydev = self.0.get();
        to_result(unsafe { bindings::phy_init_hw(phydev) })
    }
```

For the other way around, the function `from_result` calls a closure that
returns a `Result`, and converts the Result to a C integer result.

```
pub(crate) fn from_result<T, F>(f: F) -> T
where
    T: From<i16>,
    F: FnOnce() -> Result<T>,
{
    match f() {
        Ok(v) => v,
        Err(e) => T::from(e.to_errno() as i16),
    }
}
```

This can be useful for callback functions. When you want to call a Rust function
with a return type `Result` form inside `extern "C"` block, the callback
function should return an C integer error in order to be processed further by
the C code. Subsequently you see an example of the implementation of such a
callback function.

```
# use kernel::from_result;
# use kernel::bindings;
unsafe extern "C" fn probe_callback(
    pdev: *mut bindings::platform_device,
) -> core::ffi::c_int {
    from_result(|| {
        let ptr = devm_alloc(pdev)?;
        bindings::platform_set_drvdata(pdev, ptr);
        Ok(0)
    })
}
```

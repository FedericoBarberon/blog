+++
title = "Testing Environment-Dependent Code in Rust"
description = "Isolating tests that depend on environment variables using RAII guards and serial_test"
date = 2026-06-24
[taxonomies]
tags = ["rust", "testing"]
+++

## The problem

When we have a function that interacts with external resources, testing that code might be difficult because side-effects could impact on other tests. Environment variables are global variables stored in the process state. One change in an env variable changes the process state, so this mutation persists along the process lifetime.

In Rust, unit tests are compiled into one binary, which means that all tests run in the same process. That is why we need to be careful when testing code that modifies env vars, because those changes may affect other tests and could produce an undesirable behaviour.

On top of that, Rust tests run in parallel by default, so we need to make sure that there is no more than one thread modifying the same env var at the same time.

## Stopping concurrent access to Environment Variables

One way to guarantee that there is only one thread modifying an env var is to simply use one thread! We can achieve that by running `cargo test -- --test-threads=1`. This is the simplest solution to the problem, however, if the number of unit tests in our project grows a lot, the execution time of the tests could be annoying because all tests, including the ones that do not need to be single-threaded, run on a single thread anyway.

Another solution more appropriate to this case is to use the macro `#[serial]` from the [serial_test](https://crates.io/crates/serial_test) crate. This macro allows us to select the tests that we want to run in serial, while maintaining the others intact

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use serial_test::serial;
    
    #[test]
    fn normal_test1() {
        ...
    }
    
    #[test]
    fn normal_test2() {
        ...
    }

    #[test]
    #[serial]
    fn serial_test1() {
        ...
    }
    
    #[test]
    #[serial]
    fn serial_test2() {
        ...
    }
}
```

In this example, we guarantee that `serial_test1` and `serial_test2` run in serial, while (maybe) at the same time `normal_test1` and `normal_test2` run in parallel, so we get the best of both worlds.

## Why `serial_test` alone isn't enough

With `serial_test` we solve the problem of concurrent mutations to env vars, but we still have the problem that those changes persist across the tests. We need to find a way to restore the previous state of the env vars, so that other tests are not contaminated.

We may be tempted to do it with something like this:

```rust
#[test]
#[serial]
fn test() {
    let var = "PATH";
    let previous_state = std::env::var_os(var).unwrap();

    ... // test the function that modifies <var> env variable

    // SAFETY: There are no other threads modifying the env var because all tests that do it have #[serial] macro.
    unsafe {
        std::env::set_var(var, previous_state);
    }
}
```

However, this has a problem. The last statement may _NEVER_ run if the test panics, so this is not a solution at all.

## The RAII guard

Rust implements RAII (Resource Acquisition Is Initialization), which means that once the owner of some data goes out of scope, the `drop()` method of that data is called and the data is freed.

We can take advantage of this by implementing a common design pattern in Rust, the RAII guard, creating an object that holds the original value of the env vars that we are going to change, and implementing the `Drop` trait so that it restores the env vars state when the object goes out of scope (even if the test panics).

```rust
use std::{ffi::OsString, env};

struct EnvGuard {
    key: OsString,
    value: Option<OsString>
}

impl EnvGuard {
    // SAFETY:
    // The caller must guarantee exclusive access to the process environment
    // for the lifetime of this guard.
    unsafe fn capture(key: OsString) -> Self {
        Self {
            value: env::var_os(&key),
            key,
        }
    }
}

impl Drop for EnvGuard {
    fn drop(&mut self) {
        // SAFETY:
        // The caller of `capture` guaranteed exclusive access to the process
        // environment for the lifetime of this guard.
        unsafe {
            match &self.value {
                Some(val) => env::set_var(&self.key, val),
                None => env::remove_var(&self.key)
            }
        }
    }
}
```

Note that `capture` itself performs no unsafe operation — it only calls `env::var_os`, which is safe. It's marked `unsafe` to push the safety contract to the call site, since that contract ("exclusive access to the environment") covers the guard's entire lifetime, not just this one function.

Now we can use it in our tests

```rust
#[test]
#[serial]
fn test() {
    let var = "PATH";
    
    // SAFETY: There are no other threads modifying the env var because all tests that do it have #[serial] macro.
    let _guard = unsafe { EnvGuard::capture(var.into()) };

    ... // test the function that modifies <var> env variable

    // When `_guard` goes out of scope, even if the test panics, the env var is restored to the original value.
}
```

## Result

We end up with a simple and idiomatic solution to a common problem of testing environment variables. It is worth mentioning that this solution is easy to extend to multiple variables, and also it is easy to adapt to other external state besides the environment variables.

---

*If you spot something incorrect in this post, let me know — I'm still learning this too :)*

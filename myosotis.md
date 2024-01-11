# The practical Leak trait and the drop guarantee
<!-- TODO: rename into Forget/Lose/Trace/Reach or and maybe use -able suffix -->

<a title="Forget-me-nots - Sedum Tauno Erik, CC BY-SA 2.5 &lt;https://creativecommons.org/licenses/by-sa/2.5&gt;, via Wikimedia Commons" href="https://commons.wikimedia.org/wiki/File:Myosotis_arvensis_ois.JPG"><img width="512" alt="Myosotis arvensis ois" src="https://upload.wikimedia.org/wikipedia/commons/thumb/e/eb/Myosotis_arvensis_ois.JPG/512px-Myosotis_arvensis_ois.JPG"></a>

## Background

Currently there is a consensus about abscence of the drop guarantee. To be precise, in today's Rust you can forget some value via [`core::mem::forget`](https://doc.rust-lang.org/1.75.0/core/mem/fn.forget.html) or via some other safe contraption like cyclic shared references `Rc/Arc`.

As you may know in the early days of Rust the drop guarantee was intended to exist. Instead of today's [`std::thread::scope`](https://doc.rust-lang.org/1.75.0/std/thread/fn.scope.html) there was [`std::thread::scoped`](https://doc.rust-lang.org/1.0.0/std/thread/fn.scoped.html) which worked in a similar manner, except it used a guard value with a drop implementation to join the spawned thread so that it wouldn't refer to any local stack variable after the parent thread exited the scope and destroyed them, but due to abscense of the drop guarantee it was found to be unsound and was removed from standard library.<sup id="cite_ref-1">[\[1\]](#cite_note-1)</sup> Let's name these two approaches as <dfn id="intro-guarded_closure"> [guarded closure](#term-guarded_closure) </dfn> and <dfn id="intro-guarde_object"> [guard object](#term-guard_object) </dfn>. Also to note C++20 has analogous [`std::jthread`](https://en.cppreference.com/w/cpp/thread/jthread) guard object.

There is also a discussion among Rust theorists about <dfn id="intro-linear_type"> [linear types](#term-linear_type) </dfn> which leads them researching (or maybe revisiting) the possible `Leak` trait. I've noticed some confusion and thus hesitation when people are trying to define what does leaking a value mean. I will try to clarify and define what does leak actually mean.

## Problem

```rust
use std::marker::PhantomData;
use std::thread;

struct JoinGuard<'a> {
    // using unit as a return value for simplicity
    thread: thread::JoinHandle<()>,
    _marker: PhantomData<&'a ()>,
}

impl<'a> JoinGuard<'a> {
    fn spawn<F>(f: F) -> Self
    where
        F: FnOnce() + Send + 'a,
    {
        JoinGuard {
            thread: thread::spawn(f),
            _marker: PhantomData,
        }
    }
}

impl Drop for JoinGuard<'_> {
    fn drop(&mut self) {
        // Ignoring error, not propating, fine in this situation
        let _ = self.thread.join();
    }
}
```

For the guard object mechanism to work TODO
<!-- TODO: thread/jthread, tigerbeetle::Client callback: 'static -->

## Solution

<!-- TODO: Definition, the other way around -->

<!-- TODO: Theorem and it's assumptions, GC analogy -->

## Implementation

<!-- TODO: `unsafe auto trait Leak {}` and it being a default type bound -->
<!-- TODO: `struct PhantomUnleak` -->

## Details and implications

<!-- TODO: Interation with subtyping and variance -->
<!-- TODO: std::mem::forget_static, and reasons to avoid it (types should implement Leak if they have 'static) -->
<!-- no backwards compatible fix for thread::spawn, but it is already not possible because of return type -->
<!-- TODO: std::mem::forget_unchecked -->
<!-- TODO: `MutexGuard<'a, T>: !Leak` bound is not required but can exist. However with `MutexGuard` current API this bound could safely be broken in any scenario i can think of -->
<!-- TODO: the wast majority of types are Leak -->
<!-- TODO: `mpsc::channel<T>() where T: Leak` -->
<!-- TODO: if Vec element panics during drop currently it then forgets this value and moves forward, which should be modified to accomodate `!Leak` types -->

## Forward compatibility

<!-- TODO: Possible switch to the default `!Leak` bound on types for some future edition could not be painful at all? -->

<!-- TODO: Drop but not AsyncDrop, possible generalization as a cleanup code -->

## Possible problems

<!-- TODO: compiler introducing leaks somewhere unknown place because of leak assumption -->

## Conclusion

<!-- TODO: this is very promising -->

## Terminology

<dl>

<dt id="term-linear_type"> <a href="#intro-linear_type" title="Jump up">^</a> Linear type </dt>

<dd>

value of which should be used at least once, generally speaking.

</dd>

<dt id="term-guarded_closure"> <a href="#intro-guarded_closure" title="Jump up">^</a> Guarded closure </dt>

<dd>

A pattern of a safe library API in Rust. It is a mechanism to guarantee library's cleanup code is run after user code (closure) used some special object. It is usually used only in situations when this guarantee is required to achieve API safety, because it is unnecessary unwieldy otherwise.

```rust
// WARNING: Yes I know you can rewrite this more efficiently, it's just a demonstration

fn main() {
    let mut a = 0;
    foo::scope(|foo| {
        for _ in 0..10 {
            a += foo.get_secret();
            // cannot forget(foo) since we only have a reference to it
        }
    });
    println!("a = {a}");
}

// Implementation

mod foo {
    use std::marker::PhantomData;
    use std::panic::{catch_unwind, resume_unwind, AssertUnwindSafe};

    pub struct Foo<'scope, 'env> {
        secret: u32,
        // use lifetimes to avoid the error
        // strange lifetimes to achieve invariance over them
        _scope: PhantomData<&'scope mut &'scope ()>,
        _env: PhantomData<&'env mut &'env ()>,
    }

    impl Foo<'_, '_> {
        pub fn get_secret(&self) -> u32 {
            // There should be much more complex code
            self.secret
        }

        fn cleanup(&self) {
            println!("Foo::cleanup");
        }
    }

    pub fn scope<'env, F, T>(f: F) -> T
    where
        F: for<'scope> FnOnce(&'scope Foo<'scope, 'env>) -> T,
    {
        let foo = Foo {
            secret: 42,
            _scope: PhantomData,
            _env: PhantomData,
        };

        // AssertUnwindSafe is fine because we rethrow the panic
        let res = catch_unwind(AssertUnwindSafe(|| f(&foo)));

        foo.cleanup();

        match res {
            Ok(v) => v,
            Err(payload) => resume_unwind(payload),
        }
    }
}
```

Output:

```
Foo::cleanup
a = 420
```

</dd>

</dl>
<dl>

<dt id="term-guard_object"> <a href="#intro-guarde_object" title="Jump up">^</a> Guard object </dt>

<dd>

TODO

</dd>

</dl>

## References

1. <a href="#cite_ref-1" id="cite_note-1" title="Jump up">^</a> [rust-lang/rust github issue #24292 - std::thread::JoinGuard (and scoped) are unsound because of reference cycles](https://github.com/rust-lang/rust/issues/24292)

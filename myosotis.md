# The drop guarantee
<!-- TODO: rename into Forget/Lose/Trace/Reach or and maybe use -able suffix -->

<a title="Forget-me-nots - Sedum Tauno Erik, CC BY-SA 2.5 &lt;https://creativecommons.org/licenses/by-sa/2.5&gt;, via Wikimedia Commons" href="https://commons.wikimedia.org/wiki/File:Myosotis_arvensis_ois.JPG"><img width="512" alt="Myosotis arvensis ois" src="https://upload.wikimedia.org/wikipedia/commons/thumb/e/eb/Myosotis_arvensis_ois.JPG/512px-Myosotis_arvensis_ois.JPG"></a>

## Background

Currently there is a consensus about abscence of the drop guarantee. To be precise, in today's Rust you can forget some value via [`core::mem::forget`](https://doc.rust-lang.org/1.75.0/core/mem/fn.forget.html) or via some other safe contraption like cyclic shared references `Rc/Arc`.

As you may know in the early days of Rust the drop guarantee was intended to exist. Instead of today's [`std::thread::scope`](https://doc.rust-lang.org/1.75.0/std/thread/fn.scope.html) there was [`std::thread::scoped`](https://doc.rust-lang.org/1.0.0/std/thread/fn.scoped.html) which worked in a similar manner, except it used a guard value with a drop implementation to join the spawned thread so that it wouldn't refer to any local stack variable after the parent thread exited the scope and destroyed them, but due to abscense of the drop guarantee it was found to be unsound and was removed from standard library.<sup id="cite_ref-1">[\[1\]](#cite_note-1)</sup> Let's name these two approaches as <dfn id="intro-guarded_closure"> [guarded closure](#term-guarded_closure) </dfn> and <dfn id="intro-guard_object"> [guard object](#term-guard_object) </dfn>. Also to note C++20 has analogous [`std::jthread`](https://en.cppreference.com/w/cpp/thread/jthread) guard object.

There is also a discussion among Rust theorists about <dfn id="intro-linear_type"> [linear types](#term-linear_type) </dfn> which leads them researching (or maybe revisiting) the possible `Leak` trait. I've noticed some confusion and thus hesitation when people are trying to define what does leaking a value mean. I will try to clarify and define what does leak actually mean.

## Problem

There is a class of problems that we will try to solve. In particular, we return some object from a function or a method that mutably (exclusivelly) borrows one of function arguments. While returned object is alive we could not refer to borrowed value, which can be a useful property to exploit. You can invalidate some invariant of a borrowed type but then you restore it inside of returned object's drop. This is a fine concept until you realize in some circumstances drop is not called, which would in turn mean that the borrowed type invariant invalidation may never cause <dfn id="intro-undefined_behavior"> [undefined behavior](#term-undefined_behavior) </dfn> (UB in short) if left untreated. However, if drop is guaranteed, we could mess with borrowed type invariant, knowing that the cleanup will restore the invariant and make impossible to cause UB after. I found one example of this as once mentioned planned feature [`Vec::drain_range`](https://github.com/rust-lang/rust/issues/24292#issuecomment-93513451).

One other special case would be owned scoped thread. It may be included within class of problems mentioned, but I am not sure. Anyway, in the most trivial case this is the same as once deleted `std::thread::{scoped, JoinGuard}` described above. However, many C APIs may in some sense use this via the <dfn id="intro-callback_registration"> [callback registration](#term-callback_registration) </dfn> pattern, most common for multithreaded client handles. Abscense of a drop guarantee thus implies `'static` lifetime for a callback so that the user wouldn't use invalidated references inside of the callback, if client uses guard object API pattern ([see example](https://docs.rs/tigerbeetle-unofficial-core/latest/tigerbeetle_unofficial_core/struct.Client.html#method.with_callback)).

## Solution

Most importantly in these two cases objects with the drop guarantee would be bounded by lifetime arguments. So to define the drop guarantee:

```
Drop guarantee asserts that bounding lifetime of an object must end only after its drop. Somehow breaking this guarantee can lead to UB.
```

Notice what this implies for `T: 'static` types. Since static lifetime never ends, the drop may never be called. This property does not conflict with described usecases. `JoinGuard<'static, T>` indeed doesn't requre a drop guarantee, since there would be no references what would ever invalidate.

In context of discussion some argue it is possible to implement `core::mem::forget` via threads and an infinite loop.<sup id="cite_ref-2">[\[2\]](#cite_note-2)</sup> That forget implementation won't violate a drop guarantee as defined above, since either you use regular threads which requre `F: 'static` or use scoped threads which would join this never completing thread thus no drop and no lifetime end. My further advice would be in general to think not in terms of time but in terms of semantic lifetimes, mostly because fundamentally there's no general way to know if your program hangs up or completes.<sup id="cite_ref-3">[\[3\]](#cite_note-3)</sup>

To move forward let's determine required conditions for drop guarantee. Drop is only ever run on owned values, so for a drop to run on a value, **the value should preserve transitive ownership of it by a function stack/local values**. If you familiar with [tracing garbage collection](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)#Tracing) this is similair to it, so that the required alive value should be traceable from function stack. Also **the value has to not own itself or be owned by something that would own itself**, at least before the end of its bounding lifetime, otherwise drop would not be called.

## Trivial implementation

One trivial implementation might have already creeped into your mind.

```rust
unsafe auto trait Leak {}
```

This trait would help to forbid `!Leak` values use problematic functionality. First, the `core::mem::forget` will have this bound over its generic type argument. Second, most data structures introducing shared ownership will be limited or disabled for `!Leak` types, things like `Rc`, `Arc`, various channel types having some shared buffer like inside of `std::sync::mpsc` module. That is because reference counted types can be moved into themselves or send your receiver into shared buffer with some value to be leaked (synchronous(?) rendezvous channels seem to not have this issue). However, there is a decision to be made about what parts of API should be restricted and which should not: type contructors, `Rc::clone` or type itself?

Given that `!Leak` implies new restrictions compared to current rust value semantics, by default every type is assumed to be `T: Leak`, kinda like with `Sized`, e.g. implicit `Leak` trait bound on every type and type argument unless specified otherwise (`T: ?Leak`). I pretty sure this feature should not introduce any breaking changes. There could be a way to disable implicit `T: Leak` bounds between editions, although I do not see it as a desirable change, since `!Leak` types would be a small minority.

<!-- TODO: Interation with subtyping and variance -->
<!-- TODO: std::mem::forget_static, and reasons to avoid it (types should implement Leak if they have 'static) -->
<!-- no backwards compatible fix for thread::spawn, but it is already not possible because of return type -->
<!-- TODO: std::mem::forget_unchecked -->
<!-- TODO: `MutexGuard<'a, T>: !Leak` bound is not required but can exist. However with `MutexGuard` current API this bound could safely be broken in any scenario i can think of -->
<!-- TODO: the wast majority of types are Leak -->
<!-- TODO: `mpsc::channel<T>() where T: Leak` -->
<!-- TODO: if Vec element panics during drop currently it then forgets this value and moves forward, which should be modified to accomodate `!Leak` types -->

By analogy with `trait Unpin` and `struct PhantomPinned`, there should probably be `struct PhantomUnleak`.

## Extensions and alternatives

### Ranked Leak trait

### Disowns trait

<!-- TODO: custom auto traits -->

### NeverGives trait

## Forward compatibility

<!-- TODO: Possible switch to the default `!Leak` bound on types for some future edition could not be painful at all? -->

<!-- TODO: Drop but not AsyncDrop, possible generalization as a cleanup code -->

## Possible problems

<!-- TODO: compiler introducing leaks somewhere unknown place because of leak assumption -->

<!-- TODO: -->

## Conclusion

<!-- TODO: this is very promising -->

## Terminology

<dl>

<dt id="term-linear_type"> <a href="#intro-linear_type" title="Jump up">^</a> Linear type </dt>

<dd>

value of which should be used at least once, generally speaking. TODO

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

<dt id="term-guard_object"> <a href="#intro-guard_object" title="Jump up">^</a> Guard object </dt>

<dd>

TODO

</dd>

<dt id="term-callback_registration"> <a href="#intro-callback_registration" title="Jump up">^</a> Callback registration </dt>

<dd>

TODO

</dd>

<dt id="term-undefined_behavior"> <a href="#intro-undefined_behavior" title="Jump up">^</a> Undefined behavior or UB </dt>

<dd>

TODO

</dd>

</dl>

## References

1. <a href="#cite_ref-1" id="cite_note-1" title="Jump up">^</a> [rust-lang/rust github issue #24292 - std::thread::JoinGuard (and scoped) are unsound because of reference cycles](https://github.com/rust-lang/rust/issues/24292)

2. <a href="#cite_ref-2" id="cite_note-2" title="Jump up">^</a> [Yoshua Wuyts - Linear Types One-Pager # Updates](https://blog.yoshuawuyts.com/linear-types-one-pager/#updates)

3. <a href="#cite_ref-3" id="cite_note-3" title="Jump up">^</a> [wikipedia.org - Halting_problem](https://en.wikipedia.org/wiki/Halting_problem)

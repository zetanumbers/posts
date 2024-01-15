# The drop guarantee

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

This is an automatic trait, which would mean that it is implemented for types in a similair manner to `Send`.<sup id="cite_ref-4">[\[4\]](#cite_note-4)</sup> Name `Leak` is a subject for a possible future change. I used it as it came up in many people's thoughts as `Leak`. Since `T: !Leak` types possibly could leak in a practical meaning, it can be renamed into `Forget`. Other variants could be `Lose`, `!Trace` or `!Reach` (last two as in tracing GC), maybe add `-able` suffix?

By analogy with `trait Unpin` and `struct PhantomPinned`, there should probably be `struct PhantomUnleak`.

This trait would help to forbid `!Leak` values use problematic functionality. First, the `core::mem::forget` will have this bound over its generic type argument. Second, most data structures introducing shared ownership will be limited or disabled for `!Leak` types, things like `Rc`, `Arc`, various channel types having some shared buffer like inside of `std::sync::mpsc` module. That is because reference counted types can be moved into themselves or send your receiver into shared buffer with some value to be leaked (synchronous(?) rendezvous channels seem to not have this issue). However, there is a decision to be made about what parts of API should be restricted and which should not: type contructors, `Rc::clone` or type itself?

Given that `!Leak` implies new restrictions compared to current rust value semantics, by default every type is assumed to be `T: Leak`, kinda like with `Sized`, e.g. implicit `Leak` trait bound on every type and type argument unless specified otherwise (`T: ?Leak`). I pretty sure this feature should not introduce any breaking changes. There could be a way to disable implicit `T: Leak` bounds between editions, although I do not see it as a desirable change, since `!Leak` types would be a small minority in my vision.

One thing that we should be aware of in the future would be users' desire of making their types `!Leak` while not actually needing. The appropriate example would be `MutexGuard<'a, T>` being `!Leak`. It is not required, since it is actually safe to forget a value of this type or to never unlock a mutex, but it can exist. In this case, you can safely violate `!Leak` bound, making it useless in practice. Thus unnecessary `!Leak` impls should be avoided. To address users' underlying itch to do this, they should be informed that forgetting or leaking a value is already undesirable and may be considered a logic bug.

Of course there should be an unsafe `core::mem::forget_unchecked` for any value if you really know what you're doing, there are some ways to implement `core::mem::forget` for any type with unsafe code still, for example with `core::ptr::write`. There should also probably be safe `core::mem::forget_static` since you can basically do that using thread with an endless loop. However `!Leak` types should implement `Leak` for static lifetimes themselves to satisfy any function's bounds over types.

```rust
struct JoinGuard<'a, T: 'a> {
    // ...
    _unleak: PhantomUnleak,
}
unsafe impl<T: 'static> Leak for JoinGuard<'static, T> {}
```

One interesting case comes up when thinking about types [contravariant](https://doc.rust-lang.org/reference/subtyping.html) by their generic argument lifetime. Since this lifetime can be extended into `'static` you should implement `Leak` then for any possible lifetime in this position.

```rust
struct ContravariantLifetime<'contravariant, 'covariant> {
    // ...
    _variance: PhantomData<fn(&'contravariant ()) -> &'covariant ()>
}
unsafe impl<'contravariant> Leak for ContravariantLifetime<'contravariant, 'static> {}
```

While implementing `!Leak` types you should also make sure you cannot move a value of this type into itself. In particular `JoinGuard` may be made `!Send` to ensure that user won't send `JoinGuard` into its inner thread, creating a reference to itself, thus escaping from a parent thread while having live references to parent thread local variables.

```rust
unsafe impl<T: 'static> Send for JoinGuard<'static, T> {}
```

There is also a way to forbid `JoinGuard` from moving into its thread if we bound it by a different lifetime which is shorter than input closure's lifetime (see `JoinGuardScoped` in leak-playground [docs](https://zetanumbers.github.io/leak-playground/leak_playground/) and [repo](https://github.com/zetanumbers/leak-playground)).

## Extensions and alternatives

### Ranked Leak trait

### Disowns trait

<!-- TODO: custom auto traits -->

### NeverGives trait

## Forward compatibility

<!-- TODO: Possible switch to the default `!Leak` bound on types for some future edition could not be painful at all? -->

<!-- TODO: Drop but not AsyncDrop, possible generalization as a cleanup code -->

## Possible problems

Some current std library functionality relies upon forgetting values, like `Vec` does it in some cases like panic during element's drop. I'm not sure if anyone relies upon this, so we could use abort instead. Or instead we can add `std::mem::is_leak::<T>() -> bool` to determine if we can forget values or not and then act accordingly.

There could also be some niche compilation case, where compiler assumes every type is `Leak` and purposefully forgets a value.

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

4. <a href="#cite_ref-4" id="cite_note-4" title="Jump up">^</a> [unstable book - auto-traits](https://doc.rust-lang.org/beta/unstable-book/language-features/auto-traits.html)

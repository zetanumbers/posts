# The drop guarantee and linear types formulation

<a title="Forget-me-nots - Sedum Tauno Erik, CC BY-SA 2.5 &lt;https://creativecommons.org/licenses/by-sa/2.5&gt;, via Wikimedia Commons" href="https://commons.wikimedia.org/wiki/File:Myosotis_arvensis_ois.JPG"><img width="512" alt="Myosotis arvensis ois" src="https://upload.wikimedia.org/wikipedia/commons/thumb/e/eb/Myosotis_arvensis_ois.JPG/512px-Myosotis_arvensis_ois.JPG"></a>

## Background

Currently there is a consensus about absence of the drop
guarantee. To be precise, in today's Rust you can forget some value via
[`core::mem::forget`](https://doc.rust-lang.org/1.75.0/core/mem/fn.forget.html)
or via some other safe contraption like cyclic shared references `Rc/Arc`.

As you may know in the early days of Rust the
drop guarantee was intended to exist. Instead of today's
[`std::thread::scope`](https://doc.rust-lang.org/1.75.0/std/thread/fn.scope.html)
there was
[`std::thread::scoped`](https://doc.rust-lang.org/1.0.0/std/thread/fn.scoped.html)
which worked in a similar manner, except it used a guard value
with a drop implementation to join the spawned thread so that it
wouldn't refer to any local stack variable after the parent thread
exited the scope and destroyed them, but due to absence of the drop
guarantee it was found to be unsound and was removed from standard
library.<sup id="cite_ref-1">[\[1\]](#cite_note-1)</sup> Let's
name these two approaches as <dfn id="intro-guarded_closure">
[guarded closure](#term-guarded_closure)
</dfn> and <dfn id="intro-guard_object"> [guard
object](#term-guard_object) </dfn>. Also to note C++20 has analogous
[`std::jthread`](https://en.cppreference.com/w/cpp/thread/jthread)
guard object.

There is also a discussion among Rust theorists about <dfn
id="intro-linear_type"> [linear types](#term-linear_type) </dfn>
which leads them researching (or maybe revisiting) the possible `Leak`
trait. I've noticed some confusion and thus hesitation when people are
trying to define what does leaking a value mean. I will try to clarify
and define what does leak actually mean.

## Problem

There is a class of problems that we will try to solve. In particular,
we return some object from a function or a method that mutably
(exclusively) borrows one of function arguments. While returned object
is alive we could not refer to borrowed value, which can be a useful
property to exploit. You can invalidate some invariant of a borrowed
type but then you restore it inside of returned object's drop. This
is a fine concept until you realize in some circumstances drop is not
called, which would in turn mean that the borrowed type invariant
invalidation may never cause <dfn id="intro-undefined_behavior">
[undefined behavior](#term-undefined_behavior) </dfn> (UB in
short) if left untreated. However, if drop is guaranteed, we could
mess with borrowed type invariant, knowing that the cleanup will
restore the invariant and make impossible to cause UB after. I
found one example of this as once mentioned planned feature
[`Vec::drain_range`](https://github.com/rust-lang/rust/issues/24292#issuecomment-93513451).

One other special case would be owned scoped thread. It may be included
within class of problems mentioned, but I am not sure. Anyway, in the most
trivial case this is the same as once deleted `std::thread::{scoped,
JoinGuard}` described above. However, many C APIs may in some
sense use this via the <dfn id="intro-callback_registration">
[callback registration](#term-callback_registration) </dfn>
pattern, most common for multithreaded client handles. Absence
of a drop guarantee thus implies `'static` lifetime for a
callback so that the user wouldn't use invalidated references
inside of the callback, if client uses guard object API pattern ([see
example](https://docs.rs/tigerbeetle-unofficial-core/latest/tigerbeetle_unofficial_core/struct.Client.html#method.with_callback)).

## Solution

Most importantly in these two cases objects with the drop guarantee
would be bounded by lifetime arguments. So to define the drop guarantee:

```
Drop guarantee asserts that bounding lifetime of an object must end only
after its drop. Somehow breaking this guarantee can lead to UB.
```

Notice what this implies for `T: 'static` types. Since static lifetime
never ends, the drop may never be called. This property does not conflict
with described use cases. `JoinGuard<'static, T>` indeed doesn't requre
a drop guarantee, since there would be no references what would ever
be invalidated.

In the context of discussion around `Leak` trait some argue it is possible
to implement `core::mem::forget` via threads and an infinite loop.<sup
id="cite_ref-2">[\[2\]](#cite_note-2)</sup> That forget implementation
won't violate a drop guarantee as defined above, since either you use
regular threads which require `F: 'static` or use scoped threads which
would join this never completing thread thus no drop and no lifetime
end. **That definition only establishes order between drop and end
of a lifetime, but not existence of a lifetime's end inside of any
execution time.** My further advice would be in general to **think not
in terms of execution time but in terms of semantic lifetimes**, which
role would be to conservatively establish order of events if those ever
exist. Alternatively you will be fundamentally limited by the [halting
problem](https://en.wikipedia.org/wiki/Halting_problem).

On the topic of abort, it shouldn't be considered an end to any lifetime,
since otherwise abort and even spontaious termination of a program like
SIGTERM becomes unsafe.

To move forward let's determine required conditions for drop
guarantee. Rust language already makes sure you could never
use a value which bounding lifetime has ended. Drop is only
ever run on owned values, so for a drop to run on a value,
**the value should preserve transitive ownership of it by
functions' stack/local values**. If you familiar with [tracing garbage
collection](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)#Tracing)
this is similar to it, so that the required alive value should be
traceable from function stack. The value has to not own itself or be
owned by something that would own itself, at least before the end of its
bounding lifetime, otherwise drop would not be called. Last statement
could be simplified, given that **owner of a value transitively must also
satisfy these requirements**, leaving us with just **the value has to not
own itself**. Also reminding you that `'static` values can be moved into
static context like static variables, which lifetime exceedes lifetime
of a program's execution itself, so consider that analogous to calling
`std::process::abort()` before `'static` ends.

## Trivial implementation

One trivial implementation might have already crept into your mind.

```rust
unsafe auto trait Leak {}
```

This is an automatic trait, which would mean that it is
implemented for types in a similar manner to `Send`.<sup
id="cite_ref-3">[\[3\]](#cite_note-3)</sup> Name `Leak` is a subject
for a possible future change. I used it as it came up in many people's
thoughts as `Leak`. Since `T: !Leak` types possibly could leak in a
practical meaning, it can be renamed into `Forget`. Other variants could
be `Lose`, `!Trace` or `!Reach` (last two as in tracing GC), maybe add
`-able` suffix?

This trait would help to forbid `!Leak` values from using problematic
functionality. First, the `core::mem::forget` will have this bound over
its generic type argument. Second, most data structures introducing shared
ownership will be limited or disabled for `!Leak` types, things like `Rc`,
`Arc`, various channel types having some shared buffer like inside of
`std::sync::mpsc` module. That is because reference counted types can be
moved into themselves or send your receiver into shared buffer with some
value to be leaked (synchronous(?) rendezvous channels seem to not have
this issue). However, there is a decision to be made about what parts
of API should be restricted to `T: Leak` and which should not: type
constructors, `Rc::clone` or type itself? It is safe to use `Rc` with
`T: ?Leak` if we make sure we won't leak any `Rc` value, so I would say
`Rc::new_unchecked` for `?Leak` types is appropriate.

Given that `!Leak` implies new restrictions compared to current rust
value semantics, by default every type is assumed to be `T: Leak`, kinda
like with `Sized`, e.g. implicit `Leak` trait bound on every type and
type argument unless specified otherwise (`T: ?Leak`). I pretty sure this
feature should not introduce any breaking changes. This means working with
new `!Leak` types is opt-in, kinda like library APIs may consider adding
`?Sized` support after release. There could be a way to disable implicit
`T: Leak` bounds between editions, although I do not see it as a desirable
change, since `!Leak` types would be a small minority in my vision.

To make `!Leak` struct you would need to use new `Unleak` wrapper type:


```rust
// Some usage of Unleak, probably move JoinGuard there

// Unleak definition

#[repr(transparent)]
#[derive(Default, Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Unleak<T>(pub T, PhantomUnleak);

impl<T> Unleak<T> {
    pub const fn new(v: T) -> Self {
        Unleak(v, PhantomUnleak)
    }
}

unsafe impl<T: 'static> Leak for Unleak<T> {}

#[derive(Default, Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
struct PhantomUnleak;

impl !Leak for PhantomUnleak {}
```

Will `PhantomUnleak` ever be public/stable is to be determined.

One thing that we should be aware of in the future would be users'
desire of making their types `!Leak` while not actually needing it. The
appropriate example would be `MutexGuard<'a, T>` being `!Leak`. It is
not required, since it is actually safe to forget a value of this type or
to never unlock a mutex, but it can exist. In this case, you can safely
violate `!Leak` bound, making it useless in practice. Thus unnecessary
`!Leak` impls should be avoided. To address users' underlying itch to
do this, they should be informed that **forgetting or leaking a value
is already undesirable and can be considered a logic bug**.

Of course there should be an unsafe `core::mem::forget_unchecked` for
any value if you really know what you're doing, there are some ways
to implement `core::mem::forget` for any type with unsafe code still,
for example with `core::ptr::write`. There should also probably be safe
`core::mem::forget_static` since you can basically do that using thread
with an endless loop. However `?Leak` types implement `Leak` for static
lifetimes transitively from `PhantomUnleak` to satisfy any function's
bounds over types.

```rust
struct JoinGuard<'a, T: 'a> {
    // ...
    _marker: PhantomData<fn() -> T>, // TODO: figure out
    _unleak: Unleak<PhantomData<&'a ()>>,
}
```

One interesting case comes up when thinking about types
[contravariant](https://doc.rust-lang.org/reference/subtyping.html) by
their generic lifetime argument. Since this lifetime can be extended into
`'static` you should implement `Leak` then for any possible lifetime in
this position.

```rust
struct ContravariantLifetime<'contravariant, 'covariant> {
    // ...
    _variance: PhantomData<fn(&'contravariant ())>,
    _unleak: Unleak<PhantomData<&'covariant ()>>,
}
```

While implementing `!Leak` types you should also make sure you cannot move
a value of this type into itself. In particular `JoinGuard` may be made
`!Send` to ensure that user won't send `JoinGuard` into its inner thread,
creating a reference to itself, thus escaping from a parent thread while
having live references to parent thread local variables.

```rust
struct JoinGuard<'a, T: 'a> {
    // ...
    _marker: PhantomData<fn() -> T>, // TODO: figure out
    _unleak: Unleak<PhantomData<&'a ()>>,
    _unsend: PhantomData<*mut ()>,
}

unsafe impl<'a, T> Send for JoinGuard<'a, T> where Self: Leak {}
unsafe impl<'a, T> Sync for JoinGuard<'a, T> {}
```

There is also a way to forbid `JoinGuard` from moving into its thread
if we bound it by a different lifetime which is shorter than input
closure's lifetime. See prototyped `JoinGuardScoped` in leak-playground
[docs](https://zetanumbers.github.io/leak-playground/leak_playground/)
and [repo](https://github.com/zetanumbers/leak-playground). There's no
proposed `Leak` trait, so conditions are enforced manually. It works
and does not when needed, but I'm not sure without an actual proof.

One other decision to be made about handling a panic during a drop of a
`!Leak` value. Panic in this case could indicate a failure of a drop
implementation, thus state of borrowed values might remain invalid
acording to the type invariant. It should be safe to make some type
`!Leak` with dummy types, so we cannot put a requirement prohibiting
panic within drop for `!Leak` types. This leaves us with abort on a
drop panic, which would propagate to any type containing `!Leak` value
too. Still, interactions with generic `T: ?Leak` types remain unclear
as to dynamically differentiate between `Leak` and `!Leak` types or not.
On the other side any safety invariant violation could only be caused by
unsafe code, so we can retain old behaviour of drop and define the drop
panic of `!Leak` types to being a valid cleanup, assuming inner fields
are dropped too after that or abort happened, so **you would have to be
careful with panics too**. In this case you can manually abort on panic.

Consider one other example from [leak-playground](https://zetanumbers.github.io/leak-playground/leak_playground/):

<div id="internal_unleak_future">

```rust
fn _internal_unleak_future() -> impl std::future::Future<Output = ()> + Leak {
    async {
        let num = std::hint::black_box(0);
        let bor = Unleak::new(&num);
        let () = std::future::pending().await;
        assert_eq!(*bor.0, 0);
    }
}
```

</div>

During the execution of a future, local variables have non-static
lifetimes, however after future yields these lifetimes become static
unless they refer to something outside of it. This is an example of sound
and safe lifetime extension thus making the whole future `Leak`. However,
if when we use `JoinGuard` it becomes a little bit trickier:

```rust
fn _internal_join_guard_future() -> impl std::future::Future<Output = ()> + Leak {
    async {
        let local = 42;
        let thrd = JoinGuard::spawn({
            let local = &local;
            move || {
                let _inner_local = local;
            }
        });
        let () = std::future::pending().await;
        drop(thrd);
    }
}
```

Code above may lead to UB if we `forget` this future, meaning the memory
holding this future is deallocated without dropping this future first. But
remember that self-referencial (`!Unpin`) future after it starts is
pinned forever, which means it is guaranteed there is/should be no way
to forget and deallocate underlying value in safe code (see pin's [drop
guarantee](https://doc.rust-lang.org/std/pin/index.html#drop-guarantee)).
However outside of rust-lang project some people would not follow this
rule because they don't know about it or maybe discard it puposefuly
(the *Rust police* is coming for you). Maybe in the future it would be
possible to somehow relax this rule in some cases, but it would be a
different problem.

## Extensions and alternatives

*This section is optional as it contains unpolished concepts, which is
not essential understanding the overall design of proposed feature.*

### Disowns trait

<!-- TODO: custom SafeForRC traits -->

### NeverGives trait

### Ranked Leak trait

## Forward compatibility

<!-- TODO: Drop but not AsyncDrop, possible generalization as a cleanup code -->
It could be the case that `JoinGuard` logic can be extended to analogous
`AwaitGuard` representing async tasks.

## Possible problems

Some current std library functionality relies upon forgetting values,
like `Vec` does it in some cases like panic during element's drop. I'm
not sure if anyone relies upon this, so we could use abort instead. Or
instead we can add `std::mem::is_leak::<T>() -> bool` to determine if
we can forget values or not and then act accordingly.

Currently [internally unleak futures](#internal_unleak_future)
examples emit errors where shouldn't or should emit different errors,
so I guess some compiler hacking is required. There could also be some
niche compilation case, where compiler assumes every type is `Leak`
and purposefully forgets a value.

<!--

## Discarded ideas

*This section may confuse readers so you might not want to skip it. It
is intended to be refered to when discussion touches these thought of
before but then discarded ideas.*

### Leak<'a>

I cannot describe this

-->

## Conclusion

<!-- TODO: this is very promising -->
<!-- TODO: one step towards linear types -->

## Terminology

<dl>

<dt id="term-linear_type"> <a href="#intro-linear_type" title="Jump up">^</a> Linear type </dt>

<dd>

value of which should be used at least once, generally speaking. TODO

</dd>

<dt id="term-guarded_closure"> <a href="#intro-guarded_closure" title="Jump up">^</a> Guarded closure </dt>

<dd>

A pattern of a safe library API in Rust. It is a mechanism to guarantee
library's cleanup code is run after user code (closure) used some special
object. It is usually used only in situations when this guarantee is
required to achieve API safety, because it is unnecessary unwieldy
otherwise.

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

3. <a href="#cite_ref-3" id="cite_note-3" title="Jump up">^</a> [unstable book - auto-traits](https://doc.rust-lang.org/beta/unstable-book/language-features/auto-traits.html)

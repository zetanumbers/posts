# The practical Leak trait and safe drop guarantee

<a title="Myosotis arvensis ois - Sedum Tauno Erik, CC BY-SA 2.5 &lt;https://creativecommons.org/licenses/by-sa/2.5&gt;, via Wikimedia Commons" href="https://commons.wikimedia.org/wiki/File:Myosotis_arvensis_ois.JPG"><img width="512" alt="Myosotis arvensis ois" src="https://upload.wikimedia.org/wikipedia/commons/thumb/e/eb/Myosotis_arvensis_ois.JPG/512px-Myosotis_arvensis_ois.JPG"></a>

## Background

Currently there is a consensus about abscence of the drop guarantee. To be precise, in today's Rust you can forget some value via [`core::mem::forget`](https://doc.rust-lang.org/1.75.0/core/mem/fn.forget.html) or via some other safe contraption like cyclic shared references `Rc/Arc`.

As you may know in the early days of Rust the drop guarantee was intended to exist. Instead of today's [`std::thread::scope`](https://doc.rust-lang.org/1.75.0/std/thread/fn.scope.html) there was [`std::thread::scoped`](https://doc.rust-lang.org/1.0.0/std/thread/fn.scoped.html) which worked in a similar manner, except it used a guard value with a drop implementation to join the spawned thread so that it wouldn't refer to any local stack variable after the parent thread exited the scope and destroyed them, but due to abscense of the drop guarantee it was found to be unsound and was removed from standard library.<sup id="cite_ref-1">[\[1\]](#cite_note-1)</sup> Let's name these two approaches as <dfn> [guarded closure](#term-guarded_closure) </dfn> and <dfn> guardian value </dfn>. C++ has no problem with analogous [`std::jthread`](https://en.cppreference.com/w/cpp/thread/jthread) since C++ .

There is also a discussion among Rust theorists about <dfn title="value of which should be used at least once, generally speaking"> linear types </dfn> which leads them researching (or maybe revisiting) the possible `Leak` trait. I've noticed some confusion and thus hesitation when people are trying to define what does leaking a value mean. I will try to clarify and define what does leak actually mean.

## The problem



<!-- TODO: MutexGuard: Leak -->

## Terminology

<dl>

<dt id="term-guarded_closure"> Guarded closure </dt>
<dd>
A pattern of safe library API in Rust to ensure to run cleanup code after user code used some special type.

```rust
// TODO: some code
```

</dd>

</dl>

## References

1. <a href="#cite_ref-1" id="cite_note-1" title="Jump up">^</a> [rust-lang/rust github issue #24292 - std::thread::JoinGuard (and scoped) are unsound because of reference cycles](https://github.com/rust-lang/rust/issues/24292)

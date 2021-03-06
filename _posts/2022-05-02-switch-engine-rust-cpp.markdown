---
layout: post
title: "Why I switched my game engine from C++ to Rust"
date: 2022-05-02 17:40:00 +0200
categories: 
    - rust 
    - c++ 
    - game engine
comments: true
---

## Introduction
I started programming in C++ six years ago, starting with a few toy projects on Unreal Engine 4.
I had a small experience before, having programmed in C#, Java, Python but, C++ really hooked me up for several reasons:
- The art of pointers
- The power of template metaprogramming
- The **really** big ecosystem of libraries, tools, etc
- And flexing about me doing C++! (but on Windows, no archlinux vim setup sorry!)

Also, I really wanted to get into game engine development, I've made some toy projects with LWJGL (OpenGL/Vulkan bindings for Java) and SharpDX (D3D bindings for C#) on Java/C# beforehand so to me it was natural switching to C++ for that job.

With C++, I made about ~3/4 toy game engines that died off pretty quickly, I have used OpenGL, Direct3D 11, Direct3D 12 and Vulkan but I have the most experience with Direct3D11 and Vulkan (though I have now quite more experience with Vulkan).

## Why Rust?

So, why Rust?

When programming my toy game engine v102 during the 2020 lockdown, I discovered "Rust". A recent programming language, made by the folks at Mozilla, low-level, compiled. Nice.
I saw alot of rustaceans praising the language for its safety, how C and C++ was bad and so unsafe blabla. I admit, at first I didn't quite like Rust just because of a small amount of Rust fanatics, a classic "the most unexperienced people speak the louder because they need to show to people that they do Rust" (it's the same case for C++, C and a lot of low-level languages!). It is only this year that I started doing Rust, and.. I love it!

I loved it for multiple reasons, the modern syntax, Cargo, but there's one thing that really really cool with Rust: safety.

### Memory safety

The majority of C++ developers here know how manipulating pointers and memory can result in a lot of bugs and crashes. The most obvious example is not free-ing memory allocated with `new` or `malloc()`: boom, memory leak! Well, this problem really shouldn't exist anymore in modern C++ as we got now smart pointers: `std::unique_ptr`/`std::shared_ptr`/`std::weak_ptr`, free-ing your memory when leaving theirs scope. That's cool, but it doesn't resolve all memory safety problems in C++: for example it cannot null out your pointers pointing to an object that was deleted, like a garbage collector would do, you'll get a **dangling pointer**<sup>1</sup>.

### Thread safety
Thread safety can be really hard, data races<sup>2</sup> are one of the worst type of bugs you can encounter, they can be really difficult to debug. In native C++, there is also no mechanism that protect you from them. Of course, the language standard library have `std::mutex`, `std::atomic`, ... that can help you avoiding them, but you can still make errors and get data races if you forgot one or if you have a bad architecture.

### How Rust try to protect you

In Rust, the compiler try to prevent most of the major memory and thread safety problems. How? With introducing strict rules in the language itself.

#### References
Like C++, Rust does have references (when you reference something you "borrow" it in Rust), being non-nullable pointers to a variable. But, unlike C++ they follow two rules:
- You can't create a mutable reference while there is non-mutable references alive.
- You can't create mulitple mutable references to the same object, only one should exist at anytime.

So, for example, if we try to create a mutable reference to `val`, and in the same scope/lifetime create a const one:
```rust
let mut val = 20;

let mut ref1 = &mut val;
let ref2 = &val;

println!("{}, {}", ref1, ref2);
```

We'll get this:
```
error[E0502]: cannot borrow `val` as immutable because it is also borrowed as mutable
 --> main.rs:6:17
  |
5 |     let mut ref1 = &mut val;
  |                         --- mutable borrow occurs here
6 |     let ref2 = &val;
  |                 ^^^ immutable borrow occurs here
...
9 | }
  | - mutable borrow ends here
```

Creating two mutable references will also not compile:
```rust
fn main()
{
    let mut val = 20;

    let mut ref1 = &mut val;
    let mut ref2 = &mut val;

    println!("{}, {}", ref1, ref2);
}
```
```
error[E0499]: cannot borrow `val` as mutable more than once at a time
 --> main.rs:6:25
  |
5 |     let mut ref1 = &mut val;
  |                         --- first mutable borrow occurs here
6 |     let mut ref2 = &mut val;
  |                         ^^^ second mutable borrow occurs here
...
9 | }
```

But having two non-mutable references will compile normally:
```rust
fn main()
{
    let mut val = 20;

    let ref1 = &val;
    let ref2 = &val;

    println!("{}, {}", ref1, ref2);
}
```
```
20, 20
```

#### Lifetimes
The lifetime mechanism is what is used to power the safety provided by references in Rust. They're made so you can never in safe Rust have a dangling reference<sup>3</sup> or a data race. Stealing the example from [The Rust Progamming Language](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html), we can represent lifetimes using named (with a `'` prefix) scopes:
```rust
{
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {}", r); //          |
}      
```

We can see that we have the lifetime `'a` that basically represents the whole lifetime of `r`. 
In a inner scope we create `x` having its lifetime `'b`, and we borrow `x` in `r` (`r` will become a reference). Now, compile that and...
```
error[E0597]: `x` does not live long enough
  --> main.rs:8:9
   |
7  |             r = &x;     
   |                  - borrow occurs here
8  |         }                   
   |         ^ `x` dropped here while still borrowed
...
11 |     }   
   |     - borrowed value needs to live until here
```
The compiler prevented us from creating a dangling reference!

#### Fearless Concurrency

With Rust, the goal is to reach [fearless concurrency](https://doc.rust-lang.org/book/ch16-00-concurrency.html).
In order to achieve that, the Rust compiler will detect cases when you'll try to send to another thread a type that cannot be sended safely. This is enabled using language markers: the `Sync` trait and the `Send` trait.

`Send` types can be moved to another thread safely, for example the type `Arc` in Rust (a atomic shared-pointer) is `Send`, but `Rc` (a non-atomic shared-pointer) is not.

So if you try for example to spawn a thread and moving a `Rc` (which is `!Send`) inside:
```rust
use std::rc::Rc;
use std::thread;

fn main() 
{
    let mut a = Rc::new(20);
    
    thread::spawn(move || {
        println!("{}", a);
    });
}
```
```
error[E0277]: `Rc<i32>` cannot be sent between threads safely
   --> src/main.rs:8:5
    |
8   |       thread::spawn(move || {
    |  _____^^^^^^^^^^^^^_-
    | |     |
    | |     `Rc<i32>` cannot be sent between threads safely
9   | |         println!("{}", a);
10  | |     });
    | |_____- within this `[closure@src/main.rs:8:19: 10:6]`
    |
    = help: within `[closure@src/main.rs:8:19: 10:6]`, the trait `Send` is not implemented for `Rc<i32>`
    = note: required because it appears within the type `[closure@src/main.rs:8:19: 10:6]`
note: required by a bound in `spawn`
```
Nice! The compiler tells us that `Rc<i32>` cannot be send between threads safely, and tell us why and why it is required:
- the trait `Send` is not implemented for `Rc<i32>` => Rc<i32> is `!Send`
- note: required by a bound in `spawn` => Tells us that `spawn` requires that the closure is `Send`.

The `Sync` trait tells Rust that you can have references to your type in multiple threads without problems. This is typically enabled by using a `Mutex`, `RwLock`, etc.

So, we saw that Rust implements by default a lot of safety about memory and threads, using rules & semantics implemented in the language directly we can greatly increase the safety of our code without much headache (a bit though when learning Rust haha). You may achieve the same sort of protection in C++ using external tools (like sanitizers), but having that inside the language directly is a great advantage.

#### Unsafe code in Rust
I'm not going to talk much about unsafe code in Rust, but yes, you can write unsafe code in Rust. This is required for implementing safe types or for FFI:
[Unsafe Rust](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html)

### Cargo

Coming from C++, Cargo is a gift from god, really. I really forgot what was a great package manager thanks to C++.
It is really easy to setup, configure and the amount of crates there already are is really cool! Bye CMake + vcpkg!

## State of game engine development in Rust

Now, let's talk about game engine development.

As we saw before, Rust have some great advantages compared to C++, and for many (including me) it can be a serious competitor to C++ in the gamedev world.
As game and game engines because more and more complex and multithreaded, the need for more safety is increasing, even more for thread safety where it can really get complicated. 

Browsing [crates.io](https://crates.io/) shows that there is quite a lot of crates dedicated to gamedev and game engine dev. My game engine has a primary Vulkan backend, so naturally I looked for Vulkan crates and found two interesting ones:
- `vulkano`: a high-level safe wrapper around Vulkan, great if you don't want to handle all the most annoying things you need for Vulkan like managing the memory
- `ash`: unsafe bindings for Vulkan. That's what I choose.

Using `ash` was really simple, it even provides some helper things like builder types so you can easily create the big Vulkan structures:
```rust
let depth_stencil_state = vk::PipelineDepthStencilStateCreateInfo::builder()
        .depth_test_enable(info.depth_stencil_state.enable_depth_test)
        .depth_write_enable(info.depth_stencil_state.enable_depth_write)
        .depth_compare_op(convert_compare_op(info.depth_stencil_state.depth_compare_op))
        .depth_bounds_test_enable(info.depth_stencil_state.enable_depth_bounds_test)
        .stencil_test_enable(info.depth_stencil_state.enable_stencil_test)
        .front(convert_stencil_op_state(info.depth_stencil_state.front_face))
        .back(convert_stencil_op_state(info.depth_stencil_state.back_face))
        .min_depth_bounds(info.depth_stencil_state.min_depth_bounds)
        .max_depth_bounds(info.depth_stencil_state.max_depth_bounds)
```

For window management, you have crates for SDL2 and GLFW, the most used libraries for that.
I personally used the `winapi` crate to interface with the Windows API as in my C++ game engine I don't use any library because I wanted to experiment with managing that myself (and for now it went great, but I still haven't ported my engine to X11/Wayland).

With those two crates (and some other misc things but not related to game engine developement whatsoever) I started re-re-writing my engine but this time in Rust! And it went... really great! 

Understanding lifetimes, borrowing took me about one week, it was quite hard at the beginning, because in my Vulkan backend in my C++ engine, each device resource has a reference to the device, which is well logical, but building something like that in Rust is hard, because you need to introduce explicit lifetimes when you store references in Rust, so the compiler know that the reference will not outlive what it points to, but.. my device was created and stored in a `Box` (`std::unique_ptr` equivalent) and it is the device that created the resource and they are also stored inside a `Box`, the compiler complained that the reference may outlive the device, and well it was quite true, I could drop the device and boom, crash. So I accepted that this was a design flaw, but I really didn't have other solutions, so... I wrote unsafe Rust! (just raw pointers). It works, I think now after a month of developing this Rust engine I should've went with shared pointers, but whatever it works :p

## Conclusion

I really like Rust, it seriously is refreshing after doing many years of C++, and I really think that in the coming years/decades Rust will be more and more used in the game industry as its safety features are really a big plus. As more and more game engines use multiple threads, I think investing your resources in using Rust may be really interesting. 

I think I'll doing some updates soon about my game engine, maybe I will abandon Rust and go back to C++, who know?

I hope you liked this post, it is my first one so it may be poorly written, and as english is not my first language it can complicate comprehension, but I hope it has been clear!

---

1. A pointer pointing to an invalid memory location (that was freed for example).
2. Happens when a thread try to write to a shared resource while another thread read from it.
3. Same as a dangling pointer.

<div id="disqus_thread"></div>
<script>
    var disqus_config = function () {
    this.page.url = "https://zino2201.github.io{{ page.url }}";
    this.page.identifier = "{{ page.id }}";
    };
    
    (function() {
    var d = document, s = d.createElement('script');
    s.src = 'https://zino2201.disqus.com/embed.js';
    s.setAttribute('data-timestamp', +new Date());
    (d.head || d.body).appendChild(s);
    })();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
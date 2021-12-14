---
title: Rust入门
description: Get start with Rust
date: 2021-12-14
slug: rust/startup
image: img/2021/12/LittleBirds.jpg
categories:
  - rust
---

Learned from offical [book](https://doc.rust-lang.org/book)

## Rust Key Concepts

1. module & crate
2. ownership, clone, move
3. borrow(reference), `'static`, lifecycle
4. struct, trait, impl for, as, `dyn Any`
5. macro
6. closure
7. smart pointers (like c++)

### Module & Crate

From large to small which is package(or cargo project), crate(module), your code.

```rust
rustlings  // project
├── src // source
    ├── library // mod or crate
    │   ├── book.rs
    │   └── user.rs
    ├── main.rs // main
    └── lib.rs // library (declaration and use)
```

```rust
pub struct Button;

pub mod a {
    // use super to call parent level mod
    use super::Button;

    pub struct AdButton {
        pub b: Button,
    }
}

pub mod b {
    // use crate to call same level mod
    use crate::a::AdButton;

    // use self to call current mod
    use self::c::CdButton;

    fn bd() {
        let _a = AdButton { b: Button };
        let _c = CdButton;
    }

    pub mod c {
        pub struct CdButton;
    }
}
```

### Ownership

1. For passing a clone, the object shall impl trait `Copy`
2. Integers, bool, floats, `&str`, tuple are compatible with `Copy` trait
3. `Copy` is shallow clone, `Clone` is deep clone

```rust
fn takes_ownership(s: String) {
    // do something
}

fn gives_ownership() -> String {
    let a = String::from("hello");
    a
}

fn main() {
    let s = String::from("ok");
    takes_ownership(s);

    // if we call println! after takes_ownership, we would encounter a compile error
    println!("{}", s);

    let s1 = gives_ownership();
    println!("it's ok to println! s1: {}", s1);
}
```

### Borrow Move

Only one mut borrow or serval immut borrow at one time.

```rust
mod bm {
    fn borrow(s: &String) {
        println!("{}", s);
    }

    fn borrow_mut(s: &mut String) {
        s.push_str(" zzz");
        println!("{}", s)
    }

    fn main() {
        let s = String::from("123");
        borrow(&s);
        borrow(&s);

        let mut z = String::from("zzz");
        borrow_mut(&mut z);

        // compile error under below
        borrow(&z);
    }
}
```

### Struct Trait

- `'static` lives with the application
- `'a` indicate a specific scope

```rust
mod st {
    trait Printer {
        fn print(&self);
    }

    struct Console;

    impl Printer for Console {
        fn print(&self) {
            println!("I am a console")
        }
    }

    fn call_printer(p: &'static dyn Printer) {
        p.print()
    }

    // a & b are in same scope or a's scope include b's
    fn lifecycle<'a>(a: &'a str, b: &'a str) {
        println!("{}{}", a, b)
    }
}
```

### Macro

meta-programming in rust, I don't know much yet.

```rust
macro_rules! success {
    ($fmt:literal, $ex:expr) => {{
        use console::{style, Emoji};
        use std::env;
        let formatstr = format!($fmt, $ex);
        if env::var("NO_EMOJI").is_ok() {
            println!("{} {}", style("✓").green(), style(formatstr).green());
        } else {
            println!(
                "{} {}",
                style(Emoji("✅", "✓")).green(),
                style(formatstr).green()
            );
        }
    }};
}

```

### Closure

`Fn` & `FnOnce`

```rust
mod cl {
    use std::thread;

    fn a() {
        let _print = |x: Box<dyn Display>| {
            println!("{}", x);
        };

        let _print_move = move |x: String| {
            println!(x);
        };

        let s = String::from("oops");
        thread::spawn(move || {
            println!(s);
        });
    }
}
```

### Smart Pointers

trait `Deref` & `Drop` for `*a` & graceful recycle

- `Box<T>` for allocating values on the heap
- `Rc<T>`, a reference counting type that enables multiple ownership
- `Ref<T>` and `RefMut<T>`, accessed through `RefCell<T>`, a type that enforces the borrowing rules at runtime instead of compile time

```rust

```

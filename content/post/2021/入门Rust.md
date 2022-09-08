---
title: 入门 Rust
description: 开始学习 Rust
date: 2021-12-15
slug: rust-first-sight
image: img/2021/12/LittleBirds.jpg
categories: [Learning]
tags: [Learning, Rust]
---

Learned from [official book](https://doc.rust-lang.org/book). One simple advice, **Check the source code**, there are a lot of usage examples and explanation.

## Key Concepts

1. Module & Crate
2. Ownership: clone | move
3. Borrow (reference) & lifecycle (`'static`, `'a`)
4. Struct, trait, impl for, as, `dyn Any`
5. Macro (meta programming)
6. Closure
7. Smart pointers

### Module & Crate

package (cargo project) -> crate (module) -> your code.

```rust
rustlings  // project
├── src // source dir
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

1. For passing a clone, the callee shall impl trait `Copy`
2. Ints, bool, floats, `&str`, tuple are compatible with `Copy` trait
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

### Borrow or Move

Only one mutable borrow or multiple immutable borrows at one time.

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

        // compile error under below, extra immutable borrow
        borrow(&z);
    }
}
```

### Struct & Trait

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

meta programming in rust, I didn't know much yet, here is an example.

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

`Fn` & `FnMut` & `FnOnce` as Function pointers

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
        let _pp = |f: dyn Fn<()>| {
            f();
        };
    }

    fn do_twice<F>(mut func: F)
        where F: FnMut<()> {
        func();
    }
}
```

### Smart Pointers

trait `Deref` & `Drop` for dereference (`*a`) and graceful recycle

- `Box<T>` for allocating values on the heap
- `Rc<T>`, a reference counting type that enables multiple ownership
- `Ref<T>` and `RefMut<T>`, accessed through `RefCell<T>`, a type that enforces the borrowing rules at runtime instead of compile time

```rust
mod sp {
    use std::cell::RefCell;
    use std::rc::{Rc, Weak};

    fn t_box() {
        // move value from stack to heap
        let val: u8 = 5;
        let _b: Box<u8> = Box::new(val);

        // move value from heap to stack by dereference
        let boxed: Box<u8> = Box::new(5);
        let _val: u8 = *boxed;
    }

    struct Owner {
        name: String,
        gadgets: RefCell<Vec<Weak<Gadget>>>,
    }

    struct Gadget {
        id: i32,
        owner: Rc<Owner>,
    }

    fn t_rc() {
        let gadget_owner: Rc<Owner> = Rc::new(
            Owner {
                name: "Gadget Man".to_string(),
                gadgets: RefCell::new(vec![]),
            }
        );

        let g1 = Rc::new(
            Gadget {
                id: 1,
                owner: Rc::clone(&gadget_owner),
            }
        );
        let g2 = Rc::new(
            Gadget {
                id: 2,
                owner: gadget_owner.clone(),
            }
        );

        {
            let mut gadgets = gadget_owner.gadgets.borrow_mut();
            gadgets.push(Rc::downgrade(&g1));
            gadgets.push(Rc::downgrade(&g2));
        }

        for gadget_weak in gadget_owner.gadgets.borrow().iter() {
            let g = gadget_weak.upgrade().unwrap();
            println!("Gadget {} owned by {}", g.id, g.owner.name);
        }
    }
}
```

## At Last

Hopefully, there are many rust projects on github, I shall learn them along with practices.

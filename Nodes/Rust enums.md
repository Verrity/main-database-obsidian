---




---
---
* [[Rust типы|Назад]]
---

```rust unfold
#![warn(clippy::all, clippy::pedantic)]

#[derive(Debug)]
enum Status {
    One(String),
    Two,
    Three{alpha: u8},
    Four(u16,u16,u16),
}

impl Status {
    fn print_value(&self) {
        match self {
            Self::One(s) => println!("{s}"),
            Self::Two => println!("--"),
            Self::Three { alpha } => println!("{alpha}"),
            Self::Four(a,b,c) => println!("{a} {b} {c}"),
        }
    }
}

fn main() {

    let status1 = Status::Two;
    let status2 = Status::One(String::from("dfjklsdfjdkl"));
    let status3 = Status::Three{alpha: 3};
    let status4 = Status::Four(100, 200, 300);


    println!("status1 (debug): {:?}", status2);
    println!("status2 (debug): {status2:?}");
    status3.print_value();
    println!("status4 (debug): {status4:?}");
}
```
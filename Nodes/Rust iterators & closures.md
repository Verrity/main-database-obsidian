---




---
---
* [[Rust|Назад]]
---

#### Closure
По умолчанию делает заимствование захваченной переменной!!!
Ничего не делает до вызова.
`{}` скобки можно не ставить, если closure занимает 1 строку

```rust
fn plus_(param: u8) -> u8 {
    42 + param
}

fn main() {
    let some_param = 10;
    let value: Option<u8> = None;

    // Closure
    let res: u8 = value.unwrap_or_else(|| -> u8 {
        plus_(some_param) // Захватывает параметры
    });

    // Можно хранить в переменной
    let closure = || -> u8 { plus_(some_param) };
    // closure() - выполнить
    // closure - передать, чтобы выполнилось внутри
    let res: u8 = value.unwrap_or_else(closure);

    let mut b = 16;
    let mut demo = |a| {
        println!("{a}");
        b += a;
    };

    demo(10);
    println!("{b}");

    let mut v = vec![1,2,3];
    let mut process = || v.push(3);

    process();

    println!("{:?}", v);
    
	// Чтобы переместить значение нужно сделать move!!
	let mut process2 = move || v.push(3);
}
```

#### Итераторы
Итератор - это объект, который будет итерироваться по коллекции.
При создании он ничего не делает!!! В Rust они ленивые. 
Если сделать 


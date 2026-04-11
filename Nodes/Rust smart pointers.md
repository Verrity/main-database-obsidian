---




---
---
* [[Rust|Назад]]
---

#### Указатели и разыменование
Умный указатель `a` (указывает на данные в куче, владеет), простой указатель `b` (указывает на указатель, данными не владеет)
```rust unfold
let a: Vec<i32> = vec![1, 2, 3];
let b: &Vec<i32> = &a;
```

**Разыменование (часто делается автоматически)**
* `*` - берет данные. Работает для простых типов (`trait Copy`), не работает для сложных типов (размер которых заранее неизвестен, не поддерживают `trait Copy`)
* `.deref()` - берет слайс на данные. Работает для сложных типов (поддерживают  `trait Deref`), не работает для простых типов (не поддерживают `trair Deref`)

#### Простейший указатель `Box`
<font color="#c0504d">Трейт `Drop` - все равно что деструктор. Его можно вызвать `std::mem::drop(obj)`</font>

Создать `i32` в куче
```rust unfold
let a: Box<i32> = Box::new(42);
```

```rust title="Структура (указатель на структуру) в стеке, имя в куче"
struct Cat {
    name: String,
    age: u8,
}

fn main() {
    let c1 = Cat {
        name: String::from("Kyle"),
        age: 8,
    };
}
```

Но если `Cat` указывает на `Cat`, разместить его в стеке нельзя. Потому что размер структуры становится заранее неизвестным (рекурсивная структура).
```rust unfold
struct Cat {
	name: String,
	age: u8,
	parent: Option<Cat>,
}
```

Нужно создать `Box` (одна структура в стеке, другая в куче)
Но тогда одна структура становится недоступна. А если клонировать - становятся не связаны. Или можно поебаться с lifetime
```rust
struct Cat {
    name: String,
    age: u8,
    parent: Option<Box<Cat>>,
}

fn main() {
    let cat1 = Box::new(Cat {
        name: String::from("Kyle"),
        age: 8,
        parent: None,
    });

    let cat2 = Cat {
        name: String::from("John"),
        age: 3,
        parent: Some(cat1),
    };

    // cat1 - ERROR - moved
}
```

#### Указатель `Rc` (Reference counting)
Не потокобезопасен!
Множественное владение (как `shared_pointer`), но <font color="#c0504d">данные нельзя менять даже если они `mut`</font>
```rust title="Пример Rc указателя"
use std::rc::Rc;

struct Cat {
    name: String,
    age: u8,
    parent: Option<Rc<Cat>>,
}

fn main() {
    let cat1 = Rc::new(Cat {
        name: String::from("Kyle"),
        age: 8,
        parent: None,
    });

    let mut cat2 = Rc::new(Cat {
        name: String::from("John"),
        age: 3,
        parent: Some(Rc::clone(&cat1)), // Копируется указатель на данные (+1 владелец), а не сами данные
    });

    let cat3 = Cat {
        name: String::from("Eminem"),
        age: 3,
        parent: Some(Rc::clone(&cat2)),
    };

    cat2.age = 7; // ERROR

    cat1; // OK
}
```

#### Указатель `RefCell`
Позволяет изменять данные, даже если они помечены как неизменяемые (`let`).
Но `RefCell` не имеет множественного владения. Он единолично владеет данными.
```rust unfold
let a: RefCell<i32> = RefCell::new(5);
let b: &RefCell<i32> = &a;
*a.borrow_mut() += 1; // Заимствовать значение как изменяемое
let c: i32 = *a.borrow_mut();
println!("{}, {}", a.borrow(), b.borrow()); // Заимствовать значение как неизменяемое
```

Пример использования 
```rust title="Пример использования Rc + RefCell"
use std::{ cell::RefCell, rc::Rc };

struct Cat {
    name: String,
    age: u8,
    parent: Option<Rc<RefCell<Cat>>>,
}

fn main() {
    let cat1 = Rc::new(
        RefCell::new(Cat {
            name: String::from("Kyle"),
            age: 8,
            parent: None,
        })
    );

    let cat2 = Rc::new(
        RefCell::new(Cat {
            name: String::from("John"),
            age: 3,
            parent: Some(Rc::clone(&cat1)),
        })
    );

    let cat3 = Rc::new(
        RefCell::new(Cat {
            name: String::from("Eminem"),
            age: 3,
            parent: Some(Rc::clone(&cat2)),
        })
    );

    cat2.borrow_mut().age = 7; // OK

    iterate_parents(&cat3);
}

fn iterate_parents(child: &Rc<RefCell<Cat>>) {
    let mut current = Some(Rc::clone(child));

    while let Some(current_cat) = current.take() {
        let current_cat_ref = current_cat.borrow();
        println!("Cat name {}, age {}", current_cat_ref.name, current_cat_ref.age);

        if let Some(parent) = current_cat_ref.parent.as_ref() {
            let parent_ref = parent.borrow();
            println!(
                "{}'s parent is {}, age {}",
                current_cat_ref.name,
                parent_ref.name,
                parent_ref.age
            );

            current = Some(Rc::clone(parent));
        }

        println!("=====");
    }
}
```

#### Указатель Arc
Позволяет разделять владение одними и теми же данными между несколькими потоками (не позволяет изменять данные, если они не обёрнуты в `Mutex` или `RwLock`).
Как `Rc`, но для многопотока, использует атомарные операции для счетчика ссылок.
`Arc` — это способ сказать компилятору: «Эти данные будут жить столько, сколько нужно всем потокам, и я гарантирую, что мы не получим гонку данных, потому что только читаем».

```rust unfold
use std::sync::Arc;
let shared_numbers = Arc::new(numbers);   // оборачиваем Vec в Arc
// ...
let child_numbers = Arc::clone(&shared_numbers);   // создаём новый ссылающийся Arc для потока
```

#### Указатель Cow
Позволяет иногда владеть данными, а иногда только ссылаться на них, и клонирует данные только в момент, когда их нужно изменить («при записи»). Не является потокобезопасным.
`Cow` позволяет вернуть _либо ссылку_ (если данные не нужно менять), _либо владеющую копию_ (если нужны изменения) — без лишних копирований в случае, когда изменения не требуются.
```rust unfold
use std::borrow::Cow;
fn process(s: &str) -> Cow<'_, str> {
    if s.contains(' ') {
        // Меняем — нужна копия
        Cow::Owned(s.replace(' ', "_"))
    } else {
        // Не меняем — просто ссылка
        Cow::Borrowed(s)
    }
}
let s1 = process("hello world"); // Cow::Owned
let s2 = process("hello");       // Cow::Borrowed
```

Пример лучше
`Cow<[i32]>` Это тип, который может быть **либо ссылкой** (`&[i32]`), **либо вектором** (`Vec<i32>`). Он хранит флаг, какой вариант используется.
Когда ты вызываешь `.to_mut()`, происходит:
- Если внутри ссылка → Rust создаёт новый `Vec`, копирует туда все данные, и заменяет ссылку на этот `Vec`.
- Если внутри `Vec` → просто возвращает `&mut Vec` без копирования.

```rust title="большой пример"
// This exercise explores the `Cow` (Clone-On-Write) smart pointer. It can
// enclose and provide immutable access to borrowed data and clone the data
// lazily when mutation or ownership is required. The type is designed to work
// with general borrowed data via the `Borrow` trait.

use std::borrow::Cow;

fn abs_all(input: &mut Cow<[i32]>) {
    for ind in 0..input.len() {
        let value = input[ind];
        if value < 0 {
            // Clones into a vector if not already owned.
            input.to_mut()[ind] = -value;
        }
    }
}

fn main() {
    // You can optionally experiment here.
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn reference_mutation() {
        // Clone occurs because `input` needs to be mutated.
        let vec = vec![-1, 0, 1];
        let mut input = Cow::from(&vec);
        abs_all(&mut input);
        assert!(matches!(input, Cow::Owned(_)));
    }

    #[test]
    fn reference_no_mutation() {
        // No clone occurs because `input` doesn't need to be mutated.
        let vec = vec![0, 1, 2];
        let mut input = Cow::from(&vec);
        abs_all(&mut input);
        assert!(matches!(input, Cow::Borrowed(_)));
    }

    #[test]
    fn owned_no_mutation() {
        // We can also pass `vec` without `&` so `Cow` owns it directly. In this
        // case, no mutation occurs (all numbers are already absolute) and thus
        // also no clone. But the result is still owned because it was never
        // borrowed or mutated.
        let vec = vec![0, 1, 2];
        let mut input = Cow::from(vec);
        abs_all(&mut input);
        assert!(matches!(input, Cow::Owned(_)));
    }

    #[test]
    fn owned_mutation() {
        // Of course this is also the case if a mutation does occur (not all
        // numbers are absolute). In this case, the call to `to_mut()` in the
        // `abs_all` function returns a reference to the same data as before.
        let vec = vec![-1, 0, 1];
        let mut input = Cow::from(vec);
        abs_all(&mut input);
        assert!(matches!(input, Cow::Owned(_)));
    }
}
```



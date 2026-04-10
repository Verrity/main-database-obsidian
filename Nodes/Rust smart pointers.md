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
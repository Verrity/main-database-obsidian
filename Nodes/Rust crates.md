---




---
---
* [[Rust|Назад]]
---

* crate - Минимальная единица трансляции.
	* Исполняемые
	* Библиотечные
* package - множество бинарных крейтов и один библиотечный крейт

### Создать, использовать, обращаться
```rust unfold
mod generator {
	pub fn generate() -> u8 {
		200
	}
}

use generator::generate; // Use func
// OR
use generator::generate::*; // Use all
// OR
use generator::{generate, new}; // Use this

// Alias
use generator as gen;

fn main() {
	let rand = gen::generate();
}

use crate::generator; // Обращение от корня крейта
```

Можно обращаться к вышележащим модулям через 
```rust unfold
super::<what_to_use_name>
```

***У каждого модуля независимая область видимости!***

Все в модуле по умолчанию создается приватным. Нужно делать public таким способом:
```rust unfold
mod generator {
    pub struct RandomNumber {
        pub value: u8,
    }

    impl RandomNumber {
        pub fn new(value: u8) -> Self {
            Self {
                value,
            }
        }
    }

    pub fn generate() -> RandomNumber {
        RandomNumber::new(200)
    }
}
```

Можно использовать `self`
```rust unfold
mod delicious_snacks {
    pub use self::fruits::PEAR as fruit;

    mod fruits {
        pub const PEAR: &str = "Pear";
        pub const APPLE: &str = "Apple";
    }
}

let a = delicious_snacks::fruit;
```
### Разделение на файлы
1. Создать файл `somename.rs`
	1. Создать не вложенный модуль: file `somename.rs`, in main `use generator::*;` **Имя файла становится именем модуля**
	2. Создать вложенные модуль
		1. Создать папку `somename`
		2. Создать файл `file1.rs`
		3. Чтобы использовать модуль из другого файла нужно его создать
		```rust unfold
		mod file1;
		```
		4. А потом использовать
		```rust unfold
		use crate::file1;
		```
		5. Часто используется прелюдия - объявление в `main.rs`, содержащий все константы/модули, которые требуются в разных местах проекта
		```rust unfold
		mod prelude {
			pub use std::env as enviroment;
			pub use generator::{generate, other_fn};	
		}

		use prelude::*; // То что подключено в прелюдии можно использовать во всех файлах проекта через use::prelude::*;
		```
2. (Deprecated) Создать папку `somename` с файлом `mod.rs`
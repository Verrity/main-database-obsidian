---




---
---
* [[Rust|Назад]]
---

* crate - Минимальная единица трансляции.
	* Исполняемые
	* Библиотечные
* package - множество бинарных крейтов и один библиотечный крейт

Создать, использовать, обращаться
```rust unfold
mod generator {
	pub fn generate() -> u8 {
		200
	}
}

use generator::generate; // Use func
// OR
use generator::generate::*; // Use all

// Alias
use generator as gen;
```

Все в модуле по умолчанию создается приватным
```rust unfold

```
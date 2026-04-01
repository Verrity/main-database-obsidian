---




---
---
* [[Rust типы|Назад]]
---

Простое определение
```rust unfold
#[derive(Debug)] // -- автореализация трейта дебажного вывода, не обязательно
struct Car {
	barand: String,
	speed: u16,
}
```
Определение без полей
```rust unfold
struct ColorRgb(i32, i32, i32);
struct AlwaysEqual;
```
Простое создание и обращение
```rust unfold
let mut car = Car {
	brand: String::from("Ford"),
	speed: 180,
};

let _b = car.speed;
```

### Методы
```rust unfold
impl Car {
	fn new(
		barand: String,
		speed: u16,
	) -> Self {
		Self {
		barand: barand,
		speed: speed,
		}
	}

	fn drive(& mut self, speed: u16) {
		self.speed = speed;
	}
}
```
Если имя поля совпадает с именем переменной в этом поле, можно не писать `field: field`

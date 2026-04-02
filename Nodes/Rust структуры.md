---




---
---
* [[Rust типы|Назад]]
---

### Простое определение
***По умолчанию и struct и все ее поля private!! (см. модули)***
```rust unfold
#[derive(Debug)] // -- автореализация трейта дебажного вывода, не обязательно
struct Car {
	barand: String,
	speed: u16,
}
```
### Определение без полей
```rust unfold
struct ColorRgb(i32, i32, i32);
struct AlwaysEqual;
```
### Простое создание и обращение
```rust unfold
let mut car = Car {
	brand: String::from("Ford"),
	speed: 180,
};

let _b = car.speed;
```
### Создание на основе
***Перемещает сложные поля!***
```rust unfold
struct Car {
	brand: String,
	speed: u16,
	a: String
}

let car1 = Car::new(String::from("Ford"), 32, String::from("aaaa"));
let car2 = Car {
	brand: String::from("Audi"),
	..car1
};
let s = &car1.a; // Error - data already moved!!
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
Если имя поля совпадает с именем переменной в этом же `scope`, можно не писать `field: field`
```rust unfold
impl Car {
	fn new(
		barand: String,
		speed: u16,
	) -> Self {
		Self {
			barand,
			speed,
		}
	}
```
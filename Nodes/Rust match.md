---




---
---
* [[Rust|Назад]]
---
Обычная запись
```rust unfold
match choise {
	1 => Some(c_to_f(temp)),
	2 => Some(f_to_c(temp)),
	_ => None,
}
```

Сокращенная запись
```rust unfold
if let Some(temp) = convert(temperature, choise)
{
	println!("Converted: {temp}");
}
```
Сравнение чисел
```rust unfold
match val1.cmp(val2) {
	Ordering::Equal => return val1;
	Ordering::Greater => return val2;
	Ordering::Less => {
		let _b = "do something";
		val1
	}
}
```
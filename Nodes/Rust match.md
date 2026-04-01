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
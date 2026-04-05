---




---
---
* [[Rust|Назад]]
---

```rust unfold
fn sum<T>(numbers: &[T]) -> T
where
	T: std::maker::Copy + std::ops::Add<Output = T>
{
	numbers.iter().copied().reduce(|acc, n| acc + n).unwrap()
}

fn sum<T: std::maker::Copy + std::ops::Add<Output = T>>(numbers: &[T]) -> T {
	numbers.iter().copied().reduce(|acc, n| acc + n).unwrap()
}
```

```rust unfold
trait Die {
	fn die(&self) {
		// ...
	}
}

struct Employee<T,U> {
	age: T,
	salary: U,
	tax: T
}

impl<T,U> Employee<T,U> {
	fn salary_with_tax(&self) -> U {
		self.salary - (self.sala)
	}
}

impl Die for Employee {
	fn die(&self) {
		// ...
	}
}
```

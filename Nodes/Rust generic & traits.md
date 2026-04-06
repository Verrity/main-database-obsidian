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

Реализация трейта для `String`
```rust unfold
trait AppendBar {
	fn append_bar(self) -> Self;
}

impl AppendBar for Vec<String> {
    fn append_bar(mut self) -> Self {
        self.push(String::from("Bar"));
        self
    }
}
```

Реализация дефолтного трейта
```rust unfold
trait Licensed {
	fn licensing_info(&self) -> String {
		String::from("Some information")
	}
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
    tax: U,
}

impl<T, U: std::marker::Copy + std::ops::Sub<Output = U> + std::ops::Mul<Output = U>> Employee<T,U> {
    fn salary_with_taxes(&self) -> U {
        self.salary - (self.salary * self.tax)
    }
}
// ИЛИ
impl<T, U> Employee<T, U>
where
    U: Copy + std::ops::Sub<Output = U> + std::ops::Mul<Output = U>,
{
    fn salary_with_taxes(&self) -> U {
        self.salary - (self.salary * self.tax)
    }
}

impl<T,U> Die for Employee<T,U> {
    fn die(&self) {
        // ...
    }
}


fn main() {
    println!("Hello, world!");
}
```

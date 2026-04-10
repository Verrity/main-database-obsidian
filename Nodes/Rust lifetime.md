---




---
---
* [[Rust|Назад]]
---

Выходное значение будет существовать, пока существуют оба входных значения (потому выходное значение может быть любым входным значением). Если здесь будет возвращаться всегда `vec1`, например, тогда lifetime не нужен.
```rust unfold
fn main() {
    let v1 = vec![1, 2, 3];
    let v2 = vec![4, 5, 6];

    let _v = get_vec_with_bigger_sum(&v1, &v2);
}

fn get_vec_with_bigger_sum<'a>(vec1: &'a Vec<i32>, vec2: &'a Vec<i32>) -> &'a Vec<i32>
{
    if vec1.iter().sum::<i32>() > vec2.iter().sum::<i32>() {
        &vec1
    } else {
        &vec2
    }
}

```

Вот так указывать в случае с generic
```rust unfold
fn sum<'a, T>(numbers: &'a [T]) -> T where T: std::iter::Sum<&'a T> {
	numbers.iter().sum()
}
```

Вот так в случае со структурами
```rust unfold
struct Test<'a> {
    field: &'a str,
}

impl<'a> Test<'a> {
    fn call_me(&self) -> &str {
        self.field
    }
}
```

Статический lifetime `&'static` - будет жить, пока программа работает. По умолчанию присваивается литералам, но можно использовать его и для других типов.
```rust unfold
let a: &str = "literal string";

fn str_data() -> &'static str {
	"Hello, world!"
}
```

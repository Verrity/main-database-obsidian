---




---
---
* [[Rust|Назад]]
---

* `cargo test`

```rust unfold
#[cfg(test)]
mod tests {
	use super::*;
	
	const ARR: [i32, 10] = [-1, 3, 5, 7, 8, 10, 24, 37, 42, 135];
	
	#[test]
	fn element_fond() {
		assert_eq!((-1, 0), bin_search_func(&ARR, -1).unwrap());
	}
}
```
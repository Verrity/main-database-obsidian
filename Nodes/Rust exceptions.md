---




---
---
* [[Rust|Назад]]
---

* `panic!()` - завершает программу

*Result*
```rust unfold
use std::{env::current_dir, path};

fn main() {
	let dir: Result<std::path::PathBuf, std::io::Error> = current_dir();
	let mut target = match dir {
		Ok(path) => path,
		Err(e ) => panic!("failed"),
	};

    target.push("?");

    create_dir_all(&target).unwrap(); // panic on error
    // OR
    create_dir_all(&target).unwrap_or_else(|e| { panic!("Error: {e:?}!"); });
    // OR
    create_dir_all(&target).expect("Failed"); // panic with message
    // OR
    match create_dir_all(&target) {
        Ok(()) => println!("Created"), // () - пустой кортэж, его возвращают когда значения нет
        Err(e) => match e.kind() {
            ErrorKind::InvalidFilename => { panic!("Invalid filename"); }
            _ => { panic!("Unhandled error"); }
        }
    }
}
```
Создать Result

`?` - в случае ошибки вернуть ошибку как результат работы. Работает для `Result` и `Optional`
```rust unfold
fn get_current_path() -> Result<PathBuf, std::io::Error> {
	let path = current_dir()?;
	Ok(path) // Явно преобразовать в Result
}
```
Вернуть ошибку в случае `std::io`
```rust unfold
pub main -> std::io::Result<()> {
	Ok(())
}
```
Вернуть ошибку в случае любой ошибки
```rust unfold
pub main -> std::io::Result<(), Box<dyn std::error::Error>> {
	Ok(())
}
```
---




---
---
* [[Rust|Назад]]
---

* Unit tests - одна функция, вход/выход изолированно
* Integration tests - функционал вместе
* system tests - на конкретном железе
* e2e tests - проверять сценарий работы от А до Я
* smoke tests - базовый функционал, приблизительно правильно работает

#### Unit-tests

#### Запуск тестов
* `cargo test`
* `cargo test -- --show-output` - выводить `println!()` и прочее
* `cargo test -- --test-threads=1` - указать количество потоков на тест
* `cargo test it_always_panics` - запустить конкретный тест
* `cargo test panics` - запустит все тесты, которые в названии содержат `panics`
* `cargo test -- --ignored` - запустить тесты с `#[ignore]`
* `cargo test -- --include-ignored` - запустить все тесты, включая `#[ignore]`

Тесты могут возвращать и `assert` и `Result`

```rust title="Пример"
fn str_data() -> &'static str {
    "Hello, world!"
}

fn boom() {
    println!("3... 2... 1..."); // В тестах не печатает stdout без --show-output
    eprintln!("Error 3... 2... 1..."); // stderror
    panic!("boom!")
}

fn result(a: u8) -> Result<u8, String>
{
    if a < 100 {
        Ok(a)
    } else {
        Err(format!("Number {} is too large", a))
    }
}

#[cfg(test)]
mod tests {
    use crate::*;
    // OR
    // use super::*;

    #[test]
    fn it_panics_when_a_is_too_large() -> Result<(), String>
    {
        let res = result(90);
        if res.is_err() {
            Err("Got an error!".into())
        } else {
            Ok(())
        }
    }

    #[test]
    #[ignore] // Не запускать, если это не указано явно
    fn it_returns_true1() {
        assert!(true)
    }

    #[test]
    fn it_returns_true2() {
        assert_eq!(true, true)
    }

    #[test]
    fn it_returns_true3() {
        assert_ne!(true, false)
    }

    #[test]
    fn it_returns_str() {
        // 3 параметр - строка с ошибкой
        assert_eq!(str_data(), "Hello, world!", "Invalid string has been returned")
    }

    #[test]
    #[should_panic] // Без этого тест будет провален при panic!() (ЛЮБАЯ ОШИБКА)
    fn it_always_panics1() {
        boom();
    }

    #[test]
    #[should_panic(expected = "boom!")] // Ждет исключение с указанным значением
    fn it_always_panics2() {
        boom();
    }
}
```

#### Integration-tests

Создать директорию `tests` в корне
(Можно тестировать только то что в библиотеках - <font color="#c0504d">зарезервированное имя для корневого файла библиотеки `lib.rs`. Для тестов - `tests/*.rs`</font>)

Создать файл `lib.rs`
```rust unfold
pub fn demo() -> u8 {
    42
}
```


Создать файл `some_integration_test.rs`
```rust unfold
use l8_tests::demo;

#[test]
fn test_demo() {
    assert_eq!(demo(), 42);
}
```


#### Зависимости только для разработки
```toml title="Cargo.toml"
[dev-dependencies]
pretty_assertions = "1" - пишет в более симпотичном виде
```
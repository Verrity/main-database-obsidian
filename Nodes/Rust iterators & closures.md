---




---
---
* [[Rust|Назад]]
---

#### Closure
По умолчанию делает заимствование захваченной переменной!!!
Ничего не делает до вызова.
`{}` скобки можно не ставить, если closure занимает 1 строку

```rust title="Пример"
fn plus_(param: u8) -> u8 {
    42 + param
}

fn main() {
    let some_param = 10;
    let value: Option<u8> = None;

    // Closure
    let res: u8 = value.unwrap_or_else(|| -> u8 {
        plus_(some_param) // Захватывает параметры
    });

    // Можно хранить в переменной
    let closure = || -> u8 { plus_(some_param) };
    // closure() - выполнить
    // closure - передать, чтобы выполнилось внутри
    let res: u8 = value.unwrap_or_else(closure);

    let mut b = 16;
    let mut demo = |a| {
        println!("{a}");
        b += a;
    };

    demo(10);
    println!("{b}");

    let mut v = vec![1,2,3];
    let mut process = || v.push(3);

    process();

    println!("{:?}", v);
    
	// Чтобы переместить значение нужно сделать move!!
	let mut process2 = move || v.push(3);
}
```

#### Итераторы
Итератор - это объект, который будет итерироваться по коллекции.
**Итераторы в Rust ленивые (lazy) — они ничего не делают, пока вы не вызовете "пожирающий" метод (адаптер - consuming adapter)**

* Примеры **Ленивых методов**
	* `map()`       преобразование
	* `filter()`    фильтрация
	* `take()`     ограничение
	* `skip()`     пропуск
	* `flat_map()`  сплющивание
	* `zip()`      сжатие
	* `enumerate()`  нумерация
* Примеры **Пожирающих методов**
	* `collect()`  собирает в коллекцию
	* `for_each()` выполняет для каждого
	* `sum()`  суммирует
	* `count()`  считает
	* `fold()`  сворачивает
	* `any() / all()`  проверяет условия
	* `next()`  берет следующий элемент

```rust title="Пример"
fn main() {
    let v1 = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    let mut i = v1.iter();
    let v2: Vec<i32> = i
        .map(|x| {
            println!("v3: element {x}");
            x + 1
        })
        .take(2)
        .collect(); // Взять 2 элемента, собрать вектор
    /* Output:
        v3: element 1
        v3: element 2
    */
    // Операция map передана, но не выполнится до действия!!!
    // То есть, мы сначала возьмем 2 элемента, а потом сделаем map к ним, а не наоборот!!

    let v = vec!["zero", "one", "two", "three", "four", "five"];
    // .enumerate заменит строки на пары (индекс; строка)
    // .fold собирает все значения в один результат. init - начальное значение
    v.iter()
        .enumerate()
        .filter(|(i, _)| *i % 2 == 1)
        .map(|(i, _)| i * i)
        .fold(0, |result, i| result + i);
}

```
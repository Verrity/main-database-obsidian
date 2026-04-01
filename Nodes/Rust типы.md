---




---
---
* [[Rust|Назад]]
---
Типизация статическая (нужно указывать типы данных), но если можно явно понять какой тип содержит переменная, писать его не обязательно.

### **Скалярные типы данных (Scalar)**
* Integer: `i8 i16, i32, i64, i128`; `isize, usize` - размер зависит от архитектуры процессора `64 -> 64`, `32 -> 32`.
* Float: `u8, u16, u32, u64, u128`. 
* `f16, f32, f64, f128`
* `bool` (`true`/`false`)
* `char` 

### **Составные типы данных (Compound)**
* `Tuple (Кортэж)`. Может хранить разные типы данных.
```rust unfold
let tup: (i8, char, bool) = (42, 'f', false);
let a = tup.0 // 42
// Декструктурируем кортеж
let cat = ("Funny man", 3.5);
let (name, age) = cat;
```
* `Array` (fixed size)
```rust unfold
let arr: [i32; 3] = [1,2,3];
let arr2 = [1,2,3,];
let a = [42; 100]; // 100 элементов, все равны 42
let a = arr[1];
```

По умолчанию переменные считаются неизменяемыми `let`
Можно создать изменяемые переменные `let mut`

### **Диапазон (Range)**
* `0..5`

### **Слайсы**
* `str`
* `&[i32]`
```rust unfold
let a = [1, 2, 3, 4, 5];
let nice_slice = &a[1..=3; // 2,3,4
let nice_slice = &a[1..3]; // 2,3
```


Вставка в макрос insert
```rust unfold
let a: i32 = 42;
println!("Hello {a} world")
```

Создание констант **const**
```rust unfold
const PI: f64 = 3.14;
```
Оператор возведения в степень
```rust unfold
c = 2 ** 16
```

Небольшой пример
```rust unfold
const C: f32 = 32.0;

fn c_to_f(celsius_temp: f32) -> f32 {
    (celsius_temp * (9.0 / 5.0)) + C
}

fn f_to_c(fahrenheit_temp: f32) -> f32 {
    (fahrenheit_temp - C) * (5.0 / 9.0)
}

fn convert(temp: f32, choise: u8) -> Option<f32> {
    match choise {
        1 => Some(c_to_f(temp)),
        2 => Some(f_to_c(temp)),
        _ => None,
    }
}

fn main() {
    println!("Temperature converter. \n (1) C to F\n (2) F to C");

    // let mut user_choise = String::new();
    let choise_str = std::io::read_to_string(std::io::stdin()).unwrap();
    let choise = choise_str.trim().parse::<u8>().expect("Number has expected");

    println!("Enter temperature: ");

    // let mut temperature = String::new();
    let temperature_str = std::io::read_to_string(std::io::stdin()).unwrap();
    let temperature = temperature_str.trim().parse::<f32>().expect("Number has expected");

    if let Some(temp) = convert(temperature, choise)
    {
        println!("Converted: {temp}");
    } else {
        println!("Unknown conversation choise: {choise}");
    }
}
```

### Циклы
```rust unfold
let a = loop {
	//...
	continue;
	break 42;
} 

for el in array {
}
```

### Вектор
```rust unfold
let a = vec![10, 20, 30];
```
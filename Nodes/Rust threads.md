---




---
---
* [[Rust|Назад]]
---

Создать поток. По умолчанию не ждет окончания потоков (для этого нужно использовать `.join()`).
```rust unfold
fn main() {
    let handle = thread::spawn(move || {
        println!("hi");
        thread::sleep(Duration::from_millis(100));
    });

    handle.join();
}
```

Поток это Closure. По умолчанию в него move сложные типы и copy простые типы.

#### Примитив обмена Mpsc
 механизм для безопасной передачи данных между потоками, следующий принципу «**M**ulti-**P**roducer, **S**ingle-**C**onsumer» (много производителей, один потребитель). Он реализован в стандартной библиотеке как `std::sync::mpsc`.
 
 Вместо того чтобы блокировать общую память, потоки отправляют друг другу _сообщения_ через канал, который состоит из двух концов:
- **`Sender<T>` (tx)**: отправляющий конец. Он может быть клонирован, что позволяет создать множество «производителей» данных[](https://google.github.io/comprehensive-rust/da/concurrency/channels.html).
- **`Receiver<T>` (rx)**: принимающий конец. Он уникален и существует в единственном экземпляре, выступая в роли единственного «потребителя»

Стандартная библиотека Rust предоставляет два вида каналов:
- **Асинхронный канал (`mpsc::channel()`)**[](https://docs.rs/rustemo/0.1.0/rustemo/std/sync/mpsc/index.html):
       - Имеет бесконечный (растущий) буфер.
    - Отправка (`send`) никогда не блокирует поток.
    - Может привести к большому потреблению памяти, если потребитель не успевает обрабатывать данные.
- **Синхронный канал (`mpsc::sync_channel(n)`)**[](https://docs.rs/rustemo/0.1.0/rustemo/std/sync/mpsc/index.html):
	- Имеет ограниченный буфер на `n` элементов.
	- Если буфер заполнен, отправка (`send`) заблокирует поток, пока потребитель не освободит место.
	- Позволяет регулировать нагрузку и контролировать переполнение.

```rust title="Простой пример"
fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let recv = rx.recv().unwrap();
    println!("Got: {}", recv);

    let recv = rx.recv().unwrap(); // ERROR (неблокирующее получение, сообщений нет)
}
```
```rust title="Другой простой пример"
fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_millis(300));
        }
    });

    for recv in rx {
        println!("Got: {}", recv);
    }
}
```

#### Примитив синхронизации Mutex (похож на RefCell)
```rust title="Простой пример"
    let data = vec![
        String::from("hi"),
        String::from("from"),
        String::from("the"),
        String::from("mutex"),
    ];

    let m = Mutex::from(data);

    {
        let mut val = m.lock().unwrap();
        (*val).push(String::from("additional string"));
    }
```
```rust title="Пример с мьютексом чуть сложнее"
fn main() {
    let counter = Arc::new(Mutex::new(1000));
    let mut handlers = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });

        handlers.push(handle);
    }

    for handler in handlers {
        handler.join().unwrap();
    }
}
```


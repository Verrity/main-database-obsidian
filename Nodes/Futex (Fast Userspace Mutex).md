---




---
---
* [[Синхронизация (User Space)|Назад]]
---
Fast Userspace Mutex (futex) — это фундаментальный примитив синхронизации в ядре Linux, предоставляющий эффективный механизм для реализации блокирующих операций в пользовательском пространстве (userspace). Его основная идея заключается в том, чтобы избегать дорогостоящих системных вызовов (syscalls) в неконкурентном (uncontended) случае и обращаться к ядру только тогда, когда поток действительно необходимо заблокировать или разбудить [](https://www.kerrisk.com/linux/man-pages/man7/futex.7.html)[](https://fuchsia.googlesource.com/fuchsia/+show/64146007cd0e132002a7f326f7e7c4722c2928d6/docs/futex.md).

### Основная концепция и мотивация

До появления futex (впервые реализованы в Linux 2.5.x) реализация блокирующих примитивов, таких как мьютексы, требовала либо постоянного нахождения в ядре (что медленно), либо использования ненадежных или неэффективных пользовательских алгоритмов. Futex был спроектирован как строительный блок (building block), который позволяет создавать высокопроизводительные механизмы синхронизации (например, `pthread_mutex_t` в библиотеке NPTL) [](https://www.kerrisk.com/linux/man-pages/man7/futex.7.html)[](https://browse.library.kiwix.org/content/stackoverflow.com_en_all_2023-11/questions/6364314/why-is-a-pthread-mutex-considered-slower-than-a-futex).

Ключевая идея futex заключается в разделении ответственности:

1. **Пользовательское пространство:** Хранит основное состояние блокировки (счетчик или флаг) в виде 32-битного целочисленного значения (`futex word`), доступного всем потокам/процессам [](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
    
2. **Ядро:** Поддерживает очередь ожидания (`waitqueue`) для потоков, заблокированных на конкретном адресе этого 32-битного слова. Ядро не интерпретирует значение слова, а лишь использует его адрес как ключ [](https://fuchsia.googlesource.com/fuchsia/+show/64146007cd0e132002a7f326f7e7c4722c2928d6/docs/futex.md)[](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
    

### Принцип работы: Алгоритм на основе атомарных операций

Взаимодействие между userspace и kernel space строится на атомарных операциях сравнения с обменом (compare-and-swap), доступных на уровне процессора, и системном вызове `futex()` [](https://www.kerrisk.com/linux/man-pages/man7/futex.7.html)[](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).

Рассмотрим классический алгоритм работы мьютекса на основе futex. Состояние futex-слова может быть, например:

- `0` — мьютекс свободен.
    
- `1` — мьюкс захвачен, ожидающих нет.
    
- `-1` — мьютекс захвачен, есть ожидающие в очереди ядра.
    

#### Захват мьютекса (lock)

1. **Быстрый путь (fast path):** Поток выполняет атомарную инструкцию (например, `cmpxchg`) в userspace, пытаясь изменить значение futex-слова с `0` (свободно) на `1` (захвачено). Если операция успешна, мьютекс захвачен без участия ядра [](https://www.kerrisk.com/linux/man-pages/man7/futex.7.html)[](https://browse.library.kiwix.org/content/stackoverflow.com_en_all_2023-11/questions/6364314/why-is-a-pthread-mutex-considered-slower-than-a-futex).
    
2. **Медленный путь (slow path):** Если атомарная операция показывает, что слово не равно `0` (мьютекс уже захвачен), поток должен:
    
    - Изменить значение слова на `-1` (или другое значение, сигнализирующее о наличии ожидающих).
        
    - Вызвать системный вызов `futex(uaddr, FUTEX_WAIT, val, ...)`. Здесь `uaddr` — адрес futex-слова, `val` — ожидаемое значение (`-1`). Ядро атомарно проверяет, что значение по адресу `uaddr` все еще равно `val`, и если это так, переводит вызывающий поток в состояние ожидания (`TASK_INTERRUPTIBLE`) и добавляет его в очередь ожидания, связанную с этим адресом. Если значение изменилось (например, другой поток успел освободить мьютекс), системный вызов немедленно возвращается с ошибкой `EAGAIN`, и поток повторяет попытку захвата [](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
        

#### Освобождение мьютекса (unlock)

1. **Быстрый путь:** Поток сбрасывает значение futex-слова с `1` на `0` с помощью атомарной операции в userspace. Если после этого он видит, что старое значение было `1` (т.е. не было ожидающих), работа завершена [](https://www.kerrisk.com/linux/man-pages/man7/futex.7.html).
    
2. **Медленный путь:** Если атомарная операция показывает, что старое значение было `-1` (были ожидающие), поток должен вызвать системный вызов `futex(uaddr, FUTEX_WAKE, 1, ...)`. Это указывает ядру разбудить один поток из очереди ожидания, связанной с адресом `uaddr` [](https://www.kerrisk.com/linux/man-pages/man7/futex.7.html)[](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
    

Этот дизайн гарантирует, что системный вызов происходит только при реальной конкуренции (contention), что делает неконкурентные операции захвата и освобождения исключительно быстрыми (порядка десятков наносекунд) [](https://habr.com/en/articles/1000192/comments/)[](http://bbs.warensemble.com/?page=001-forum.ssjs&sub=unet-clangcpp&thread=55479).

### Системный вызов `futex()`

Для взаимодействия с ядром используется системный вызов `futex()`. В glibc нет обертки для него, поэтому программисты обычно используют `syscall(SYS_futex, ...)` [](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).

```c unfold
#include <linux/futex.h>
#include <sys/syscall.h>
#include <unistd.h>
long syscall(SYS_futex, uint32_t *uaddr, int futex_op, uint32_t val,
             const struct timespec *timeout, /* or: uint32_t val2 */
             uint32_t *uaddr2, uint32_t val3);
```

- `uaddr`: Указатель на 32-битное futex-слово в userspace. Оно должно быть выровнено [](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
    
- `futex_op`: Операция, которую нужно выполнить (например, `FUTEX_WAIT`, `FUTEX_WAKE`). К ней можно применить флаги, такие как `FUTEX_PRIVATE_FLAG` [](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
    
- `val`: Смысл зависит от операции. Для `FUTEX_WAIT` это ожидаемое значение; для `FUTEX_WAKE` — максимальное количество пробуждаемых потоков [](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
    

Ключевые операции:

- **`FUTEX_WAIT`**: Проверяет, что по адресу `uaddr` по-прежнему находится значение `val`, и если да — блокирует поток. Это атомарная операция "проверка-и-блокировка", предотвращающая race conditions [](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
    
- **`FUTEX_WAKE`**: Пробуждает до `val` потоков, ожидающих на `uaddr` [](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
    
- **`FUTEX_REQUEUE` / `FUTEX_CMP_REQUEUE`**: Позволяет переместить ожидающие потоки с одного futex-адреса на другой без лишних пробуждений и засыпаний. Это критически важно для эффективной реализации условных переменных (condition variables) [](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
    
- **`FUTEX_PRIVATE_FLAG`**: Флаг, указывающий, что futex используется только для синхронизации потоков одного процесса. Это позволяет ядру применять оптимизации, так как не нужно заботиться о виртуальных адресах разных процессов [](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
    

### Futex в контексте C/C++ и POSIX

Для подавляющего большинства разработчиков прямое использование futex не требуется и не рекомендуется из-за высокой сложности и подверженности ошибкам [](https://www.kerrisk.com/linux/man-pages/man7/futex.7.html)[](https://browse.library.kiwix.org/content/stackoverflow.com_en_all_2023-11/questions/6364314/why-is-a-pthread-mutex-considered-slower-than-a-futex)[](http://bbs.warensemble.com/?page=001-forum.ssjs&sub=unet-clangcpp&thread=55479). Стандартные механизмы синхронизации спроектированы как эффективные обертки над futex.

- **POSIX Threads (NPTL):** В современной Linux-реализации NPTL (Native POSIX Threads Library) все примитивы, такие как `pthread_mutex_t`, `pthread_cond_t`, `pthread_rwlock_t`, реализованы поверх futex [](https://www.kerrisk.com/linux/man-pages/man7/futex.7.html)[](https://habr.com/en/articles/1000192/comments/). Это означает, что `pthread_mutex_lock` в неконкурентном случае выполнит несколько атомарных инструкций и вернется, не входя в ядро [](https://browse.library.kiwix.org/content/stackoverflow.com_en_all_2023-11/questions/6364314/why-is-a-pthread-mutex-considered-slower-than-a-futex).
    
- **C++17 и C++20:** В C++17 появились `std::shared_mutex`, а в C++20 — методы `wait()`, `notify_one()` и `notify_all()` для атомарных типов (например, `std::atomic<int>::wait`). В стандартных реализациях для Linux эти возможности также строятся на базе futex [](https://webkit.crisal.io/rust/source/library/std/src/sys/pal/unix/futex.rs)[](http://bbs.warensemble.com/?page=001-forum.ssjs&sub=unet-clangcpp&thread=55479).
    

### Технические нюансы и сложности

- **Атомарность:** Корректная работа с futex требует глубокого понимания модели памяти и атомарных операций. Неправильное использование барьеров памяти (memory barriers) может привести к потерянным пробуждениям (lost wakeups) или состояниям гонки (race conditions) [](https://fuchsia.googlesource.com/fuchsia/+show/64146007cd0e132002a7f326f7e7c4722c2928d6/docs/futex.md)[](http://bbs.warensemble.com/?page=001-forum.ssjs&sub=unet-clangcpp&thread=55479).
    
- **Приватные vs. разделяемые futex:** Разделяемые futex (используемые между процессами через общую память, например, `mmap`) требуют от ядра более сложной логики, так как один и тот же физический адрес может отображаться на разные виртуальные адреса в разных процессах. Флаг `FUTEX_PRIVATE_FLAG` отключает эту поддержку, повышая производительность для внутрипроцессной синхронизации [](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
    
- **Обработка таймаутов:** Для операций ожидания можно задать таймаут. Важно различать относительные (используются в `FUTEX_WAIT`) и абсолютные (используются в `FUTEX_WAIT_BITSET`) таймауты, а также источник времени (`CLOCK_MONOTONIC` или `CLOCK_REALTIME`) [](https://manpages.opensuse.org/Leap-16.0/man-pages/futex.2.en.html).
    
- **Поведение при высоких нагрузках:** Как показывают практические тесты, эффективность пользовательской реализации блокировки на сырых futex может сильно деградировать при высокой конкуренции по сравнению с оптимизированной реализацией `pthread_mutex`, которая использует адаптивные алгоритмы (spin-then-futex-wait) [](http://bbs.warensemble.com/?page=001-forum.ssjs&sub=unet-clangcpp&thread=55479).
    

### Заключение

Fast Userspace Mutex — это элегантный и высокоэффективный механизм ядра Linux, который служит основой для всех высокоуровневых примитивов синхронизации. Его главное достижение — возможность выполнять блокирующие операции, оставаясь в userspace в 99% случаев, и обращаться к ядру только для управления очередями ожидания. Понимание принципов работы futex необходимо для низкоуровневой оптимизации, отладки сложных многопоточных приложений и, конечно, для разработки собственных библиотек синхронизации, хотя на практике подавляющее большинство разработчиков используют готовые и хорошо отлаженные обертки в виде POSIX мьютексов или примитивов C++
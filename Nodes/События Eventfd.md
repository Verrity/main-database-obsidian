---




---
---
* [[Синхронизация (User Space)|Назад]]
---
Механизм `eventfd` представляет собой легковесный механизм ядра Linux для уведомления о событиях, реализованный в виде файлового дескриптора. Он был представлен Давиде Либенци (Davide Libenzi) в ядре версии 2.6.22 (commit `e1ad7468`) как более эффективная альтернатива трубам (pipe) для простой сигнализации между процессами или между ядром и пользовательским пространством [](https://www.kernel.dk/cgit/stable/commit/include/linux/eventfd.h?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://manpage.me/index.cgi?q=eventfd&sektion=2&apropos=0&manpath=Debian+8.1.0). Ниже представлен детальный разбор его работы, основанный на исходном коде ядра и официальной документации.

## Объяснение

### 1. Концепция и структура данных

В отличие от pipe, который требует двух файловых дескрипторов и буферизованного ввода-вывода, `eventfd` использует всего один дескриптор и хранит в ядре всего одно значение — **64-битный беззнаковый счетчик (`uint64_t`)** [](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://manpages.debian.org/bookworm/manpages-ru-dev/eventfd_write.3)[](https://manpage.me/index.cgi?q=eventfd&sektion=2&apropos=0&manpath=Debian+8.1.0). Именно простота этого состояния обеспечивает низкие накладные расходы.

Внутри ядра каждый объект `eventfd` представлен структурой `struct eventfd_ctx`, определенной в `fs/eventfd.c` [](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0):

```c unfold
struct eventfd_ctx {
    spinlock_t lock;          // Спин-лок для атомарного доступа к count
    wait_queue_head_t wqh;    // Очередь ожидания для блокирующих операций read/write и poll
    __u64 count;              // Текущее значение счетчика
};
```

- **`count`**: Основной элемент состояния. Его значение определяет, есть ли событие.
    
- **`lock`**: Обеспечивает атомарность операций увеличения/уменьшения счетчика и доступа к очереди ожидания в контексте, где могут одновременно выполняться несколько потоков или даже код из атомарного контекста ядра.
    
- **`wqh`**: Очередь, в которой спят процессы, ожидающие изменения состояния `count` (например, вызов `read` при нулевом счетчике).
    

### 2. Создание: системный вызов `eventfd()`

Создание объекта начинается с системного вызова, который имеет два варианта: старый `eventfd()` (без флагов) и новый `eventfd2()` (с флагами). Glibc предоставляет обертку, использующую более новый вызов, если он поддерживается ядром [](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://manpages.debian.org/bookworm/manpages-ru-dev/eventfd_write.3)[](https://manpages.debian.org/bullseye-backports/manpages-ru-dev/eventfd2.2.ru.html).

```c unfold
int eventfd(unsigned int initval, int flags);
```

- **`initval`**: Устанавливает начальное значение счетчика `count` [](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html).
    
- **`flags`**: Может содержать комбинацию следующих флагов (bitwise OR) [](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://manpages.debian.org/bookworm/manpages-ru-dev/eventfd_write.3)[](https://manpages.debian.org/bullseye-backports/manpages-ru-dev/eventfd2.2.ru.html):
    
    - `EFD_CLOEXEC`: Устанавливает флаг закрытия при exec (аналогично `O_CLOEXEC`).
        
    - `EFD_NONBLOCK`: Устанавливает неблокирующий режим (аналогично `O_NONBLOCK`).
        
    - `EFD_SEMAPHORE`: Включает семафорную семантику для операции `read` (см. ниже). Этот флаг появился в Linux 2.6.30 [](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html).
        

Ядро создает новый экземпляр структуры `eventfd_ctx`, инициализирует `count` значением `initval`, спин-лок и очередь ожидания, после чего связывает его с новым файловым дескриптором, используя механизм анонимных inode [](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://manpages.debian.org/bookworm/manpages-ru-dev/eventfd_write.3). Файловые операции (`f_op`) этого дескриптора указывают на реализованные в ядре функции `eventfd_read`, `eventfd_write`, `eventfd_poll` и т.д.

### 3. Операции над файловым дескриптором

#### Чтение (`read(2)`)

Системный вызов `read` извлекает значение из счетчика. Его поведение строго зависит от флага `EFD_SEMAPHORE` и текущего значения `count` [](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://manpage.me/index.cgi?q=eventfd&sektion=2&apropos=0&manpath=Debian+8.1.0).

1. **Проверка буфера**: Буфер пользователя должен быть не менее 8 байт, иначе возвращается `EINVAL`.
    
2. **Ожидание**: Если счетчик равен нулю, читающий процесс блокируется (помещается в очередь `wqh`) до тех пор, пока счетчик не станет больше нуля. Если установлен флаг `O_NONBLOCK`, то вместо блокировки возвращается `EAGAIN` [](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://manpages.debian.org/bookworm/manpages-ru-dev/eventfd_write.3).
    
3. **Извлечение значения** (выполняется атомарно под защитой `spinlock`):
    
    - **Без `EFD_SEMAPHORE` (режим по умолчанию)**: Процесс читает текущее значение счетчика, и счетчик **сбрасывается в ноль** [](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://manpages.debian.org/bookworm/manpages-ru-dev/eventfd_write.3).
        
    - **С флагом `EFD_SEMAPHORE` (семафорный режим)**: Процесс читает значение **1**, а счетчик **уменьшается на 1** [](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://manpages.debian.org/bookworm/manpages-ru-dev/eventfd_write.3)[](https://manpages.debian.org/bullseye-backports/manpages-ru-dev/eventfd2.2.ru.html).
        
4. **Возврат данных**: Прочитанное 64-битное значение копируется в пользовательский буфер. Успешный `read` всегда возвращает 8 (количество прочитанных байт).
    

#### Запись (`write(2)`)

Системный вызов `write` добавляет значение к счетчику, сигнализируя о событии [](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://manpage.me/index.cgi?q=eventfd&sektion=2&apropos=0&manpath=Debian+8.1.0).

1. **Проверка**: Буфер пользователя должен содержать ровно 8 байт, а записываемое значение не должно равняться `0xffffffffffffffff` (`UINT64_MAX`). В противном случае возвращается `EINVAL` [](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://manpages.debian.org/bookworm/manpages-ru-dev/eventfd_write.3).
    
2. **Условное ожидание**: Если добавление переданного 8-байтного значения к текущему `count` приведет к переполнению (превышению `UINT64_MAX - 1`, т.е. `0xfffffffffffffffe`), то операция записи блокируется до тех пор, пока счетчик не будет уменьшен (с помощью `read`), освобождая место. В неблокируемом режиме такая ситуация приводит к возврату `EAGAIN` [](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://manpages.debian.org/bookworm/manpages-ru-dev/eventfd_write.3).
    
3. **Увеличение счетчика** (атомарно под `spinlock`): К текущему значению `count` добавляется переданное значение. Ядро гарантирует, что операция `write` из пользовательского пространства никогда не вызовет переполнения, блокируясь при необходимости [](https://www.kernel.dk/cgit/stable/commit/include/linux/eventfd.h?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0).
    
4. **Пробуждение**: После успешного увеличения счетчика ядро пробуждает все процессы, ожидающие в очереди `wqh` (например, заблокированные в `read` или `poll`), так как состояние дескриптора изменилось [](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0).
    

#### Мониторинг (`poll`, `select`, `epoll`)

Возможность использовать `eventfd` в циклах обработки событий — его ключевая особенность [](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://manpage.me/index.cgi?q=eventfd&sektion=2&apropos=0&manpath=Debian+8.1.0).  
Функция `eventfd_poll` реализует следующую логику, проверяя состояние счетчика под спин-локом [](https://www.kernel.dk/cgit/stable/commit/include/linux/eventfd.h?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0):

- **`POLLIN`**: Устанавливается, если `count > 0`. Это означает, что есть данные для чтения [](https://www.kernel.dk/cgit/stable/commit/include/linux/eventfd.h?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0).
    
- **`POLLOUT`**: Устанавливается, если значение `count` не превышает максимально допустимое для записи без блокировки (`UINT64_MAX - 1` > `count`). Фактически это означает, что в счетчик можно добавить, как минимум, 1 [](https://www.kernel.dk/cgit/stable/commit/include/linux/eventfd.h?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0).
    
- **`POLLERR`**: Устанавливается в исключительном случае, когда произошло переполнение счетчика. Это может случиться, если ядро вызывает функцию `eventfd_signal` (см. ниже) из атомарного контекста, которая не может блокироваться и, в целях продолжения работы, позволяет счетчику достичь `UINT64_MAX`. `read` в такой ситуации вернет значение `UINT64_MAX`, сигнализируя о переполнении [](https://www.kernel.dk/cgit/stable/commit/include/linux/eventfd.h?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://manpages.opensuse.org/Tumbleweed/man-pages/eventfd_write.3.en.html)[](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0).
    

### 4. Интерфейс для ядра: `eventfd_signal`

Основная мощь `eventfd` раскрывается при использовании его внутри ядра. Модули ядра (например, планировщики ввода-вывода, KVM, AIO) могут получить экземпляр `struct file`, соответствующий `eventfd`, и использовать функцию `eventfd_signal` для уведомления пользовательского пространства [](https://www.kernel.dk/cgit/stable/commit/include/linux/eventfd.h?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://lore.kernel.org/all/alpine.DEB.1.10.0906221132210.10952@makko.or.mcafeemobile.com/).

```c unfold
int eventfd_signal(struct file *file, int n);
```

- **Контекст**: Эта функция специально спроектирована для вызова из **любого контекста**, включая атомарный контекст и обработчики прерываний [](https://www.kernel.dk/cgit/stable/commit/include/linux/eventfd.h?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0). Она никогда не блокируется и не спит.
    
- **Механизм**: Функция увеличивает счетчик `count` на значение `n` под защитой спин-лока (`spin_lock_irqsave`), что безопасно в контексте прерывания.
    
- **Обработка переполнения**: В отличие от `write` из userspace, `eventfd_signal` не может блокироваться. Поэтому при попытке превысить максимальное значение (`UINT64_MAX`) она добавляет ровно столько, сколько может (`ULLONG_MAX - ctx->count`), устанавливая счетчик в `ULLONG_MAX`, и возвращает фактически добавленное значение (которое будет меньше запрошенного `n`). Это приводит к возникновению состояния переполнения, о котором сигнализирует флаг `POLLERR` [](https://www.kernel.dk/cgit/stable/commit/include/linux/eventfd.h?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0)[](https://www.kernel.dk/cgit/stable/commit/fs/eventfd.c?id=e1ad7468c77ddb94b0615d5f50fa255525fde0f0).
    
- **Пробуждение**: После изменения счетчика, если в очереди `wqh` есть ожидающие процессы, они будут пробуждены вызовом `wake_up_locked(&ctx->wqh)`.
    

### Заключение

Механизм `eventfd` предоставляет элегантное и эффективное решение для уведомления о событиях. Его реализация в ядре Linux основана на простой структуре данных (`eventfd_ctx`) с 64-битным счетчиком и минимальным набором операций, защищенных спин-локом. Благодаря интеграции с файловой моделью VFS, он бесшовно работает со стандартными механизмами мультиплексирования ввода-вывода (`epoll`, `poll`) и предоставляет как пользовательский API для межпроцессного взаимодействия, так и kernel-интерфейс (`eventfd_signal`) для асинхронного уведомления из различных контекстов ядра, включая атомарный.

## Примеры

### Пример 1: Базовая сигнализация между процессами (потоками)

Это самый простой случай использования `eventfd`, когда один поток (или процесс) ожидает уведомления от другого. Поток-писатель сигнализирует о наступлении события, увеличивая внутренний счетчик. Поток-читатель, заблокированный в `read`, просыпается и обрабатывает событие. Этот пример наглядно демонстрирует основную атомарную природу операций с 64-битным счетчиком [](https://cloud.tencent.cn/developer/article/2438981?from=15425&frompage=seopage).

```c
#include <sys/eventfd.h>
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <inttypes.h>
#define handle_error(msg) \
    do { perror(msg); exit(EXIT_FAILURE); } while (0)
    
int efd;

static void *writer_thread(void *arg) {
    uint64_t u = 1; // Значение, которое будет добавлено к счетчику
    for (int i = 0; i < 3; i++) {
        sleep(1); // Имитируем подготовку события
        printf("writer: writing...\n");
        
        ssize_t s = write(efd, &u, sizeof(uint64_t));
        if (s != sizeof(uint64_t))
            handle_error("write");
    }
    return NULL;
}

static void *reader_thread(void *arg) {
    uint64_t u;
    for (int i = 0; i < 3; i++) {
        printf("reader: waiting...\n");
        
        ssize_t s = read(efd, &u, sizeof(uint64_t));
        if (s != sizeof(uint64_t))
            handle_error("read");
        printf("reader: got value %" PRIu64 "\n", u);
    }
    return NULL;
}
int main() {
    pthread_t rd, wr;
    // Создаем eventfd с начальным счетчиком 0, без флагов (блокирующий режим)
    efd = eventfd(0, 0);
    if (efd == -1)
        handle_error("eventfd");
        
    pthread_create(&wr, NULL, writer_thread, NULL);
    pthread_create(&rd, NULL, reader_thread, NULL);
    
    pthread_join(wr, NULL);
    pthread_join(rd, NULL);
    
    close(efd);
    printf("All done.\n");
    exit(EXIT_SUCCESS);
}
```

**Технический разбор:**

1. **Инициализация:** Вызов `eventfd(0, 0)` создает новый объект eventfd. Внутри ядра создается структура `struct eventfd_ctx` со счетчиком (`count`), установленным в 0. Флаг `0` означает, что файловый дескриптор работает в блокирующем режиме, а операция `read` будет иметь стандартную семантику (чтение значения и сброс счетчика в ноль) [](https://man.archlinux.org/man/extra/man-pages-ru/eventfd.2.ru)[](https://manpages.debian.org/bullseye-backports/manpages-ru-dev/eventfd2.2.ru.html).
    
2. **Ожидание (Reader):** Поток-читатель вызывает `read()`. Так как счетчик равен 0, ядро переводит поток в состояние ожидания (`TASK_INTERRUPTIBLE`) и помещает его в очередь ожидания `wqh`, связанную с данным экземпляром `eventfd_ctx`. Планировщик больше не будет выделять ему процессорное время [](https://webkit.crisal.io/rust/rev/a52b1d6e29475037a677ceab7d5be5f682c90515/src/tools/miri/src/shims/unix/linux/eventfd.rs)[](https://cloud.tencent.cn/developer/article/2438981?from=15425&frompage=seopage).
    
3. **Сигнализация (Writer):** Поток-писатель вызывает `write()`, передавая значение `1`. Системный вызов `write` под защитой спин-блокировки (`spinlock`) атомарно добавляет 1 к текущему значению `count`, которое становится равным 1. После изменения счетчика ядро вызывает `wake_up_locked(&ctx->wqh)`, что приводит к пробуждению всех процессов (в данном случае одного), ожидающих в очереди [](https://webkit.crisal.io/rust/rev/a52b1d6e29475037a677ceab7d5be5f682c90515/src/tools/miri/src/shims/unix/linux/eventfd.rs)[](https://cloud.tencent.cn/developer/article/2438981?from=15425&frompage=seopage).
    
4. **Обработка (Reader):** Проснувшийся поток-читатель получает управление, и его вызов `read` завершается. Он читает текущее значение счетчика (1), копирует 8 байт (значение `1`) в буфер пользователя и, так как флаг `EFD_SEMAPHORE` не установлен, **сбрасывает счетчик обратно в 0**. Цикл повторяется.
    

---

### Пример 2: Интеграция с epoll (сервер событий)

Главная сила `eventfd` раскрывается при его использовании вместе с механизмами мультиплексирования ввода-вывода, такими как `epoll`. Это позволяет приложению в одном потоке обрабатывать как сетевые события, так и пользовательские сигналы. В этом примере главный поток ожидает события на `eventfd`, в то время как рабочий поток генерирует эти события [](https://cloud.tencent.com.cn/developer/information/eventfd)[](https://cloud.tencent.cn/developer/article/2438981?from=15425&frompage=seopage).

```c
#include <sys/eventfd.h>
#include <sys/epoll.h>
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <inttypes.h>
#define handle_error(msg) \
    do { perror(msg); exit(EXIT_FAILURE); } while (0)
    
int efd;

static void *worker_thread(void *arg) {
    uint64_t u = 1;
    for (int i = 0; i < 5; ++i) {
        sleep(1);
        printf("worker: sending event %d\n", i+1);
        if (write(efd, &u, sizeof(u)) != sizeof(u))
            handle_error("write");
    }
    return NULL;
}

int main() {
    int epfd;
    struct epoll_event ev, events[10];
    // Создаем eventfd в неблокирующем режиме, что рекомендуется при использовании с epoll
    efd = eventfd(0, EFD_NONBLOCK);
    if (efd == -1)
        handle_error("eventfd");
        
    // Создаем epoll-инстанс
    epfd = epoll_create1(0);
    if (epfd == -1)
        handle_error("epoll_create1");
        
    // Добавляем eventfd в набор отслеживаемых дескрипторов epoll
    ev.events = EPOLLIN; // Ждем только событий чтения
    ev.data.fd = efd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, efd, &ev) == -1)
        handle_error("epoll_ctl");
        
    pthread_t worker;
    pthread_create(&worker, NULL, worker_thread, NULL);
    printf("main: waiting for events...\n");
    while (1) {
        // Блокируемся до появления события на любом из отслеживаемых fd
        int nfds = epoll_wait(epfd, events, 10, -1);
        if (nfds == -1)
            handle_error("epoll_wait");
            
        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == efd) {
                uint64_t u;
                // Неблокирующее чтение. Если бы мы пропустили проверку POLLIN,
                // EPOLLET мог бы привести к блокировке. С EPOLLIN мы знаем, что данные есть.
                ssize_t s = read(efd, &u, sizeof(u));
                if (s == sizeof(u)) {
                    printf("main: event received! count: %" PRIu64 "\n", u);
                } else if (s == -1) {
                    // В неблокирующем режиме, если бы данных не было, вернулся бы EAGAIN.
                    // Но благодаря epoll мы знаем, что данные есть, поэтому это настоящая ошибка.
                    handle_error("read");
                }
            }
        }
    }
    pthread_join(worker, NULL);
    close(epfd);
    close(efd);
    return 0;
}
```

**Технический разбор:**

1. **Настройка мониторинга:** `epoll_ctl` регистрирует `efd` в интересах ядра. Функция `eventfd_poll` из `fs/eventfd.c` будет вызываться ядром для определения текущего состояния дескриптора. `efd` добавлен с флагом `EPOLLIN` [](https://cloud.tencent.com.cn/developer/information/eventfd)[](https://cloud.tencent.cn/developer/article/2438981?from=15425&frompage=seopage).
    
2. **Ожидание:** Главный поток вызывает `epoll_wait` и блокируется. Поток ядра, обслуживающий `epoll`, переводит его в сон.
    
3. **Сигнализация и пробуждение:** Рабочий поток выполняет `write` на `eventfd`. Как и в первом примере, ядро увеличивает счетчик и пробуждает процессы из очереди ожидания _самого `eventfd`_.
    
4. **Каскадное пробуждение:** Функция `eventfd_poll` (или внутренние механизмы `epoll`) определяет, что состояние дескриптора изменилось (теперь `count > 0`, значит, доступны данные для чтения). Это приводит к тому, что главный поток, ожидающий в `epoll_wait`, также пробуждается, и `epoll_wait` возвращает управление с указанием того, что `efd` готов для чтения [](https://cloud.tencent.com.cn/developer/information/eventfd)[](https://cloud.tencent.cn/developer/article/2438981?from=15425&frompage=seopage).
    
5. **Обработка:** Главный поток вызывает `read`. Поскольку дескриптор создан с флагом `EFD_NONBLOCK`, вызов `read` гарантированно не заблокируется (мы знаем, что данные есть) и считает значение, сбросив счетчик в 0.
    

---

### Пример 3: Счетчик событий с семафорной семантикой (EFD_SEMAPHORE)

Этот пример демонстрирует использование флага `EFD_SEMAPHORE`. Он полезен, когда нужно точно учитывать количество поступивших сигналов, а не просто получать факт их наличия. Каждый успешный вызов `read` потребляет ровно одну единицу из счетчика, даже если за один раз было записано большое число. Это классический паттерн "производитель-потребитель" для подсчета ресурсов или задач [](https://man.archlinux.org/man/extra/man-pages-ru/eventfd.2.ru)[](https://manpages.debian.org/bullseye-backports/manpages-ru-dev/eventfd2.2.ru.html).

```c
#include <sys/eventfd.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <inttypes.h>
#include <pthread.h>
#define handle_error(msg) \
    do { perror(msg); exit(EXIT_FAILURE); } while (0)
    
int efd;

void *producer(void *arg) {
    uint64_t u;
    for (int i = 1; i <= 3; i++) {
        sleep(1);
        // Производим 2, 3 и 4 "задачи" за раз
        u = i + 1;
        
        printf("Producer: adding %" PRIu64 " tasks\n", u);
        if (write(efd, &u, sizeof(u)) != sizeof(u))
            handle_error("write");
    }
    return NULL;
}
void *consumer(void *arg) {
    uint64_t u;
    for (int i = 0; i < 9; i++) { // Ожидаем суммарно 2+3+4 = 9 событий
        printf("Consumer: waiting for a task...\n");
        // read вернет 1, а счетчик уменьшится на 1
        if (read(efd, &u, sizeof(u)) != sizeof(u))
            handle_error("read");
        printf("Consumer: processing task (token: %" PRIu64 ")\n", u);
        sleep(2); // Имитируем обработку задачи
    }
    return NULL;
}
int main() {
    // Создаем eventfd с флагом EFD_SEMAPHORE
    efd = eventfd(0, EFD_SEMAPHORE);
    if (efd == -1)
        handle_error("eventfd");
        
    pthread_t prod, cons;
    pthread_create(&prod, NULL, producer, NULL);
    pthread_create(&cons, NULL, consumer, NULL);
    
    pthread_join(prod, NULL);
    pthread_join(cons, NULL);
    
    close(efd);
    return 0;
}
```

**Технический разбор:**

1. **Инициализация:** `eventfd(0, EFD_SEMAPHORE)`. Флаг `EFD_SEMAPHORE` сохраняется в структуре файла и коренным образом меняет поведение системного вызова `read` для этого файлового дескриптора [](https://man.archlinux.org/man/extra/man-pages-ru/eventfd.2.ru)[](https://manpages.debian.org/bullseye-backports/manpages-ru-dev/eventfd2.2.ru.html).
    
2. **Производство:** Поток-производитель трижды вызывает `write`, передавая значения 2, 3 и 4. Счетчик внутри `eventfd_ctx` становится равен `0 + 2 + 3 + 4 = 9`.
    
3. **Потребление (семафорная семантика):** Поток-потребитель входит в цикл на 9 итераций. При первом вызове `read` ядро проверяет флаг `EFD_SEMAPHORE`. Видя, что он установлен, и счетчик `count` равен 9 (>0), оно выполняет следующие атомарные действия:
    
    - Копирует в пользовательский буфер значение **1**, а не 9.
        
    - Уменьшает внутренний счетчик на 1 (`count` становится равным 8).  
        Возвращается из системного вызова [](https://man.archlinux.org/man/extra/man-pages-ru/eventfd.2.ru)[](https://manpages.debian.org/bullseye-backports/manpages-ru-dev/eventfd2.2.ru.html).
        
4. **Результат:** Несмотря на то, что за одну запись было добавлено несколько "событий", потребитель обрабатывает их по одному, делая 9 отдельных циклов. Если бы флаг `EFD_SEMAPHORE` отсутствовал, первый же `read` считал бы число 9, сбросил счетчик в 0, и потребитель "потерял" бы информацию о том, что нужно обработать 9 задач, а не одну.
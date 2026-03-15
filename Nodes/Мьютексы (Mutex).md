---




---
---
* [[Синхронизация (User Space)|Назад]]
---
### 1. Определение и состояние мьютекса (Mutex)

**Мьютекс** (MUTual EXclusion) — это примитив синхронизации, предназначенный для защиты разделяемых данных от одновременных изменений и реализации критических секций [](https://manpages.debian.org/bookworm/glibc-doc/pthread_mutex_init.3.en.html)[](https://man7.org/linux/man-pages/man3/pthread_mutex_unlock.3.html). С технической точки зрения, мьютекс — это объект, который может находиться в одном из двух состояний [](https://manpages.debian.org/bookworm/glibc-doc/pthread_mutex_init.3.en.html)[](https://man7.org/linux/man-pages/man3/pthread_mutex_unlock.3.html):

- **Разблокирован (0)**: Не принадлежит ни одному потоку.
    
- **Заблокирован (1)**: Принадлежит ровно одному потоку. Мьютекс никогда не может принадлежать двум разным потокам одновременно.
    

### 2. Критическая секция (Critical Section)

В теории синхронизации **критическая секция** — это участок кода, который гарантированно может выполняться только одним потоком в конкретный момент времени для обеспечения корректности программы [](https://github.com/angrave/SystemProgramming/wiki/Synchronization,-Part-1:-Mutex-Locks#solving-critical-sections). Доступ к разделяемым данным (например, глобальной переменной) должен быть ограничен критической секцией. Мьютексы используются для реализации этих секций: код, расположенный между вызовами `pthread_mutex_lock()` и `pthread_mutex_unlock()`, образует критическую секцию [](https://manpages.debian.org/bookworm/glibc-doc/pthread_mutex_init.3.en.html)[](https://man7.org/linux/man-pages/man3/pthread_mutex_unlock.3.html)[](http://china.qnx.com/developers/docs/7.1/com.qnx.doc.neutrino.sys_arch/topic/kernel_Mutexes.html).

Важно отметить: доступ к самой памяти внутри критической секции не блокируется на аппаратном уровне. Код по-прежнему доступен для чтения всеми потоками, но попытка выполнить его приведет к ожиданию на захвате мьютекса [](https://github.com/angrave/SystemProgramming/wiki/Synchronization,-Part-1:-Mutex-Locks#solving-critical-sections).

### 3. Механизм работы (Flow of Execution)

Работа с мьютексами осуществляется через операции `lock` (захват) и `unlock` (освобождение). Согласно спецификациям POSIX [](https://manpages.debian.org/bookworm/glibc-doc/pthread_mutex_init.3.en.html)[](https://man7.org/linux/man-pages/man3/pthread_mutex_unlock.3.html):

1. Поток, вызывающий `pthread_mutex_lock()` для разблокированного мьютекса, становится его владельцем, и функция возвращает управление немедленно.
    
2. Если мьютекс уже заблокирован другим потоком, вызывающий поток переводится в состояние ожидания (suspend) до тех пор, пока мьютекс не будет освобожден.
    
3. Поток, владеющий мьютексом, вызывает `pthread_mutex_unlock()`, чтобы освободить его. Если на момент освобождения существуют другие потоки, заблокированные на этом мьютексе, планировщик определяет, какой из них получит мьютекс и продолжит выполнение [](https://browse.library.kiwix.org/content/stackoverflow.com_en_all_nopic_2022-07/questions/16889063/pthread-mutex-pthread-mutex-unlock-consumes-lots-of-time)[](https://www.ibm.com/docs/sl/aix/7.2.0?topic=p-pthread-mutex-lock-pthread-mutex-trylock-pthread-mutex-unlock-subroutine).
    
4. Порядок, в котором потоки захватывают мьютекс, не детерминирован (race condition на захват) [](https://www.ibm.com/docs/sl/aix/7.2.0?topic=p-pthread-mutex-lock-pthread-mutex-trylock-pthread-mutex-unlock-subroutine).
    

### 4. Фундаментальные свойства и ограничения

- **Владение (Ownership)**: Мьютекс имеет строгую концепцию владельца. Только поток, захвативший мьютекс, может его освободить (за исключением особых типов мьютексов, таких как "fast", но это непереносимое поведение) [](https://manpages.debian.org/bookworm/glibc-doc/pthread_mutex_init.3.en.html)[](https://github.com/angrave/SystemProgramming/wiki/Synchronization,-Part-1:-Mutex-Locks#solving-critical-sections)[](https://man7.org/linux/man-pages/man3/pthread_mutex_unlock.3.html).
    
- **Deadlock (Взаимоблокировка)**:
    
    - **Аварийное завершение**: Если поток, владеющий мьютексом, аварийно завершается, не освободив его, мьютекс остается в заблокированном состоянии. В результате все остальные потоки, ожидающие на `pthread_mutex_lock()`, блокируются навсегда. Это состояние называется взаимоблокировкой (deadlock) [](https://git.kernel.org/pub/scm/docs/man-pages/man-pages.git/tree/man3/pthread_mutex_consistent.3?id=5fbde956cb550ffeae83c31e4f8c1142544f4b4f). Для решения этой проблемы существуют "надежные" (robust) мьютексы, которые возвращают ошибку `EOWNERDEAD` следующему потоку, пытающемуся его захватить [](https://www.ibm.com/docs/sl/aix/7.2.0?topic=p-pthread-mutex-lock-pthread-mutex-trylock-pthread-mutex-unlock-subroutine)[](https://git.kernel.org/pub/scm/docs/man-pages/man-pages.git/tree/man3/pthread_mutex_consistent.3?id=5fbde956cb550ffeae83c31e4f8c1142544f4b4f).
        
    - **Множественные блокировки**: Deadlock также возникает, если два потока пытаются захватить два разных мьютекса в противоположном порядке [](https://docs.oracle.com/cd/E19120-01/open.solaris/816-5137/6mba5vq13/index.html).
        

### 5. Инициализация и типы мьютексов

- **Статическая инициализация**: Для глобальных переменных используется макрос `PTHREAD_MUTEX_INITIALIZER`. Он создает мьютекс с атрибутами по умолчанию (тип "fast") [](https://manpages.debian.org/bookworm/glibc-doc/pthread_mutex_init.3.en.html)[](https://github.com/angrave/SystemProgramming/wiki/Synchronization,-Part-1:-Mutex-Locks#solving-critical-sections)[](https://man7.org/linux/man-pages/man3/pthread_mutex_unlock.3.html).
    
- **Динамическая инициализация**: Функция `pthread_mutex_init()` позволяет задать атрибуты мьютекса, такие как тип блокировки [](https://manpages.debian.org/bookworm/glibc-doc/pthread_mutex_init.3.en.html)[](https://man7.org/linux/man-pages/man3/pthread_mutex_unlock.3.html).
    
    - **fast (по умолчанию)**: Не проверяет повторные попытки захвата тем же потоком, что приводит к deadlock'у.
        
    - **recursive**: Позволяет одному потоку многократно захватывать один и тот же мьютекс (ведется счетчик захватов).
        
    - **error checking**: Выполняет проверки и возвращает ошибку `EDEADLK` при попытке повторного захвата [](https://manpages.debian.org/bookworm/glibc-doc/pthread_mutex_init.3.en.html)[](https://man7.org/linux/man-pages/man3/pthread_mutex_unlock.3.html).
        

### 6. Анализ производительности (Performance Overhead)

Использование мьютексов влечет за собой накладные расходы, что видно на примере предоставленного кода.

```c
#include <stdio.h>
#include <pthread.h>
#define N 5
long a = 0;
// Статическая инициализация мьютекса с атрибутами по умолчанию
pthread_mutex_t m1 = PTHREAD_MUTEX_INITIALIZER;
void* thread_calc(void* args)
{
    // В оригинале была ошибка: int i, tmp; (int i переопределялся)
    // Аргумент не используется, но в примере он был, поэтому оставим сигнатуру.
    // Для простоты и корректности уберем неиспользуемые переменные.
    for (int i = 0; i < 10000000; i++) {
        pthread_mutex_lock(&m1); // Вход в критическую секцию
        a = a + 1;                // Работа с разделяемыми данными
        pthread_mutex_unlock(&m1); // Выход из критической секции
    }
    return NULL;
}
int main(void)
{
    pthread_t thread[N];
    // Создание потоков
    for (int i = 0; i < N; i++) {
        pthread_create(&thread[i], NULL, thread_calc, NULL);
    }
    // Ожидание завершения потоков
    for (int i = 0; i < N; i++) {
        pthread_join(thread[i], NULL);
    }
    printf("a = %ld\n", a); // Ожидаем 5 * 10 000 000 = 50 000 000
    return 0;
}
```

**Причины падения производительности:**

1. **Сериализация выполнения**: Хотя потоки существуют, они не могут параллельно выполнять код внутри критической секции. Инкремент переменной `a` выполняется строго последовательно, что сводит на нет преимущества параллелизма для этого участка кода [](https://github.com/angrave/SystemProgramming/wiki/Synchronization,-Part-1:-Mutex-Locks#solving-critical-sections).
    
2. **Накладные расходы на системные вызовы**: При высокой конкуренции (contention) за мьютекс, операция `pthread_mutex_unlock()` может быть дорогостоящей, так как она должна определить, есть ли ожидающие потоки, и инициировать переключение контекста для их пробуждения [](https://browse.library.kiwix.org/content/stackoverflow.com_en_all_nopic_2022-07/questions/16889063/pthread-mutex-pthread-mutex-unlock-consumes-lots-of-time)[](http://china.qnx.com/developers/docs/7.1/com.qnx.doc.neutrino.sys_arch/topic/kernel_Mutexes.html). Если мьютексы реализованы через объекты ядра (как в некоторых реализациях), каждая операции lock/unlock может требовать перехода в режим ядра, что занимает сотни тактов [](https://staging.gamedeveloper.com/programming/programming-multi-threaded-architectures---mutex-objects-and-critical-sections).
    
3. **Отсутствие реальной параллельности**: В данном конкретном примере потоки соревнуются за доступ к одной переменной. Использование мьютекса убирает состояние гонки, но ценой практически полной потери параллельности для этой операции.
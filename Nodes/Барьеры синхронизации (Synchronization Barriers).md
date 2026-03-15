---




---
---
* [[Синхронизация (User Space)|Назад]]
---
Барьер (barrier) — это примитив синхронизации, предназначенный для координации группы потоков выполнения. Его основная задача — заставить потоки остановиться в заданной точке кода и ожидать друг друга, пока все участники не достигнут этого барьера. Только после сбора полной "команды" потоки разблокируются и продолжают работу параллельно [](http://www.qnx.com/developers/docs/7.1/com.qnx.doc.neutrino.sys_arch/topic/kernel_Barriers.html).

Ключевое отличие барьера от `pthread_join()` заключается в том, что `pthread_join()` ожидает завершения потока, тогда как барьер организует рандеву (встречу) потоков в определённой фазе их выполнения, не требуя их терминации [](http://www.qnx.com/developers/docs/7.1/com.qnx.doc.neutrino.sys_arch/topic/kernel_Barriers.html). Это особенно полезно для реализации циклических параллельных вычислений, где несколько потоков выполняют итерации и должны синхронизироваться в конце каждой итерации.

### Программный интерфейс (API)

В стандарте POSIX барьеры реализованы следующим набором функций:

- **`pthread_barrier_init(pthread_barrier_t *barrier, const pthread_barrierattr_t *attr, unsigned int count)`** — инициализирует барьер. Аргумент `count` задаёт количество потоков, которые должны вызвать `pthread_barrier_wait()`, чтобы барьер был пройден [](https://man7.org/tlpi/code/online/dist/threads/pthread_barrier_demo.c.html)[](http://www.qnx.com/developers/docs/7.1/com.qnx.doc.neutrino.sys_arch/topic/kernel_Barriers.html).
    
- **`pthread_barrier_wait(pthread_barrier_t *barrier)`** — вызывается потоком, который достиг барьера. Если барьер ещё не собрал необходимое количество потоков, вызвавший поток переводится в состояние ожидания (сна). Когда последний, `count`-й поток вызывает эту функцию, все ожидающие потоки пробуждаются и продолжают выполнение [](https://man7.org/tlpi/code/online/dist/threads/pthread_barrier_demo.c.html)[](http://www.qnx.com/developers/docs/7.1/com.qnx.doc.neutrino.sys_arch/topic/kernel_Barriers.html).
    
- **`pthread_barrier_destroy(pthread_barrier_t *barrier)`** — уничтожает барьер и освобождает связанные с ним ресурсы.
    

Особенностью функции `pthread_barrier_wait()` является её возвращаемое значение. Среди всех потоков, ожидавших на барьере, ровно один получает специальное значение `PTHREAD_BARRIER_SERIAL_THREAD`, а все остальные — `0`. Это позволяет выделить один поток для выполнения какого-либо однократного действия после прохождения барьера (например, обновления глобальных данных или вывода статистики) [](https://man7.org/tlpi/code/online/dist/threads/pthread_barrier_demo.c.html).

### Пример использования

В листинге ниже продемонстрирован типичный сценарий: несколько потоков выполняют некоторую работу (моделируется случайной задержкой), после чего синхронизируются на барьере. После прохождения барьера один из потоков (тот, который получил `PTHREAD_BARRIER_SERIAL_THREAD`) инициирует вывод разделителя для улучшения читаемости лога [](https://man7.org/tlpi/code/online/dist/threads/pthread_barrier_demo.c.html).

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static pthread_barrier_t barrier;
static int numBarriers; // Количество проходов через барьер

static void *threadFunc(void *arg) {
    long threadNum = (long) arg;
    
    for (int j = 0; j < numBarriers; j++) {
        int nsecs = rand() % 5 + 1;
        sleep(nsecs);

        printf("Поток %ld достиг барьера %d\n", threadNum, j);
        int s = pthread_barrier_wait(&barrier);
        if (s == PTHREAD_BARRIER_SERIAL_THREAD) {
            // Этот поток выполняет действие один раз после прохождения барьера
            usleep(100000);
            printf("\n--- Барьер %d пройден ---\n\n", j);
        }
    }
    return NULL;
}

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "Использование: %s <число_барьеров> <число_потоков>\n", argv[0]);
        return 1;
    }
    
    numBarriers = atoi(argv[1]);
    int numThreads = atoi(argv[2]);
    pthread_t *tid = calloc(numThreads, sizeof(pthread_t));
    
    // Инициализация барьера: нужно numThreads потоков для прохода
    pthread_barrier_init(&barrier, NULL, numThreads);
    for (long i = 0; i < numThreads; i++) {
        pthread_create(&tid[i], NULL, threadFunc, (void *) i);
    }
    
    for (int i = 0; i < numThreads; i++) {
        pthread_join(tid[i], NULL);
    }
    
    pthread_barrier_destroy(&barrier);
    free(tid);
    return 0;
}

```
### Области применения

Основная задача барьеров — обеспечение строгой синхронизации фаз выполнения в многопоточных приложениях. Они незаменимы, когда алгоритм требует, чтобы все потоки завершили текущий этап вычислений прежде, чем какой-либо из них начнёт следующий [](https://lore-kernel.gnuweeb.org/ltp/20250829095635.193116-1-liwang@redhat.com/#r).

Типичные случаи использования:

- **Параллельные численные методы**: решение систем линейных уравнений, матричные операции, где на каждой итерации требуются данные от всех потоков.
    
- **Тестирование и симуляция**: обеспечение детерминированного старта всех потоков, чтобы избежать состояний гонки на этапе инициализации. Например, в тесте `sched_football` из набора LTP использование барьера позволило гарантировать, что все "игроки" (потоки) начнут движение одновременно, что устранило нестабильность тестов
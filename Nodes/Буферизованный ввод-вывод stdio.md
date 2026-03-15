---




---
---
* [[Linux|Назад]]
---
## Определение и архитектура

**Буферизированный ввод-вывод** (стандартная библиотека `stdio`) — это обёртка над системными вызовами (open, read, write), предоставляющая промежуточные буферы в пространстве пользователя для оптимизации операций с данными [](http://rpm.pbone.net/manpage_idpl_97957952_numer_3_nazwa_stdio.html). Библиотека является частью libc и автоматически компонуется при сборке [](http://rpm.pbone.net/manpage_idpl_97957952_numer_3_nazwa_stdio.html).

### Многоуровневая архитектура данных
![[Pasted image 20260315110141.png|587]]
При использовании функций семейства `fopen`/`fread`/`fwrite` данные проходят следующий путь:

1. **Буфер библиотеки (пространство пользователя, уровень stdio)** — первый уровень накопления
    
2. **Буферы ядра (page cache, буфер грязных страниц)** — кеширование на уровне операционной системы
    
3. **Устройство хранения (ПЗУ/блочное устройство)** — физическая запись
    

Этот путь работает в обоих направлениях (чтение/запись). Управление синхронизацией между буфером библиотеки и буферами ядра берёт на себя библиотека stdio [](http://rpm.pbone.net/manpage_idpl_97957952_numer_3_nazwa_stdio.html).

## Назначение и эффективность

Буферизированный ввод-вывод оптимизирован для **работы с большими объёмами данных** (потоковые данные: аудио, видео, сетевые пакеты) [](http://uw714doc.xinuos.com/en/man/html.3S/Intro.3S.html).

**Почему это эффективно:**

- Накопление данных в буфере библиотеки до размера, кратного блоку файловой системы
    
- Одна операция записи в ядро вместо множества мелких
    
- Размер буфера часто совпадает с размером блока ФС (`st_blksize`), что позволяет выполнить запись за одну операцию [](http://aren.cs.ui.ac.id/sda/archive/resources/contest/OnlineJudge/gnudoc/libc/Controlling_Buffering.html)[](https://ru.cppreference.net/c/io/setvbuf.html)
    
- Уменьшение количества переключений контекста (syscall overhead)
    

**Когда буферизация проигрывает:** при операциях с малыми объёмами данных накладные расходы на управление буфером могут превысить выигрыш.

## Режимы буферизации

Библиотека stdio поддерживает три режима, управляемые через `setvbuf` [](https://manpages.debian.org/bookworm-backports/manpages-ru-dev/setbuffer.3.ru.html)[](http://aren.cs.ui.ac.id/sda/archive/resources/contest/OnlineJudge/gnudoc/libc/Controlling_Buffering.html)[](https://ru.cppreference.net/c/io/setvbuf.html):

|Режим|Макрос|Поведение|
|---|---|---|
|**Построчная**|`_IOLBF`|Сброс буфера при символе `'\n'`, заполнении буфера или запросе ввода из терминального устройства|
|**Поблочная (полная)**|`_IOFBF`|Сброс только при заполнении буфера или явном вызове `fflush`|
|**Небуферизированный**|`_IONBF`|Немедленная запись (информация попадает в место назначения сразу)|

### Стандартные потоки и умолчания [](http://rpm.pbone.net/manpage_idpl_97957952_numer_3_nazwa_stdio.html)[](https://manpages.debian.org/bookworm-backports/manpages-ru-dev/setbuffer.3.ru.html)[](http://uw714doc.xinuos.com/en/man/html.3S/Intro.3S.html)

- **stdout**:
    
    - При связи с терминалом — построчная буферизация
        
    - При перенаправлении в файл — поблочная буферизация
        
- **stdin**: полностью буферизируется, если не связан с интерактивным устройством
    
- **stderr**: по умолчанию небуферизированный (для немедленного вывода ошибок)
    

### Особенности работы

**Построчная буферизация** (`_IOLBF`):

```c unfold
printf("Это появится после newline\n"); // Сброс по '\n'
printf("Это зависнет в буфере");        // Не появится без fflush
fflush(stdout);                          // Принудительный сброс
```

**Поблочная буферизация** (`_IOFBF`):

- Размер буфера обычно равен размеру блока ФС (можно получить через `fstat().st_blksize`) [](http://aren.cs.ui.ac.id/sda/archive/resources/contest/OnlineJudge/gnudoc/libc/Controlling_Buffering.html)[](https://ru.cppreference.net/c/io/setvbuf.html)
    
- Данные синхронизируются при заполнении буфера или завершении программы (через `exit()`)
    
- Явный сброс: `fflush()` (только в буфер ядра, не на диск)
    
- Полная синхронизация с диском: системный вызов `sync()` (глобально) или `fsync()` (для конкретного файла)
    

**Важно:** При перенаправлении вывода в файл `'\n'` перестаёт вызывать автоматический сброс, так как режим меняется на поблочный [](http://rpm.pbone.net/manpage_idpl_97957952_numer_3_nazwa_stdio.html).

## Основные функции API

### Открытие/закрытие потоков

```c unfold
#include <stdio.h>
FILE* fopen(const char* path, const char* mode);   // Открытие по имени
FILE* fdopen(int fd, const char* mode);            // Преобразование дескриптора в поток
int fileno(FILE* stream);                           // Получение дескриптора из потока
int fclose(FILE* stream);                           // Закрытие (с автоматическим сбросом)
```
### Чтение/запись

```c unfold
size_t fread(void* ptr, size_t size, size_t nmemb, FILE* stream);
size_t fwrite(const void* ptr, size_t size, size_t nmemb, FILE* stream);
```

Параметры: `ptr` — буфер, `size` — размер элемента, `nmemb` — количество элементов [](https://cs-uob.github.io/COMSM0085/exercises/part1/posix4/c_io.html).

### Управление буферизацией

```c unfold
int fflush(FILE* stream);              // Сброс буфера библиотеки в ядро
int fpurge(FILE* stream);               // Очистка буфера (обычно признак проблемного кода)
int setvbuf(FILE* restrict stream, char* restrict buf, 
            int mode, size_t size);     // Установка режима и буфера
void setbuf(FILE* stream, char* buf);   // Упрощённая версия (setvbuf с BUFSIZ)
void setbuffer(FILE* stream, char* buf, size_t size); // BSD-расширение
void setlinebuf(FILE* stream);           // Переключение на построчный режим
```

**Правила использования `setvbuf`** [](https://manpages.debian.org/bookworm-backports/manpages-ru-dev/setbuffer.3.ru.html)[](https://ru.cppreference.net/c/io/setvbuf.html)[](https://man.omnios.org/man3c/setvbuf):

- Вызывать **после открытия**, но **до любых операций** чтения/записи
    
- Если `buf == NULL`, библиотека сама выделит буфер через `malloc`
    
- Если `buf` предоставлен пользователем, он должен существовать до закрытия потока
    
- Нельзя использовать автоматические (локальные) массивы, если поток не закрыт до выхода из блока
    

### Позиционирование

```c unfold
int fseek(FILE* stream, long offset, int whence);  // Смена позиции
long ftell(FILE* stream);                            // Текущая позиция
void rewind(FILE* stream);                            // В начало файла
```

**Предупреждение:** Использование `fseek` с буферизированными потоками может привести к потере данных в буфере [](https://cs-uob.github.io/COMSM0085/exercises/part1/posix4/c_io.html).

### Макросы и константы [](https://manpages.debian.org/bookworm-backports/manpages-ru-dev/setbuffer.3.ru.html)[](http://aren.cs.ui.ac.id/sda/archive/resources/contest/OnlineJudge/gnudoc/libc/Controlling_Buffering.html)[](https://ru.cppreference.net/c/io/setvbuf.html)

|Константа|Значение|
|---|---|
|`BUFSIZ`|Размер буфера по умолчанию (минимум 256 байт)|
|`EOF`|Признак конца файла (-1)|
|`_IOFBF`|Режим полной буферизации|
|`_IOLBF`|Режим построчной буферизации|
|`IONBF`|Небуферизированный режим|
|`SEEK_SET`/`SEEK_CUR`/`SEEK_END`|Откуда отсчитывать смещение|

## Буфер упреждающего чтения (read-ahead) и управление памятью

### Связь с mmap

Буфер упреждающего чтения — механизм ядра, предсказывающий последовательный доступ к файлу и подгружающий страницы заранее. Его политика распространяется на:

- Файловый ввод-вывод
    
- Области `mmap` (включая динамические библиотеки)
    
- Большие выделения `malloc` (через `mmap`)
    

### Управление через posix_madvise

```c unfold
int posix_madvise(void* addr, size_t len, int advice);
```

**Параметры `advice`** (подсказки ядру):

- `POSIX_MADV_NORMAL` — обычный доступ (упреждающее чтение умеренного размера)
    
- `POSIX_MADV_SEQUENTIAL` — ожидается последовательный доступ (увеличить упреждающее чтение)
    
- `POSIX_MADV_RANDOM` — ожидается случайный доступ (отключить упреждающее чтение)
    
- `POSIX_MADV_WILLNEED` — скоро понадобится (инициировать упреждающее чтение сейчас)
    
- `POSIX_MADV_DONTNEED` — больше не нужно (освободить страницы)
    

### Пример с NUMA-оптимизацией

Рассмотрим систему с двумя процессорами, каждый со своей локальной памятью (NUMA):

```c unfold
// Большое выделение памяти (попадает в mmap)
void* ptr = malloc(LARGE_SIZE);
// По умолчанию: буфер упреждающего чтения выделит все страницы
// в памяти первого процессора, обратившегося к данным
// Если оба процессора будут обрабатывать свои части:
// Процессор 1 ходит в свою память (быстро)
// Процессор 2 ходит в память процессора 1 через медленный интерконнект
// Решение: указать случайный доступ
posix_madvise(ptr, LARGE_SIZE, POSIX_MADV_RANDOM);
// Теперь страницы выделяются по мере обращения:
// Когда процессор 1 обращается к своей странице — страница в его памяти
// Когда процессор 2 обращается к своей странице — страница в его памяти
```

**Техническое обоснование:** При случайном доступе отключается упреждающее выделение страниц. Страницы выделяются при первом обращении (page fault) на том процессоре, который сгенерировал обращение, что обеспечивает локальность данных [](https://www.eklektix.com/Articles/862704/).

## Опасные практики и подводные камни

1. **Автоматический буфер в стеке** [](https://manpages.debian.org/bookworm-backports/manpages-ru-dev/setbuffer.3.ru.html)[](https://ru.cppreference.net/c/io/setvbuf.html):
    
```c unfold
  int main() {
        char buf[BUFSIZ];
        setbuf(stdout, buf); // ОШИБКА: buf разрушится при выходе из main
        printf("Hello\n");
    } // Неопределённое поведение (буфер уже не существует)
```
    
2. **Смешивание дескрипторов и указателей**:
    
    - Не используйте `read`/`write` с `FILE*`, если не синхронизировали позиции
        
    - Преобразуйте через `fileno`/`fdopen` для согласованной работы
        
3. **Предположения о '\n'**:
    
    - Работает только для терминальных потоков
        
    - При перенаправлении в файл не гарантирует сброс
        
4. **Использование fpurge**:
    
    - Очищает буфер без записи
        
    - Сигнал о необходимости пересмотреть архитектуру программы
        
5. **Игнорирование ошибок** [](https://cs-uob.github.io/COMSM0085/exercises/part1/posix4/c_io.html):
    
    - Всегда проверяйте возвращаемые значения `fread`/`fwrite`
        
    - Используйте `ferror()` и `feof()` для определения причины неполного чтения/записи
        

## Стандарты и совместимость

Функции stdio стандартизированы:

- **ISO C89/C99/C11**: базовый набор [](http://rpm.pbone.net/manpage_idpl_97957952_numer_3_nazwa_stdio.html)[](https://ru.cppreference.net/c/io/setvbuf.html)
    
- **POSIX.1-2001/2008**: расширения (fdopen, fileno, и др.) [](http://rpm.pbone.net/manpage_idpl_97957952_numer_3_nazwa_stdio.html)[](https://ipfs.io/ipfs/QmehSxmTPRCr85Xjgzjut6uWQihoTfqg9VVihJ892bmZCp/Getchar.html)
    

Поведение может незначительно отличаться на разных платформах (например, текстовый/бинарный режимы на Windows), но на POSIX-системах разницы между текстовым и бинарным режимами нет [](https://cs-uob.github.io/COMSM0085/exercises/part1/posix4/c_io.html).
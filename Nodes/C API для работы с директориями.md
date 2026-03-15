---




---
---
* [[Виртуальная файловая система|Назад]]
---
### Базовый интерфейс: потоковый обход директории

На низком уровне взаимодействие с директорией в Linux реализуется через механизм **потока директории (directory stream)**, аналогичный файловым потокам. Основная пара функций для этого — `opendir()` и `readdir()` [](http://yisuapi.yisu.com/ask/59505822.html)[](https://www.ibm.com/docs/zh/aix/7.3.0?topic=o-opendir-readdir-telldir-seekdir-rewinddir-closedir-opendir64-readdir64-telldir64-seekdir64-rewinddir64-closedir64-fdopendir-subroutine#opendir__a0909afc__title__1).

1. **Открытие потока**: `opendir(const char *name)`.  
    Это точка входа в API. Функция принимает строку — путь к директории — и при успешном выполнении возвращает указатель на непрозрачную структуру типа `DIR` [](https://www.ibm.com/docs/zh/aix/7.3.0?topic=o-opendir-readdir-telldir-seekdir-rewinddir-closedir-opendir64-readdir64-telldir64-seekdir64-rewinddir64-closedir64-fdopendir-subroutine#opendir__a0909afc__title__1). Эта структура представляет собой поток директории. Если произошла ошибка (например, отсутствуют права на чтение `EACCES` или директория не существует `ENOENT`), функция возвращает `NULL` [](https://www.ibm.com/docs/zh/aix/7.3.0?topic=o-opendir-readdir-telldir-seekdir-rewinddir-closedir-opendir64-readdir64-telldir64-seekdir64-rewinddir64-closedir64-fdopendir-subroutine#opendir__a0909afc__title__1).
    
2. **Чтение записей**: `struct dirent* readdir(DIR *dirp)`.  
    Для получения содержимого директории используется эта функция. Она возвращает указатель на структуру `struct dirent`, которая описывает **одну** запись (файл, поддиректорию, ссылку и т.д.) в открытом потоке `dirp` [](http://yisuapi.yisu.com/ask/59505822.html)[](https://www.ibm.com/docs/zh/aix/7.3.0?topic=o-opendir-readdir-telldir-seekdir-rewinddir-closedir-opendir64-readdir64-telldir64-seekdir64-rewinddir64-closedir64-fdopendir-subroutine#opendir__a0909afc__title__1). При каждом последующем вызове `readdir()` возвращается следующая запись, пока они не закончатся — в этом случае функция возвращает `NULL`. Важная техническая деталь, подтверждающая исходные данные: **порядок, в котором `readdir()` возвращает имена, не определен и зависит от реализации файловой системы**; он не сортируется и не гарантирует алфавитный порядок [](http://yisuapi.yisu.com/ask/59505822.html).
    
3. **Завершение работы**: `int closedir(DIR *dirp)`.  
    Обязательная функция для освобождения ресурсов, выделенных под поток директории, и закрытия связанного с ним файлового дескриптора. Успешное выполнение возвращает `0`, ошибка — `-1` [](https://www.ibm.com/docs/zh/aix/7.3.0?topic=o-opendir-readdir-telldir-seekdir-rewinddir-closedir-opendir64-readdir64-telldir64-seekdir64-rewinddir64-closedir64-fdopendir-subroutine#opendir__a0909afc__title__1). В исходных данных была опечатка (`closeDir`), корректное название — **`closedir`**.
    

### Высокоуровневый интерфейс: сканирование с фильтрацией и сортировкой

Для более сложных сценариев, когда требуется не просто прочитать все подряд, а отфильтровать и отсортировать список, в стандартной библиотеке C (libc) существует функция `scandir` [](https://manpages.debian.org/bullseye/manpages-fr-dev/scandir.3)[](https://man.archlinux.org/man/core/man-pages/scandir.3.en). Она инкапсулирует типичные действия: открытие, чтение, выделение памяти, фильтрацию, сортировку.

`int scandir(const char *dirp, struct dirent ***namelist, int (*filter)(const struct dirent *), int (*compar)(const struct dirent **, const struct dirent **));`

- **Назначение**: Сканирует директорию `dirp` и возвращает **массив указателей** на записи директории, прошедшие фильтр [](https://manpages.debian.org/bullseye/manpages-fr-dev/scandir.3)[](https://www.cs.auckland.ac.nz/references/unix/man/scandir.3.html).
    
- **Выходной параметр `namelist`**: Это указатель на область памяти, куда `scandir` поместит указатель на динамически выделенный массив. Сам массив состоит из указателей на структуры `struct dirent *`. Каждый элемент этого массива — отдельная запись. Освобождение памяти — ответственность вызывающего кода: сначала нужно пройтись по массиву и вызвать `free()` для каждого элемента (`namelist[n]`), а затем — для самого массива (`namelist`) [](https://man.archlinux.org/man/core/man-pages/scandir.3.en)[](https://www.cs.auckland.ac.nz/references/unix/man/scandir.3.html).
    
- **Механизм фильтрации**: Параметр `filter` — это указатель на функцию обратного вызова. `scandir` вызывает эту функцию для каждой записи в директории. Если функция-фильтр возвращает ненулевое значение (истина), запись включается в итоговый массив `namelist`. Если `filter` установлен в `NULL`, в массив попадают **все** записи, включая служебные `.` и `..` (хотя их обычно отфильтровывают вручную) [](https://manpages.debian.org/bullseye/manpages-fr-dev/scandir.3)[](https://man.archlinux.org/man/core/man-pages/scandir.3.en).
    
- **Механизм сортировки**: Параметр `compar` — это указатель на функцию сравнения, которая используется `qsort(3)` для упорядочивания записей в итоговом массиве.
    
    - Стандартная библиотека предоставляет готовые функции сравнения: `alphasort()` и `versionsort()` [](https://manpages.debian.org/bullseye/manpages-fr-dev/scandir.3)[](https://man.archlinux.org/man/core/man-pages/scandir.3.en).
        
    - `alphasort()` сортирует записи в алфавитном порядке, используя `strcoll(3)` (с учетом локали) [](https://manpages.debian.org/bullseye/manpages-fr-dev/scandir.3).
        
    - `versionsort()` использует `strverscmp(3)`, что позволяет корректно сортировать версионные имена (например, `file1`, `file2`, `file10`) [](https://manpages.debian.org/bullseye/manpages-fr-dev/scandir.3)[](https://man.archlinux.org/man/core/man-pages/scandir.3.en).
        
    - Разработчик может написать свою функцию сравнения для реализации любого нестандартного порядка сортировки, что полностью соответствует исходному запросу.
        

### Резюме

- `opendir`/`readdir`/`closedir` — это **потоковый** итератор для чтения содержимого директории "как есть", без какого-либо дополнительного управления данными, кроме последовательного перебора.
    
- `scandir` — это **функция-агент**, которая берет на себя всю рутину по сбору, фильтрации и сортировке записей, возвращая готовый отсортированный массив, но требующая ручного управления освобождением выделенной памяти.
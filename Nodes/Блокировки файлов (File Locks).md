---




---
---
* [[Синхронизация (User Space)|Назад]]
---
### 1. Фундаментальная концепция: рекомендательные (advisory) блокировки

Критически важным фактом является то, что все стандартные механизмы блокировки файлов в Unix-системах (включая Linux) являются **рекомендательными (advisory)** [](https://cloud.tencent.cn/developer/article/1868981?from=15425)[](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/ch12s07.html#understandlk-CHP-12-SECT-7.1)[](https://www.parallel.uran.ru/book/export/html/496). Это означает, что ядро не принуждает к соблюдению блокировок на уровне системных вызовов `read()` и `write()` [](https://snapshots.sourceware.org/glibc/trunk/2024-06-26_11-04_1719399841/manual/html_node/File-Locks.html#index-fcntl_002eh-6). Блокировка работает только в том случае, если все взаимодействующие процессы добровольно проверяют её перед доступом к данным, следуя своего рода "джентльменскому соглашению" [](https://precise.sao.ru/hq/sts/linux/book/bogatyrev_c_unix/gl_6_7.shtml#6_10). Процесс, не использующий блокировки, может беспрепятственно читать и писать в файл, игнорируя блокировки, установленные другими процессами, что может привести к повреждению данных [](https://cloud.tencent.cn/developer/article/1868981?from=15425)[](https://cloud.tencent.cn/developer/article/2347480).

В некоторых системах существует концепция **принудительных (mandatory) блокировок**, но в Linux она поддерживается ограниченно для определенных файловых систем и требует специальных флагов при монтировании (`-o mand`) и установки set-group-ID бита на файле, не являясь стандартным механизмом [](https://cloud.tencent.cn/developer/article/1868981?from=15425)[](https://cloud.tencent.cn/developer/information/linux%20fcntl%E9%94%81)[](https://www.parallel.uran.ru/book/export/html/496).

### 2. Основные программные интерфейсы (API) блокировок

В Linux исторически сложилось три основных подхода к блокировкам, которые различаются семантикой, областью действия и историей происхождения.

#### 2.1. flock (BSD)

Системный вызов `flock()` предоставляет самый простой уровень блокировки [](https://www.daemon-systems.org/man/flock.2.html)[](https://www.parallel.uran.ru/book/export/html/496).

- **Область действия (Granularity)**: Блокировка применяется ко **всему файлу целиком**. Невозможно заблокировать отдельный диапазон байт [](https://cloud.tencent.cn/developer/information/linux%20fcntl%E9%94%81)[](https://cloud.tencent.cn/developer/article/2347480).
    
- **Семантика**: Блокировка привязана к **открытому файловому описанию (Open File Description)**, а не к файловому дескриптору или процессу [](https://www.daemon-systems.org/man/flock.2.html). В структурах ядра `struct file` содержится информация о блокировке `flock`. Если сделать `dup()` файлового дескриптора или передать дескриптор через `fork()`, дочерний процесс будет использовать то же открытое файловое описание, и, следовательно, разделять одну блокировку `flock` с родителем. Однако если дочерний процесс явно снимет блокировку, родитель её потеряет [](https://www.daemon-systems.org/man/flock.2.html).
    
- **Типы блокировок**: Поддерживаются разделяемые (`LOCK_SH` — shared) и исключительные (`LOCK_EX` — exclusive) блокировки, а также неблокирующий флаг (`LOCK_NB`) [](https://www.daemon-systems.org/man/flock.2.html)[](https://www.parallel.uran.ru/book/export/html/496).
    
- **Совместимость**: Не работает с сетевыми файловыми системами (NFS) в стандартном режиме [](https://cloud.tencent.cn/developer/information/linux%20fcntl%E9%94%81)[](https://www.parallel.uran.ru/book/export/html/496).
    

#### 2.2. fcntl (POSIX)

Интерфейс `fcntl()` (команды `F_SETLK`, `F_SETLKW`, `F_GETLK`) предоставляет более тонкое управление и описан в стандарте POSIX [](https://precise.sao.ru/hq/sts/linux/book/bogatyrev_c_unix/gl_6_7.shtml#6_10)[](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/ch12s07.html#understandlk-CHP-12-SECT-7.1)[](https://snapshots.sourceware.org/glibc/trunk/2024-06-26_11-04_1719399841/manual/html_node/File-Locks.html#index-fcntl_002eh-6).

- **Область действия (Granularity)**: Поддерживает блокировку **произвольных диапазонов байт** в файле (record/byte-range locking) [](https://precise.sao.ru/hq/sts/linux/book/bogatyrev_c_unix/gl_6_7.shtml#6_10)[](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/ch12s07.html#understandlk-CHP-12-SECT-7.1)[](https://cloud.tencent.cn/developer/information/linux%20fcntl%E9%94%81). Это критически важно для баз данных и сложных приложений. Параметры блокируемого сегмента задаются через структуру `struct flock`, включающую поля `l_start`, `l_whence` и `l_len` (значение 0 означает "до конца файла") [](https://precise.sao.ru/hq/sts/linux/book/bogatyrev_c_unix/gl_6_7.shtml#6_10)[](https://www.parallel.uran.ru/book/export/html/496)[](https://snapshots.sourceware.org/glibc/trunk/2024-06-26_11-04_1719399841/manual/html_node/File-Locks.html#index-fcntl_002eh-6).
    
- **Семантика**: Блокировка привязана к **кортежу (процесс, индексный дескриптор)**. Она ассоциирована с `struct inode` и процессом [](https://snapshots.sourceware.org/glibc/trunk/2024-06-26_11-04_1719399841/manual/html_node/File-Locks.html#index-fcntl_002eh-6). Если процесс имеет несколько дескрипторов на один и тот же файл (например, после `dup` или `fork`), они будут использовать одну и ту же POSIX блокировку, так как `fork` создает новый процесс с тем же PID в контексте блокировок. Закрытие **любого** файлового дескриптора, ссылающегося на файл, может снять блокировку, установленную процессом для этого файла, что часто является источником ошибок [](https://snapshots.sourceware.org/glibc/trunk/2024-06-26_11-04_1719399841/manual/html_node/File-Locks.html#index-fcntl_002eh-6).
    
- **Типы блокировок**: Разделяемые (`F_RDLCK` — read lock) и исключительные (`F_WRLCK` — write lock), а также команда на разблокировку (`F_UNLCK`) [](https://precise.sao.ru/hq/sts/linux/book/bogatyrev_c_unix/gl_6_7.shtml#6_10)[](https://www.parallel.uran.ru/book/export/html/496)[](https://snapshots.sourceware.org/glibc/trunk/2024-06-26_11-04_1719399841/manual/html_node/File-Locks.html#index-fcntl_002eh-6).
    
- **Совместимость**: Поддерживается через NFS (хотя реализация зависит от версии NFS и демона `lockd`), что позволяет синхронизировать доступ к файлам в распределенных средах [](https://cloud.tencent.cn/developer/information/linux%20fcntl%E9%94%81).
    

#### 2.3. lockf

Функция `lockf` является, по сути, библиотечной оберткой над `fcntl` в Linux [](https://precise.sao.ru/hq/sts/linux/book/bogatyrev_c_unix/gl_6_7.shtml#6_10)[](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/ch12s07.html#understandlk-CHP-12-SECT-7.1)[](https://cloud.tencent.cn/developer/information/linux%20fcntl%E9%94%81). Она предоставляет упрощенный интерфейс для работы с POSIX-блокировками, оперируя от текущей позиции файлового указателя [](https://precise.sao.ru/hq/sts/linux/book/bogatyrev_c_unix/gl_6_7.shtml#6_10)[](https://www.parallel.uran.ru/book/export/html/496).

### 3. Взаимодействие и ограничения

Различные механизмы блокировки **не взаимодействуют друг с другом**. Это означает, что процесс, установивший блокировку через `flock`, не будет замечен процессом, пытающимся проверить блокировку через `fcntl`, и наоборот [](https://cloud.tencent.cn/developer/information/linux%20fcntl%E9%94%81)[](https://www.parallel.uran.ru/book/export/html/496)[](https://cloud.tencent.cn/developer/article/2347480). Это накладывает жесткое требование: в рамках одного приложения или набора взаимодействующих приложений необходимо использовать единый механизм.

Таблица совместимости блокировок `fcntl` описывается следующей логикой (при использовании `F_SETLKW`) [](https://precise.sao.ru/hq/sts/linux/book/bogatyrev_c_unix/gl_6_7.shtml#6_10):

- На чтение (`F_RDLCK`): если нет блокировки или уже есть другие блокировки на чтение — успех. Если есть блокировка на запись — ожидание.
    
- На запись (`F_WRLCK`): если есть любая блокировка (чтение или запись) — ожидание.
    

### 4. Нюансы реализации в ядре Linux

- **Структуры данных**: В ядре блокировки управляются через структуры, связанные с `struct inode` (для POSIX) и `struct file` (для flock). Недавние изменения в ядре (по состоянию на март 2024 года) разделили общую инфраструктуру для блокировок и аренды (leases) путем введения новой структуры `struct file_lock_core`, которая встраивается в `struct file_lock`, что позволяет работать с ними через общий код, но хранить раздельно [](http://git.linux-nfs.org/?p=trondmy/linux-nfs.git;a=commitdiff;h=0c750012e8f30d26930ae13e815635258aee92b3). Они поддерживают очереди ожидающих процессов (поля типа `fl_block`) [](https://gitlab.sdu.dk/sdurobotics/linux-kernels/kernel/-/blame/d7a06983a01a33605191c0766857b832ac32a2b6/fs/locks.c?page=3).
    
- **Обработка блокировок**: Основная логика для `fcntl` сосредоточена в функциях `fcntl_setlk` и `fcntl_setlk64` (для 32-битных систем) в файле `fs/locks.c` [](https://gitlab.sdu.dk/sdurobotics/linux-kernels/kernel/-/blame/d7a06983a01a33605191c0766857b832ac32a2b6/fs/locks.c?page=3). Эти функции, после проверки прав и копирования данных из пространства пользователя, вызывают `do_lock_file_wait`, которая, в свою очередь, обращается к `vfs_lock_file` — интерфейсу уровня VFS [](https://gitlab.sdu.dk/sdurobotics/linux-kernels/kernel/-/blame/d7a06983a01a33605191c0766857b832ac32a2b6/fs/locks.c?page=3).
    
- **Принудительные блокировки**: В коде ядра проверка на принудительную блокировку осуществляется функцией `mandatory_lock(inode)`, которая вызывается перед попыткой установки блокировки [](https://gitlab.sdu.dk/sdurobotics/linux-kernels/kernel/-/blame/d7a06983a01a33605191c0766857b832ac32a2b6/fs/locks.c?page=3).
    
- **Снятие блокировок при закрытии**: Критически важный аспект семантики POSIX блокировок реализован в функции `locks_remove_posix`. Она вызывается, когда файловый дескриптор удаляется из файловой таблицы процесса, и снимает все POSIX блокировки, принадлежащие этому процессу для данного файла, даже если они были установлены через другой дескриптор [](https://gitlab.sdu.dk/sdurobotics/linux-kernels/kernel/-/blame/d7a06983a01a33605191c0766857b832ac32a2b6/fs/locks.c?page=3)[](https://snapshots.sourceware.org/glibc/trunk/2024-06-26_11-04_1719399841/manual/html_node/File-Locks.html#index-fcntl_002eh-6).
    
- **Просмотр блокировок**: Информация о текущих блокировках в системе доступна через файл `/proc/locks`, который формируется с использованием операций `locks_seq_operations` [](https://gitlab.sdu.dk/sdurobotics/linux-kernels/kernel/-/blame/d7a06983a01a33605191c0766857b832ac32a2b6/fs/locks.c?page=3). Команда `lslocks` также использует этот интерфейс [](https://cloud.tencent.cn/developer/article/1868981?from=15425). В выводе можно увидеть класс блокировки (FLOCK или POSIX), тип (ADVISORY), вид блокировки (READ/WRITE) и PID владельца [](https://cloud.tencent.cn/developer/article/1868981?from=15425).
    
- **Справедливость (Fairness)**: Исторически очередь блокировок (`fl_block`) могла быть несправедливой, отдавая приоритет локальным процессам перед запросами от NFS-клиентов. Ведутся работы по модернизации очереди для обеспечения порядка "первым пришел — первым обслужен" (FIFO) для всех типов блокировок, включая NFSv4, где требуется поддержка уведомлений о блокировках.
    
- **Проблема "призрачных" блокировок**: В распределенных системах клиент может запросить блокировку и перестать отвечать (или быть убит). Для решения этой проблемы в NFSv4 используются механизмы аренды (leases) и времени ожидания, по истечении которых блокировка может быть снята или отдана другому.
    
- **Наследование при fork**: POSIX блокировки (`fcntl`) **не наследуются** дочерним процессом при вызове `fork()` [](https://precise.sao.ru/hq/sts/linux/book/bogatyrev_c_unix/gl_6_7.shtml#6_10)[](https://snapshots.sourceware.org/glibc/trunk/2024-06-26_11-04_1719399841/manual/html_node/File-Locks.html#index-fcntl_002eh-6). Это логично, так как дочерний процесс имеет свой PID, а блокировка привязана к PID родителя. `flock` ведет себя иначе из-за привязки к открытому файловому описанию, но, как отмечено в мануале, если ребенок явно снимет блокировку, родитель её потеряет, что указывает на разделение одного объекта [](https://www.daemon-systems.org/man/flock.2.html).
    

### 5. Современные тенденции и специализированные решения

- **Open File Description Locks**: В современных системах, включая GNU/Linux, появились блокировки, привязанные к открытому файловому описанию (open file description locks), которые объединяют семантику `flock` (привязка к `struct file`) и гибкость POSIX (блокировка диапазонов). Это отдельный тип блокировок, доступный через `fcntl` с командами `F_OFD_SETLK`, `F_OFD_SETLKW`, `F_OFD_GETLK` [](https://snapshots.sourceware.org/glibc/trunk/2024-06-26_11-04_1719399841/manual/html_node/File-Locks.html#index-fcntl_002eh-6).
    
- **Распределенные блокировки (DLMPFS)**: Недавно предложенная файловая система `dlmpfs` (Distributed Lock Manager Pseudo Filesystem) позволяет использовать стандартные вызовы `flock` и `fcntl` для управления распределенными блокировками в кластере через DLM (Distributed Lock Manager). Это стирает грань между локальными и кластерными блокировками, позволяя адаптировать существующие приложения без переписывания кода.
    

### Заключение

Блокировки файлов в Linux — это многослойный механизм, основанный на рекомендательной модели. Выбор между `flock` (простота, работа с файлом целиком) и `fcntl` (точность, работа с диапазонами, поддержка NFS) диктуется конкретной задачей синхронизации. Внутри ядра эти механизмы реализованы через сложные очереди и структуры данных (последние изменения в 2024 году разделили блокировки и аренду через `struct file_lock_core`), которые постоянно развиваются для обеспечения справедливости, поддержки асинхронных операций в кластерных средах и повышения производительности за счет более тонкой гранулярности внутренних блокировок ФС. Просмотр текущих блокировок осуществляется через интерфейс `/proc/locks`.
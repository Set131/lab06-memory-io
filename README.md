# Лабораторна робота №6: Віртуальна пам'ять, File Descriptors та I/O

## 1. Мета лабораторної роботи

У попередніх лабораторних ми дослідили шлях: Executable → Process → Threads → CPU. Тепер дослідимо дві фундаментальні абстракції, які operating system надає process:

```
                 Process
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
 Virtual Address Space     File Descriptors
        │                       │
        ▼                       ▼
      Memory                    I/O
```

**Центральні питання:**
- Яку пам'ять насправді бачить process?
- Як virtual address пов'язана з physical memory?
- Як process працює з файлами та іншими I/O resources?
- Що насправді відбувається під час `open()`, `read()`, `write()` та `close()`?

---

## 2. Середовище

- **WSL 2**, Ubuntu 24.04 LTS (resolute)
- **gcc 15.2.0**
- **strace 6.19**
- **PAGE_SIZE = 4096 байт**
- Архітектура: **x86-64 (AMD64)**
- Хост: Windows 11 Pro

---

# Частина I. Virtual Memory

## 3. Завдання 1: Де знаходяться дані програми?

### Програма `memory_layout.c`

```c
#include <stdio.h>
#include <stdlib.h>

int global_initialized = 10;
int global_uninitialized;

static int static_value = 20;

void function(void) { }

int main(void)
{
    int local = 30;
    int *dynamic = malloc(sizeof(*dynamic));

    if (dynamic == NULL) { perror("malloc"); return EXIT_FAILURE; }
    *dynamic = 40;

    printf("function:             %p\n", (void *)function);
    printf("global initialized:   %p\n", (void *)&global_initialized);
    printf("global uninitialized: %p\n", (void *)&global_uninitialized);
    printf("static:               %p\n", (void *)&static_value);
    printf("heap:                 %p\n", (void *)dynamic);
    printf("stack:                %p\n", (void *)&local);

    free(dynamic);
    return EXIT_SUCCESS;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra memory_layout.c -o memory_layout
./memory_layout (3 рази)
```

### Результати

| Об'єкт | Run 1 | Run 2 | Run 3 | Область |
|--------|-------|-------|-------|---------|
| function | `0x5c7b974581c9` | `0x635c09cd61c9` | `0x5e2abe3f61c9` | `.text` |
| initialized global | `0x5c7b9745b010` | `0x635c09cd9010` | `0x5e2abe3f9010` | `.data` |
| uninitialized global | `0x5c7b9745b01c` | `0x635c09cd901c` | `0x5e2abe3f901c` | `.bss` |
| static | `0x5c7b9745b014` | `0x635c09cd9014` | `0x5e2abe3f9014` | `.data` |
| heap allocation | `0x5c7ba958e010` | `0x635c4997c010` | `0x5e2ac55cb010` | heap |
| local variable | `0x7fffbedd324c` | `0x7ffc0fc4c5cc` | `0x7fff73b3a04c` | stack |

### Відповіді на питання розділу 5

**1. Які адреси близько?**
`function`, `global_*`, `static` — усі в діапазоні `0x5c7b...` (низькі адреси).

**2. Чи heap схожа на global?**
Так, обидва в діапазоні `0x5c7b...`, але heap далі (різниця ~0x14M).

**3. Чи stack в іншій області?**
Так — `0x7fff...` (високі адреси).

**4. Чи адреси однакові між запусками?**
Ні — ASLR змінює базові адреси.

**Важливий висновок:** усі адреси — **virtual addresses**, а не physical RAM.

---

## 4. Завдання 2: Memory Map Process

### Програма `memory_maps.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int global_value = 10;

int main(void)
{
    int local = 20;
    int *dynamic = malloc(sizeof(*dynamic));
    if (dynamic == NULL) { perror("malloc"); return EXIT_FAILURE; }
    *dynamic = 30;

    printf("PID:    %d\n", getpid());
    printf("main:   %p\n", (void *)main);
    printf("global: %p\n", (void *)&global_value);
    printf("heap:   %p\n", (void *)dynamic);
    printf("stack:  %p\n", (void *)&local);

    fflush(stdout);
    sleep(60);
    free(dynamic);
    return EXIT_SUCCESS;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra memory_maps.c -o memory_maps
./memory_maps

# У другому терміналі:
cat /proc/PID/maps
```

### Результат `/proc/PID/maps` (ключові regions)

| Region | Адреса | Права | Що це |
|--------|--------|-------|-------|
| `.text` executable | `5a9fb3e8e000-5a9fb3e8f000` | `r-xp` | код `main`, `function` |
| `.rodata` | `5a9fb3e8d000-5a9fb3e8e000` | `r--p` | read-only data |
| `.data`/`.bss` | `5a9fb3e91000-5a9fb3e92000` | `rw-p` | `global_value` |
| `[heap]` | `5a9fba094000-5a9fba0b5000` | `rw-p` | `malloc` |
| `libc.so.6` | `7e4f6e600000-...` | `r--p`, `r-xp`, `rw-p` | C library |
| `ld-linux` | `7e4f6e969000-...` | | dynamic loader |
| `[vvar]`, `[vdso]` | `7e4f6e961000-...` | `r--p`, `r-xp` | kernel helpers |
| `[stack]` | `7ffec8789000-7ffec87ab000` | `rw-p` | стек |

### Зіставлення адрес

| Об'єкт | Адреса з програми | Mapping у `/proc/PID/maps` |
|--------|-------------------|---------------------------|
| `main` | `0x5a9fb3e8e...` | `5a9fb3e8e000-5a9fb3e8f000 r-xp` (`.text`) |
| `global` | `0x5a9fb3e91...` | `5a9fb3e91000-5a9fb3e92000 rw-p` (`.data`) |
| `heap` | `0x5a9fba0...` | `5a9fba094000-5a9fba0b5000 rw-p [heap]` |
| `stack` | `0x7ffec87...` | `7ffec8789000-7ffec87ab000 rw-p [stack]` |

### Відповіді на питання розділу 9 (Memory Permissions)

**1. Чому code region не writable?**
Бо код не повинен змінюватися під час виконання — це захист від випадкової або зловмисної модифікації коду (W^X policy). Права `r-xp` дозволяють читати й виконувати, але не писати.

**2. Чому data region не executable?**
Бо дані не повинні виконуватися як код — це захист від code injection. Права `rw-p` дозволяють читати й писати, але не виконувати.

**3. Зв'язок із memory protection:**
MMU перевіряє права доступу для кожної сторінки. Спроба записати в `r-xp` або виконати `rw-p` викликає fault (SIGSEGV).

---

## 5. Завдання 3: Page Size

### Програма `page_size.c`

```c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    long page_size = sysconf(_SC_PAGESIZE);
    printf("Page size: %ld bytes\n", page_size);
    return 0;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra page_size.c -o page_size
./page_size
getconf PAGE_SIZE
```

### Результат

```
Page size: 4096 bytes
4096
```

**PAGE_SIZE = 4096 байт** (4 KB) — стандартний розмір сторінки на x86-64 Linux.

### Page Number та Offset

Для virtual address A:
```
Page Number = A / 4096
Offset      = A % 4096
```

Приклад: адреса `0x5c7b9745b010`
- Page Number = `0x5c7b9745b` (старші біти)
- Offset = `0x010` (молодші 12 біт)

---

## 6. Завдання 4: Heap складається з Virtual Memory

### Програма `heap_growth.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    const size_t size = 100 * 1024 * 1024;

    printf("PID: %d\n", getpid());
    char *memory = malloc(size);
    if (memory == NULL) { perror("malloc"); return EXIT_FAILURE; }

    printf("Allocated %zu bytes\n", size);
    printf("Address: %p\n", (void *)memory);
    printf("Press Enter to touch memory...\n");
    getchar();

    for (size_t i = 0; i < size; i += 4096) {
        memory[i] = 1;
    }

    printf("Memory touched. Press Enter to exit...\n");
    getchar();
    free(memory);
    return EXIT_SUCCESS;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra heap_growth.c -o heap_growth
./heap_growth

# У другому терміналі (до touch):
grep -E "VmSize|VmRSS" /proc/PID/status

# Після touch:
grep -E "VmSize|VmRSS" /proc/PID/status
```

### Результати

| Момент | VmSize | VmRSS | Різниця |
|--------|--------|-------|---------|
| **До touch** | 105176 kB (~102.7 MB) | **1880 kB** (~1.8 MB) | ~100.9 MB |
| **Після touch** | 105176 kB (~102.7 MB) | **104276 kB** (~101.8 MB) | ~0.9 MB |

### Що показує цей експеримент

1. **`malloc(100 MB)` не означає негайне виділення 100 MB фізичної RAM** — `VmSize` одразу ~105 MB, але `VmRSS` лише ~1.8 MB.

2. **Lazy allocation (demand paging)** — kernel виділяє фізичні сторінки лише тоді, коли програма **вперше звертається** до віртуальної адреси (page fault → kernel виділяє фізичну сторінку).

3. **Після touch** — `VmRSS` зріс до ~104 MB, бо програма записала по байту на кожну сторінку (кожні 4096 байт), і kernel виділив фізичні сторінки для всіх 100 MB.

4. **`VmSize` не змінився** — віртуальний address space був зарезервований ще на `malloc`.

**Висновок:**
```
malloc(100 MB)
      ↓
VmSize += 100 MB    (віртуальний reservation)
VmRSS  += ~0        (фізична RAM ще не виділена)
      ↓
touch memory (page fault)
      ↓
VmSize  = 100 MB    (не змінюється)
VmRSS  += 100 MB    (фізичні сторінки виділені)
```

Virtual memory дозволяє **розділити** reservation/allocation (віртуальний address space) та commitment/use (фізична RAM).

---

## 7. Завдання 5: Invalid Memory Access

### Програма `invalid_memory.c`

```c
#include <stdio.h>

int main(void)
{
    int *pointer = NULL;

    printf("About to access invalid memory\n");
    fflush(stdout);

    *pointer = 42;

    printf("You should not normally see this\n");
    return 0;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra invalid_memory.c -o invalid_memory
./invalid_memory
strace ./invalid_memory
```

### Результат

```
About to access invalid memory
Segmentation fault         ./invalid_memory
```

`strace` (кінець trace):
```
write(1, "About to access invalid memory\n", 31) = 31
--- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_MAPERR, si_addr=NULL} ---
+++ killed by SIGSEGV +++
```

### Що сталося концептуально

```
Process
   │
   │ *pointer = 42 (запис за адресою NULL = 0x0)
   ▼
MMU перевіряє page table для адреси 0x0
   │
   │ mapping не знайдено → SEGV_MAPERR
   ▼
Kernel надсилає SIGSEGV
   │
   ▼
Process terminated
```

### Відповіді на питання розділу 15

**1. Чому OS не дозволяє записати за address 0?**
Адреса 0 не входить у жоден mapped region process. MMU перевіряє page table, не знаходить mapping → fault.

**2. Як це захищає систему?**
Без memory protection програма могла б випадково або зловмисно записати в чужу пам'ять (іншого process, kernel). Protection ізолює processes.

**3. Чи Segmentation fault означає, що RAM фізично зламана?**
Ні — це **програмна** помилка (invalid access), а не апаратна.

**Висновок:** memory protection підтримується hardware (MMU) та OS (page table), і некоректний access не перетворюється на довільний запис у чужу пам'ять.

---

# Частина II. File Descriptors та I/O

## 8. Завдання 7: Standard File Descriptors

### Програма `standard_fds.c`

```c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    printf("STDIN  = %d\n", STDIN_FILENO);
    printf("STDOUT = %d\n", STDOUT_FILENO);
    printf("STDERR = %d\n", STDERR_FILENO);
    return 0;
}
```

### Результат

```
STDIN  = 0
STDOUT = 1
STDERR = 2
```

### Програма `fd_observer.c`

```c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    printf("PID: %d\n", getpid());
    fflush(stdout);
    sleep(60);
    return 0;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra fd_observer.c -o fd_observer
./fd_observer

# У другому терміналі:
ls -l /proc/PID/fd
```

### Результат

```
0 -> /dev/pts/4   (stdin)
1 -> /dev/pts/4   (stdout)
2 -> /dev/pts/4   (stderr)
```

**Відповідь на питання розділу 18:** усі три descriptors вказують на `/dev/pts/4` — pseudo-terminal (PTY), який надає WSL для вашого терміналу.

---

## 9. Завдання 8: Відкриваємо File

### Програма `open_file.c`

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    int fd = open("example.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);

    if (fd == -1) {
        perror("open");
        return EXIT_FAILURE;
    }

    printf("Opened file descriptor: %d\n", fd);
    sleep(30);
    close(fd);
    return EXIT_SUCCESS;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra open_file.c -o open_file
./open_file

# У другому терміналі (PID знайти через pgrep open_file):
pgrep open_file
ls -l /proc/PID/fd
```

### Результат

```
Opened file descriptor: 3

0 -> /dev/pts/4
1 -> /dev/pts/4
2 -> /dev/pts/4
3 -> /home/user/labs/lab06-memory-io/example.txt   ← наш файл!
```

### Відповіді на питання розділів 20–21

**1. Чому file отримав fd 3?**
Kernel повертає **найменший вільний** descriptor. 0, 1, 2 зайняті стандартними потоками, тому наступний вільний — 3. Але код не повинен покладатися на те, що `open()` завжди поверне саме 3.

**2. Pathname ≠ File Descriptor:**
```
"example.txt"          ← pathname (ім'я файлу)
      │
      │ open()
      ▼
    Kernel             ← резолвить pathname у внутрішню структуру файлу
      │
      ▼
     fd 3              ← descriptor (handle) для подальших read/write
```

Після `open()` програма використовує `fd` (число 3) для `read()`/`write()`, а не повторно передає `"example.txt"`.

---

## 10. Завдання 9: write()

### Програма `write_file.c`

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
    const char message[] = "Hello from write()\n";

    int fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) { perror("open"); return EXIT_FAILURE; }

    ssize_t written = write(fd, message, strlen(message));
    if (written == -1) { perror("write"); close(fd); return EXIT_FAILURE; }

    printf("Requested: %zu bytes\n", strlen(message));
    printf("Written:   %zd bytes\n", written);

    if (close(fd) == -1) { perror("close"); return EXIT_FAILURE; }
    return EXIT_SUCCESS;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra write_file.c -o write_file
./write_file
cat output.txt
```

### Результат

```
Requested: 19 bytes
Written:   19 bytes
Hello from write()
```

### Відповіді на питання розділу 23

**1. Де знаходиться `message`?**
У **virtual memory** process — на stack (локальна змінна `message[]` у `main`).

**2. Що означає `fd`?**
File descriptor — числовий handle (тут 3), який kernel пов'язав із файлом `output.txt`.

**3. Що означає `size`?**
Кількість байт для запису — `strlen(message)` = 19.

**4. Що повертає `write()`?**
Кількість реально записаних байт (`ssize_t`). Може бути менше за `size` (partial write).

### Data flow

```
message[] (stack, virtual memory)
   │
   │ write(fd=3, buffer, 19)
   ▼
══════════════════════
   Kernel
══════════════════════
   │
   ▼
File output.txt (на диску)
```

---

## 11. Завдання 10: read()

### Програма `read_file.c`

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    char buffer[16];
    int fd = open("input.txt", O_RDONLY);
    if (fd == -1) { perror("open"); return EXIT_FAILURE; }

    ssize_t bytes_read;

    while ((bytes_read = read(fd, buffer, sizeof(buffer))) > 0) {
        if (write(STDOUT_FILENO, buffer, (size_t)bytes_read) == -1) {
            perror("write");
            close(fd);
            return EXIT_FAILURE;
        }
    }

    if (bytes_read == -1) { perror("read"); }
    close(fd);
    return bytes_read == -1 ? EXIT_FAILURE : EXIT_SUCCESS;
}
```

### Виконані команди

```bash
printf 'Computer Systems\nOperating Systems\n' > input.txt
gcc -Wall -Wextra read_file.c -o read_file
./read_file
strace -e trace=openat,read,write,close ./read_file
```

### Результат

```
Computer Systems
Operating Systems
```

### `strace` (ключові рядки)

```
openat(AT_FDCWD, "input.txt", O_RDONLY) = 3
read(3, "Computer Systems", 16) = 16
write(1, "Computer Systems", 16) = 16
read(3, "\nOperating Syste", 16) = 16
write(1, "\nOperating Syste", 16) = 16
read(3, "ms\n", 16) = 3
write(1, "ms\n", 3) = 3
read(3, "", 16) = 0              ← EOF
close(3) = 0
```

### Відповідь на питання розділу 26

`read()` повертає:
- `> 0` — кількість прочитаних байт
- `0` — end of file
- `-1` — error

Pattern:
```
while read returns data
        ↓
   process data
        ↓
    read again
        ↓
       EOF
```

---

## 12. Завдання 11–12: File Offset та lseek()

### Програма `file_offset.c` (без lseek)

```c
ssize_t n1 = read(fd, buffer, 5);   // читає "Compu"
ssize_t n2 = read(fd, buffer, 5);   // читає "ter S" (продовжує з offset 5)
```

### Результат

```
First:  [Compu]
Second: [ter S]
```

### Програма `file_offset.c` (з lseek)

```c
ssize_t n1 = read(fd, buffer, 5);   // читає "Compu"
lseek(fd, 0, SEEK_SET);              // offset → 0
ssize_t n2 = read(fd, buffer, 5);   // знову "Compu"
```

### Результат

```
First:  [Compu]
Second: [Compu]   ← lseek повернув offset на початок
```

### Відповідь на питання розділу 29

**Чим sequential access відрізняється від random access?**
- **Sequential access** — читаємо/пишемо послідовно, offset автоматично просувається (як у першому варіанті без `lseek`).
- **Random access** — можемо перемістити offset у будь-яку позицію через `lseek()`, потім читати/писати звідти.

---

## 13. Завдання 14: stdout та stderr

### Програма `streams.c`

```c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    const char out[] = "Normal output\n";
    const char err[] = "Error output\n";

    write(STDOUT_FILENO, out, sizeof(out) - 1);
    write(STDERR_FILENO, err, sizeof(err) - 1);

    return 0;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra streams.c -o streams
./streams
./streams > normal.txt
cat normal.txt
```

### Результат

```
$ ./streams
Normal output
Error output

$ ./streams > normal.txt
Error output           ← stderr залишився в терміналі

$ cat normal.txt
Normal output          ← stdout пішов у файл
```

### Відповіді на питання розділів 31–32

**1. Що сталося з normal output?**
Пішов у файл `normal.txt` (shell змінив fd 1 з `/dev/pts/4` на `normal.txt`).

**2. Що сталося з error output?**
Залишився в терміналі (fd 2 не перенаправлявся).

**3. Redirection:**
```
Before:                      After `> normal.txt`:
fd 1 → /dev/pts/4            fd 1 → normal.txt
fd 2 → /dev/pts/4            fd 2 → /dev/pts/4
```

Програма все ще виконує `write(STDOUT_FILENO, ...)`, тобто `write(1, ...)`. Але shell змінив resource, на який посилається descriptor 1. Програма не потребує спеціального коду для запису у файл.

---

## 14. Завдання 15–16: printf() vs write() та fflush()

### Програма `buffering.c` (без fflush)

```c
printf("A");
printf("B");
printf("C");
write(STDOUT_FILENO, "D", 1);
```

### Результат

```
DABC

strace -e write ./buffering:
write(1, "D", 1)      = 1    ← write("D") одразу
write(1, "ABC", 3)    = 3    ← stdio buffer flush при завершенні
```

### Програма `buffering.c` (з fflush)

```c
printf("A");
printf("B");
printf("C");
fflush(stdout);
write(STDOUT_FILENO, "D", 1);
```

### Результат

```
ABCD

strace -e write ./buffering:
write(1, "ABC", 3)    = 3    ← fflush виштовхнув ABC
write(1, "D", 1)      = 1    ← write("D") після
```

### Відповіді на питання розділів 34–35

**Чому порядок може здивувати?**
`printf()` використовує C stdio buffering, а `write()` звертається до POSIX I/O interface без stdio buffer. Тому `printf("A")`, `printf("B")`, `printf("C")` накопичуються в stdio buffer, а `write("D")` одразу йде в kernel. Лише при завершенні програми stdio buffer flushиться.

**Роль `fflush(stdout)`:** примусово виштовхує вміст stdio buffer у kernel **негайно**, не чекаючи завершення програми. Це синхронізує порядок між `printf` та `write`.

**Program Order ≠ Immediate External I/O Order** — якщо використовуються різні buffering layers.

---

## 15. Завдання 17–18: Фінальна програма `file_copy.c`

### Програма `file_copy.c`

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define BUFFER_SIZE 4096

int main(int argc, char *argv[])
{
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <source> <destination>\n", argv[0]);
        return EXIT_FAILURE;
    }

    int source = open(argv[1], O_RDONLY);
    if (source == -1) { perror("open source"); return EXIT_FAILURE; }

    int destination = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (destination == -1) {
        perror("open destination");
        close(source);
        return EXIT_FAILURE;
    }

    char buffer[BUFFER_SIZE];
    ssize_t bytes_read;

    while ((bytes_read = read(source, buffer, sizeof(buffer))) > 0) {
        ssize_t total_written = 0;

        while (total_written < bytes_read) {
            ssize_t written = write(destination,
                                    buffer + total_written,
                                    (size_t)(bytes_read - total_written));
            if (written == -1) {
                perror("write");
                close(source);
                close(destination);
                return EXIT_FAILURE;
            }
            total_written += written;
        }
    }

    if (bytes_read == -1) {
        perror("read");
        close(source);
        close(destination);
        return EXIT_FAILURE;
    }

    if (close(source) == -1) { perror("close source"); }
    if (close(destination) == -1) { perror("close destination"); }
    return EXIT_SUCCESS;
}
```

### Виконані команди

```bash
printf 'Operating Systems\nComputer Architecture\n' > source.txt
gcc -Wall -Wextra file_copy.c -o file_copy
./file_copy source.txt destination.txt
cmp source.txt destination.txt
echo $?
```

### Результат (text file)

```
$ ./file_copy source.txt destination.txt
$ cmp source.txt destination.txt
$ echo $?
0              ← файли ідентичні
```

### Перевірка binary

```bash
./file_copy file_copy copy_of_file_copy
cmp file_copy copy_of_file_copy
echo $?
```

### Результат (binary file)

```
0              ← binary executable скопійовано правильно
```

**Відповідь на питання розділу 43:** той самий `read`/`write` loop копіює і text, і binary — бо програма працює з **байтами**, а не з "текстом" як спеціальною формою.

### `strace` для file_copy

```
openat(AT_FDCWD, "source.txt", O_RDONLY) = 3
openat(AT_FDCWD, "destination.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644) = 4
read(3, "Operating Systems\nComputer Archi"..., 4096) = 40
write(4, "Operating Systems\nComputer Archi"..., 40) = 40
read(3, "", 4096) = 0              ← EOF
close(3) = 0
close(4) = 0
```

**Розшифровка:**

| Syscall | Що робить | Відповідність коду |
|---------|-----------|-------------------|
| `openat(..., "source.txt", O_RDONLY)` | Відкриває source на читання | `int source = open(argv[1], O_RDONLY)` |
| `openat(..., "destination.txt", ...)` | Створює/відкриває destination на запис | `int destination = open(argv[2], ...)` |
| `read(3, ..., 4096) = 40` | Читає 40 байт з source | `read(source, buffer, sizeof(buffer))` |
| `write(4, ..., 40) = 40` | Записує 40 байт у destination | `write(destination, ...)` |
| `read(3, "", 4096) = 0` | EOF | умова `bytes_read > 0` стає false |
| `close(3) = 0` | Закриває source | `close(source)` |
| `close(4) = 0` | Закриває destination | `close(destination)` |

### Повний Data Path

```
SOURCE FILE (source.txt)
     │
     │ read(3, buffer, 4096)
     ▼
══════════════════════
     Kernel
══════════════════════
     │
     ▼
Process Virtual Memory
     │
     │ buffer[4096]  ← на stack процесу
     │
     ▼
══════════════════════
     Kernel
══════════════════════
     │
     │ write(4, buffer, bytes_read)
     ▼
DESTINATION FILE (destination.txt)
```

---

## 16. Відповіді на контрольні питання

1. **Що таке virtual address?**
   Адреса в address space process, яку бачить програма. MMU транслює її в physical address.

2. **Чим virtual address відрізняється від physical address?**
   Virtual — те, що бачить програма; physical — реальна адреса в RAM. MMU виконує трансляцію через page table.

3. **Що таке virtual address space?**
   Діапазон virtual addresses, доступних process. Зазвичай 0x0 – 0x7fff... для x86-64.

4. **Чому два processes можуть використовувати однакову virtual address?**
   Бо їхні page tables відображають цю адресу на різні physical frames. Ізоляція через MMU.

5. **Що таке page?**
   Блок virtual memory фіксованого розміру (4096 байт на x86-64). Мінімальна одиниця трансляції.

6. **Що таке page frame?**
   Блок physical RAM того ж розміру, що й page. Page → page frame через page table.

7. **Що таке page table?**
   Структура даних, яка зберігає mapping virtual page → physical frame для кожного process.

8. **Яку роль виконує MMU?**
   Memory Management Unit — hardware, який транслює virtual address у physical address, перевіряє права доступу.

9. **Що таке TLB?**
   Translation Lookaside Buffer — кеш page table entries для швидшої трансляції.

10. **Чим TLB miss відрізняється від page fault?**
    TLB miss — page table entry є, але не в кеші (швидко обробляється MMU). Page fault — mapping немає в page table (потрібен kernel).

11. **Що таке page fault?**
    Ситуація, коли MMU не може знайти mapping для virtual address у page table. Kernel обробляє fault.

12. **Чи page fault завжди означає помилку програми?**
    Ні. Page fault може бути legitimate (lazy allocation, copy-on-write, swap). Лише invalid access → SIGSEGV.

13. **Як virtual memory допомагає ізолювати processes?**
    Кожен process має власну page table, тому його virtual addresses відображаються на ізольовані physical frames. Process A не може отримати доступ до пам'яті Process B.

14. **Що означають r, w, x у memory mappings?**
    Readable, writable, executable. `r-xp` = read + execute, `rw-p` = read + write.

15. **Чому code не обов'язково повинен бути writable?**
    W^X policy: код не повинен змінюватися під час виконання (захист від модифікації).

16. **Чому malloc() не слід просто ототожнювати з негайним використанням такої ж кількості physical RAM?**
    Бо malloc лише резервує virtual address space. Physical RAM виділяється лише при першому зверненні (demand paging).

17. **Що таке file descriptor?**
    Числовий handle, який kernel пов'язує з відкритим файлом/pipes/socket/device. Використовується в read/write/close.

18. **Чим pathname відрізняється від file descriptor?**
    Pathname — ім'я файлу в файловій системі. File descriptor — внутрішній handle kernel для відкритого ресурсу.

19. **Що зазвичай представляють descriptors 0, 1, 2?**
    stdin (0), stdout (1), stderr (2).

20. **Що робить open()?**
    Відкриває файл, повертає file descriptor (найменший вільний).

21. **Що повертає read()?**
    Кількість прочитаних байт (> 0), 0 (EOF) або -1 (error).

22. **Що означає read() == 0?**
    End of file — більше даних немає.

23. **Що повертає write()?**
    Кількість реально записаних байт.

24. **Чому потрібно враховувати partial writes?**
    write() може записати менше requested bytes (наприклад, при записі у pipe або на повільний диск). Треба продовжувати запис решти.

25. **Що таке file offset?**
    Поточна позиція в файлі, з якої буде наступний read/write.

26. **Для чого використовується lseek()?**
    Для переміщення file offset у вказану позицію (random access).

27. **Чим FILE * відрізняється від file descriptor?**
    FILE * — буферизована абстракція C stdio (fopen/fread/fwrite). File descriptor — низькорівневий handle kernel (open/read/write).

28. **Чому printf() не обов'язково негайно викликає write()?**
    Бо printf використовує stdio buffer. Дані накопичуються в буфері і виштовхуються при flush (або при завершенні програми).

29. **Що таке buffering?**
    Механізм накопичення даних у проміжному буфері перед відправкою в kernel. Зменшує кількість syscalls.

30. **Що робить fflush()?**
    Примусово виштовхує вміст stdio buffer у kernel (викликає write()).

31. **Чому redirection > може працювати без зміни source code програми?**
    Бо shell змінює resource, на який посилається fd 1 (stdout), перед запуском програми. Програма все ще пише в fd 1, але fd 1 тепер вказує на файл.

32. **Чому file_copy може однаково копіювати text та binary files?**
    Бо на рівні read/write програма працює з байтами, а не з "текстом" як спеціальною формою. read повертає байти, write записує байти.

33. **Де фізично/логічно знаходиться buffer під час read()?**
    У virtual memory process — на stack (локальна змінна `char buffer[...]`). Kernel копіює дані з file у цей buffer.

34. **Як file descriptor, virtual memory та system call взаємодіють під час I/O?**
    Програма передає fd і virtual address buffer у read(). Kernel резолвить fd у файл, читає дані з файлу і копіює їх у virtual memory process за вказаною адресою (через page table → physical frames).

---

## 17. Висновок

Лабораторна робота експериментально підтвердила:

- **Virtual addresses:** усі адреси в програмі — virtual, а не physical RAM
- **Memory layout:** `.text` (низькі) → `.data`/`.bss` → heap → stack (високі)
- **`/proc/PID/maps`:** показує regions process з правами `r--p`, `r-xp`, `rw-p`
- **Memory protection:** code `r-xp` (read+execute), data `rw-p` (read+write); invalid access → SIGSEGV
- **Page size:** 4096 байт на x86-64
- **Virtual memory vs physical memory:** `malloc(100 MB)` → `VmSize += 100 MB`, але `VmRSS += ~0` до touch; після touch `VmRSS += 100 MB` (demand paging)
- **Invalid access:** `*NULL = 42` → SIGSEGV (`SEGV_MAPERR`, `si_addr=NULL`)
- **Standard fds:** 0 (stdin), 1 (stdout), 2 (stderr) → `/dev/pts/4` у WSL
- **`open()`:** повертає найменший вільний fd (3 після 0/1/2)
- **Pathname vs fd:** `"file.txt"` → `open()` → `fd 3` → `read/write`
- **`write()`:** повертає кількість записаних байт; може бути partial
- **`read()`:** повертає кількість прочитаних байт, 0 (EOF) або -1 (error)
- **File offset:** автоматично просувається після read/write; `lseek()` дозволяє random access
- **Redirection:** `> file` змінює fd 1, програма не потребує змін
- **Buffering:** `printf` буферизує, `write` — ні; `fflush` синхронізує
- **`file_copy`:** read/write loop копіює і text, і binary (бо працює з байтами)

**Головна ідея:** Module 6 об'єднує **memory** (virtual address space) та **I/O** (file descriptors) через system calls. `read(fd, buffer, size)` — це точка, де file descriptor, virtual memory та kernel взаємодіють: kernel резолвить fd у файл, читає дані і копіює їх у virtual memory process за адресою buffer.

# Лабораторная работа №1 по ОС

## Тема
Исследование компилятора GCC, язык ассемблера. Связь процесса и операционной системы. Makefile, Git.

---

## Вариант задания
Вычисление **n-го числа Фибоначчи** с использованием параллельного процесса (fork + pipe).

---

## Структура проекта

```
lab1/
├── main.c          # Главный модуль: fork(), pipe(), wait()
├── fib.c           # Модуль вычисления числа Фибоначчи
├── fib.h           # Заголовочный файл
├── Makefile        # Сборка, генерация asm, очистка
├── main.s          # Ассемблер main.c (без оптимизации, -O0)
├── fib.s           # Ассемблер fib.c (с оптимизацией, -O2)
└── README.md
```

---

## Исходный код

### fib.h
```c
#ifndef FIB_H
#define FIB_H

long long fib(int n);

#endif
```

### fib.c
```c
#include "fib.h"

// Итеративное вычисление числа Фибоначчи
long long fib(int n) {
    if (n <= 1) return n;
    long long a = 0, b = 1;
    for (int i = 2; i <= n; i++) {
        long long tmp = a + b;
        a = b;
        b = tmp;
    }
    return b;
}
```

### main.c
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include "fib.h"

int main() {
    int n = 10;
    int fd[2];          // fd[0] — чтение, fd[1] — запись

    if (pipe(fd) == -1) {
        perror("pipe");
        exit(EXIT_FAILURE);
    }

    pid_t pid = fork();

    if (pid == -1) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        // === Дочерний процесс ===
        close(fd[0]);                          // закрыть конец чтения
        long long result = fib(n);
        write(fd[1], &result, sizeof(result)); // передать результат родителю
        close(fd[1]);
        exit(EXIT_SUCCESS);
    } else {
        // === Родительский процесс ===
        close(fd[1]);                          // закрыть конец записи
        long long result;
        read(fd[0], &result, sizeof(result));  // получить результат от дочернего
        close(fd[0]);
        wait(NULL);                            // дождаться завершения дочернего
        printf("fib(%d) = %lld\n", n, result);
    }

    return 0;
}
```

---

## Makefile

```makefile
CC = gcc
CFLAGS = -Wall -Wextra

all: fib_program

fib_program: main.o fib.o
	$(CC) $(CFLAGS) -o fib_program main.o fib.o

main.o: main.c fib.h
	$(CC) $(CFLAGS) -c main.c

fib.o: fib.c fib.h
	$(CC) $(CFLAGS) -c fib.c

asm:
	gcc -S main.c -o main.s -O0
	gcc -S fib.c  -o fib.s  -O2

clean:
	rm -f *.o *.s fib_program
```

---

## Сборка и запуск

```bash
# Сборка
make

# Запуск
./fib_program

# Генерация ассемблера
make asm

# Очистка
make clean
```

**Ожидаемый вывод:**
```
fib(10) = 55
```

---

## Анализ ассемблерного кода

### Команды генерации

```bash
gcc -S main.c -o main.s -O0   # без оптимизации
gcc -S fib.c  -o fib.s  -O2   # с оптимизацией уровня 2
```

### Разница между -O0 и -O2

| Параметр | Описание | Применение |
|----------|----------|------------|
| `-O0` | Без оптимизации. Все переменные хранятся в памяти (стеке), каждая строка кода явно видна в asm | Отладка, анализ |
| `-O2` | Включает большинство оптимизаций: инлайн, устранение мёртвого кода, оптимизация регистров | Продакшн |

### Фрагмент fib.s с комментариями (-O2)

```asm
fib:
    ; Проверка: if (n <= 1) return n
    cmpl    $1, %edi            ; сравниваем n с 1
    jle     .L_base_case        ; если n <= 1 — переход к базовому случаю

    ; Инициализация: a = 0, b = 1
    xorl    %eax, %eax          ; a = 0 (через XOR — быстрее, чем movl $0)
    movl    $1, %edx            ; b = 1

    ; Счётчик цикла i = 2, условие i <= n
.L_loop:
    ; tmp = a + b
    movq    %rax, %rcx          ; tmp = a
    addq    %rdx, %rcx          ; tmp += b

    ; a = b; b = tmp
    movq    %rdx, %rax          ; a = b
    movq    %rcx, %rdx          ; b = tmp

    ; i++, проверка i <= n
    addl    $1, %esi            ; i++
    cmpl    %esi, %edi          ; сравниваем n и i
    jge     .L_loop             ; если n >= i — продолжаем цикл

    movq    %rdx, %rax          ; возвращаем b (результат)
    ret

.L_base_case:
    movslq  %edi, %rax          ; return n (расширение int → long long)
    ret
```

**Ключевые оптимизации -O2:**
- Переменные `a`, `b`, `tmp` хранятся в регистрах (`%rax`, `%rdx`, `%rcx`), а не в стеке — нет обращений к памяти
- `xorl %eax, %eax` вместо `mov $0, %eax` — команда короче и быстрее
- Цикл не разворачивается при -O2 (разворачивание — на -O3), но убраны лишние `push`/`pop`

---

## Используемые системные вызовы

| Вызов | Назначение |
|-------|-----------|
| `pipe(fd)` | Создаёт однонаправленный канал: `fd[0]` — чтение, `fd[1]` — запись |
| `fork()` | Создаёт дочерний процесс — копию родительского |
| `write(fd[1], &result, size)` | Дочерний записывает результат в канал |
| `read(fd[0], &result, size)` | Родительский читает результат из канала |
| `wait(NULL)` | Родительский ждёт завершения дочернего (предотвращает зомби-процесс) |
| `close(fd[...])` | Закрывает неиспользуемый конец канала |

### Схема взаимодействия процессов

```
Родительский процесс          Дочерний процесс
       |                              |
   fork() ─────────────────────────► |
       |                         fib(n) вычислить
       |                         write(fd[1], result)
   read(fd[0]) ◄─────────────────────|
   wait(NULL)                    exit()
   printf(result)
```

---

## Пример работы с Git

```bash
git init
git add .
git commit -m "lab1: fibonacci via fork+pipe with Makefile and asm"
git remote add origin https://github.com/<username>/os-labs.git
git push -u origin main
```

---

## Выводы

1. **GCC и ассемблер**: флаг `-S` транслирует C-код в ассемблер. Уровень `-O0` сохраняет читаемую структуру кода, `-O2` агрессивно использует регистры и убирает лишние операции с памятью.

2. **Модульность**: разбиение на `main.c` и `fib.c` позволяет независимо оптимизировать функцию вычисления и главный модуль.

3. **fork() + pipe()**: стандартный механизм IPC в Linux. Дочерний процесс вычисляет результат и передаёт его родителю через анонимный канал. Важно закрывать неиспользуемые концы канала во избежание дедлока.

4. **Makefile**: автоматизирует сборку, позволяет пересобирать только изменившиеся модули.

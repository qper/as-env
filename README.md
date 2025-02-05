# as-env

''' find /usr/bin /usr/local/bin /usr/sbin /sbin /bin /lib /lib64 /usr/lib /usr/lib64 /opt /usr/libexec \
  -type f \( -executable -o -name "*.so*" -o -name "*.a" \) -print0 | sort -z | xargs -0 sha256sum'''


find /usr/bin /usr/local/bin /usr/sbin /sbin /bin /lib /lib64 /usr/lib /usr/lib64 /opt /usr/libexec \
  -type f \( -executable -o -name "*.so*" -o -name "*.a" \) -print0 | sort -z | xargs -0 sha256sum


Ты правильно подметил ключевые моменты:  
- **Определить все зависимости** (не только библиотеки, но и другие файлы, которые использует `rmondl`).  
- **Определить необходимые привилегии** (минимальный набор, чтобы запустить без `root`).  
- **Упаковать всё в контейнер (Docker/Podman) и запустить от обычного пользователя**.  

---

## **1. Определение всех зависимостей**

Ты уже скопировал библиотеки, которые `ldd` показал. Однако этого недостаточно. Нам нужно выяснить:  
✅ **Какие файлы он открывает?**  
✅ **Какие системные вызовы требуют привилегий?**  
✅ **Какие устройства/сокеты он использует?**  

### **1.1. Проверка файлов и зависимостей**
Используй `strace`, чтобы увидеть, какие файлы ищет `rmondl`:  
```bash
strace -f -e trace=openat,access,stat,readlink ./rmondl 2>&1 | tee strace_open.log
```
После этого ищи ошибки `ENOENT` (файл не найден) в логе:  
```bash
grep 'ENOENT' strace_open.log
```
Например, если видишь строку:  
```
openat(AT_FDCWD, "/etc/rmondl.conf", O_RDONLY) = -1 ENOENT (No such file or directory)
```
значит, надо скопировать конфигурационный файл в контейнер.

---

### **1.2. Проверка системных вызовов, требующих привилегий**
Запускаем `strace`, чтобы найти ошибки `EPERM (Operation not permitted)`:  
```bash
strace -f -e trace=capset,setuid,setgid,prctl,clone ./rmondl 2>&1 | tee strace_priv.log
```
Затем ищем ошибки в логе:  
```bash
grep 'EPERM' strace_priv.log
```
Если видишь что-то вроде:
```
prctl(PR_SET_DUMPABLE, 0) = -1 EPERM (Operation not permitted)
setuid(0) = -1 EPERM (Operation not permitted)
capset(...) = -1 EPERM (Operation not permitted)
```
это значит, что программа требует **специальные Linux capabilities** (например, `CAP_NET_BIND_SERVICE`, `CAP_SYS_ADMIN`).

---

### **1.3. Проверка работы с сетью**
Некоторые сервисы требуют `root`, если слушают **порты ниже 1024**. Проверим:
```bash
strace -f -e trace=socket,bind,connect,listen ./rmondl 2>&1 | tee strace_net.log
```
Анализируем ошибки `EACCES` или `EPERM`:  
```bash
grep -E 'EACCES|EPERM' strace_net.log
```
Если видишь:
```
bind(3, {sa_family=AF_INET, sin_port=htons(443)}, 16) = -1 EACCES (Permission denied)
```
то **либо нужно запустить на другом порту (например, 8080), либо дать capability `CAP_NET_BIND_SERVICE`**.

---

### **1.4. Проверка доступа к файлам в `/dev`**
Если `rmondl` использует железо (сетевые интерфейсы, файлы `/dev/`), найдем их:
```bash
strace -f -e trace=openat ./rmondl 2>&1 | tee strace_dev.log
grep '/dev/' strace_dev.log
```
Если он обращается, например, к `/dev/shm` или `/dev/net/tun`, это важно для контейнера.

---

## **2. Запуск без `root` с нужными привилегиями**
Допустим, после анализа выяснили, что программе нужны:
✅ `CAP_NET_BIND_SERVICE` (если она слушает порты ниже 1024)  
✅ `CAP_SYS_ADMIN` (если она делает `prctl` или `mount` операции)  
✅ Доступ к файлам конфигурации `/etc/rmondl.conf`, `/var/log/rmondl.log`  

### **2.1. Добавляем `capabilities`**
В `Dockerfile` можно задать:
```dockerfile
FROM rhel8

# Добавляем зависимости
COPY rmondl /usr/local/bin/
COPY lib/* /usr/lib/

# Создаем пользователя
RUN useradd -r -s /bin/false rmondl

# Даём программе нужные капабилити
RUN setcap CAP_NET_BIND_SERVICE,CAP_SYS_ADMIN+ep /usr/local/bin/rmondl

# Переключаемся на пользователя без root
USER rmondl

ENTRYPOINT ["/usr/local/bin/rmondl"]
```

---

### **2.2. Запуск контейнера с привилегиями**
Если `setcap` недостаточно, можно запустить контейнер с `cap_add`:
```bash
podman run --cap-add=NET_BIND_SERVICE --cap-add=SYS_ADMIN my_rmondl
```
или ограничить до нужного:
```bash
podman run --security-opt="no-new-privileges" --cap-add=NET_BIND_SERVICE my_rmondl
```

---

### **3. Проверяем результат**
После всех изменений снова проверяем `strace`, но уже внутри контейнера:
```bash
podman run --rm my_rmondl strace -f ./rmondl
```
Если ошибок `EPERM` больше нет, значит, всё настроено правильно!

---

### **Вывод**
1. **Определяем зависимости:**  
   - `ldd` показывает библиотеки.  
   - `strace` выявляет недостающие файлы и привилегии.  
2. **Даём нужные `capabilities`** через `setcap` или `--cap-add` в Podman.  
3. **Запускаем в контейнере без root**, но с минимально необходимыми правами.  

Попробуй пройти этот процесс и скажи, какие ошибки остались! 🚀






Да, ты правильно думаешь. Судя по сообщению **"you must be a superuser to run this program"**, скорее всего, **проверка на `root` выполняется перед запуском основного кода**.  

### **Как программа проверяет `root`?**  
Обычно такие проверки делают **в самом начале выполнения**, и если пользователь не `root`, программа просто **завершается**.  

Стандартные способы проверки:  
1. **Системный вызов `geteuid()`**  
   ```c
   if (geteuid() != 0) {
       printf("You must be a superuser to run this program.\n");
       exit(1);
   }
   ```
   Этот вызов **возвращает UID (User ID) текущего пользователя**. Если это **не 0 (root)**, программа завершает работу.  

2. **Функция `getuid()` + сравнение**  
   ```c
   if (getuid() != 0) {
       printf("You must be a superuser to run this program.\n");
       exit(1);
   }
   ```
   Разница с `geteuid()` в том, что `getuid()` проверяет **реальный** UID, а `geteuid()` – **эффективный** (если используется `setuid`).  

3. **Проверка групп (`getegid()`, `getgid()`)**  
   В некоторых случаях программа проверяет, **состоит ли пользователь в нужной группе** (например, `wheel` или `root`).  

4. **Проверка через `prctl()`**  
   Некоторые приложения используют:  
   ```c
   prctl(PR_GET_SECUREBITS, 0, 0, 0, 0);
   ```
   Если `SECURE_NOROOT` установлен, программа **отказывается работать без `root`**.  

---

### **Как обойти проверку?**
Есть несколько вариантов.  

#### **1. Использовать `setcap`, если программе нужны привилегии**  
Если программа требует `root`, потому что **работает с сетью, монтирует файловые системы или управляет процессами**, можно дать ей нужные `capabilities`:  
```bash
setcap CAP_NET_BIND_SERVICE+ep ./rmondl
```
или  
```bash
setcap CAP_SYS_ADMIN+ep ./rmondl
```
Потом проверь:  
```bash
getcap ./rmondl
```

#### **2. Запустить с `fakeroot`**  
Иногда помогает запуск через `fakeroot`:  
```bash
fakeroot ./rmondl
```
Но это работает **только если программа не использует `geteuid()`**.  

#### **3. Подменить `geteuid()` с помощью `LD_PRELOAD`**  
Можно заставить `geteuid()` всегда возвращать `0` (как будто программа запущена от `root`).  
Создаём `fake_root.c`:  
```c
#define _GNU_SOURCE
#include <unistd.h>

uid_t geteuid(void) {
    return 0; // Всегда говорим, что это root
}

uid_t getuid(void) {
    return 0;
}
```
Компилируем:  
```bash
gcc -shared -fPIC -o fakeroot.so fake_root.c
```
И запускаем программу с подменённой библиотекой:  
```bash
LD_PRELOAD=./fakeroot.so ./rmondl
```
**Важно!** Если программа **жёстко слинкована статически** (`file ./rmondl` показывает `statically linked`), `LD_PRELOAD` **не сработает**.  

#### **4. Запуск через `unshare` (изоляция пространства имен)**  
Некоторые программы проверяют UID **в глобальном пространстве имён**, но можно запустить в новом пространстве:  
```bash
unshare -U -r ./rmondl
```
Здесь `-U` создаёт новое пространство имён пользователей, а `-r` подменяет реальный UID.  

#### **5. Патчить бинарник (жёсткое решение)**  
Если ничего не помогает, можно найти проверку `geteuid()` в бинарнике и **подменить её на `nop` (00 00 00 00)**.  
**Шаги:**  
1. Ищем `geteuid` в бинарнике:  
   ```bash
   strings -tx ./rmondl | grep geteuid
   ```
2. Дизассемблируем:  
   ```bash
   objdump -d ./rmondl | less
   ```
3. Ищем условный переход (`jne` или `jz`) после `geteuid()`  
4. Меняем `jne` (`75 XX`) на `nop` (`90 90`):  
   ```bash
   hexedit ./rmondl
   ```
5. Сохраняем и пробуем запустить.  

**⚠️ ВНИМАНИЕ:**  
- Это **грубый хак**, его могут обнаружить.  
- Лучше попробуй `LD_PRELOAD` или `setcap`.  

---

### **Вывод**  
1. **Программа проверяет, кто её запускает, скорее всего, через `geteuid()`**.  
2. **Обойти проверку можно через `setcap`, `LD_PRELOAD`, `unshare` или патчинг бинарника**.  
3. **Лучший вариант – дать только нужные привилегии (`CAP_NET_BIND_SERVICE`, `CAP_SYS_ADMIN`) и запускать в контейнере.**  

Попробуй `setcap` или `LD_PRELOAD`, скажи, что получилось! 🚀

# Отчёт по выполнению домашнего задания: Безопасность и администрирование в Linux

**Выполнил:** Илья (Шифр / Табельный номер: `my_number_M265200`)  
**Операционная система:** Rocky Linux 10  
**Инструментарий:** Bash, POSIX-стандартные утилиты  

---

## Описание выполненных работ

### Раздел 1. Создание пользователя
1. Создана специализированная группа `students`:
   ```bash
   sudo groupadd students
   ```
2. Создан обычный пользователь `user1` с фиксированным UID `1234` и основной группой `students`:
   ```bash
   sudo useradd -u 1234 -g students user1
   ```
3. Настроена политика безопасности для пароля (требование смены каждые 90 дней):
   ```bash
   sudo chage -M 90 user1
   ```

---

### Раздел 2. Мониторинг файлов и процессов

#### 2.1. Мониторинг файлов
Выполнен поиск всех файлов файловой системы с установленным set-UID битом:
```bash
sudo find / -type f -perm -4000 2>/dev/null > ~/suid_files.out
```

#### 2.2. Мониторинг процессов
Осуществлён поиск процессов, у которых эффективный UID равен `0` (root), а реальный UID принадлежит обычному пользователю ($RUID \ne 0$). 
Для демонстрации в одном терминале была запущена команда `passwd` от не-root пользователя, а во втором выполнен перехват состояния:
```bash
ps -eo ruid,euid,pid,comm | awk '$1 != 0 && $2 == 0' > ~/suid_processes.out
```

---

### Раздел 3. Изучение механизма set-UID

#### 3.1. Выбор утилиты и привилегированной операции
* **Выбранная утилита:** `/usr/bin/cat`
* **Привилегированная операция:** Чтение защищённого файла `/etc/shadow` (содержит хэши паролей пользователей, недоступен обычным пользователям).

#### 3.2. Настройка и выполнение
1. Скопирована утилита `cat` в директорию пользователя `user1`:
   ```bash
   sudo mkdir -p /home/user1/bin
   sudo cp /usr/bin/cat /home/user1/bin/mycat
   ```
2. Владельцем файла назначен `root`, установлен SUID-бит (`u+s` / `4755`):
   ```bash
   sudo chown root:root /home/user1/bin/mycat
   sudo chmod 4755 /home/user1/bin/mycat
   sudo chmod 755 /home/user1 /home/user1/bin
   ```
3. Успешно проверено чтение файла `/etc/shadow` от лица `user1` без вызова `sudo`:
   ```bash
   sudo -u user1 /home/user1/bin/mycat /etc/shadow
   ```

---

### Раздел 4. Изучение механизма привилегий (Capabilities)

#### 4.1. Выбор утилиты и привилегированной операции
* **Выбранная утилита:** `/usr/bin/chown`
* **Привилегированная операция:** Изменение владельца произвольного файла в системе.
* **Применяемая привилегия:** `CAP_CHOWN`

#### 4.2. Настройка и выполнение
1. Скопирована утилита `chown` в директорию пользователя `user1`:
   ```bash
   sudo cp /usr/bin/chown /home/user1/bin/mychown
   sudo chown user1:students /home/user1/bin/mychown
   ```
2. Утилите `mychown` назначена точечная привилегия `cap_chown`:
   ```bash
   sudo setcap cap_chown=+ep /home/user1/bin/mychown
   ```
3. Выполнено успешное изменение владельца файла от имени `user1`:
   ```bash
   touch /tmp/testfile.txt
   sudo -u user1 /home/user1/bin/mychown admin /tmp/testfile.txt
   ```

---

### Раздел 5. Изучение механизма sudo

Предоставлено право пользователю `user1` изменять системное время через `sudo` без ввода пароля root.

1. Создан конфигурационный файл в `/etc/sudoers.d/user1_time`:
   ```bash
   echo "user1 ALL=(ALL) NOPASSWD: /usr/bin/date, /usr/bin/timedatectl" | sudo tee /etc/sudoers.d/user1_time
   sudo chmod 0440 /etc/sudoers.d/user1_time
   ```
2. Синтаксис проверен с помощью утилиты `visudo`:
   ```bash
   sudo visudo -c
   ```
3. Проверено изменение времени пользователем `user1`:
   ```bash
   sudo -u user1 sudo date -s "2026-10-27 15:00:00"
   ```
4. Восстановлена синхронизация времени службы NTP:
   ```bash
   sudo systemctl restart chronyd
   ```

---

## Содержимое артефактов

В данном репозитории представлены следующие отчетные файлы:

| Файл | Описание |
| :--- | :--- |
| `mephi-screenshot.png` | Скриншот рабочего стола с выведенным индивидуальным шифром |
| `history.out` | Дамп истории команд терминала |
| `stat.out` | Вывод команды `stat` для директории `/home/user1` и утилит `mycat`, `mychown` |
| `getcap.out` | Результат проверки привилегий Linux Capabilities утилиты `mychown` |
| `suid_files.out` | Список всех найденных SUID-файлов в системе |
| `suid_processes.out` | Результат отслеживания SUID-процессов во время смены пароля |

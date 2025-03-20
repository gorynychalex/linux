Systemd — это мощная система инициализации и управления службами в Linux, которая предоставляет гибкие инструменты для выполнения процессов как по расписанию, так и в ответ на определённые события в системе. Для этих целей используются юниты systemd, такие как `.timer` для задач по расписанию и `.path`, `.socket`, `.device` для реакций на события. Давайте разберём это подробнее.

### 1. Выполнение процессов по расписанию с помощью systemd.timer
Systemd позволяет запускать задачи по расписанию через юниты типа `.timer`, которые работают в паре с юнитами `.service`. Это альтернатива классическому `cron`, но с большей интеграцией в систему и возможностью управления зависимостями.

#### Как это работает:
- Создаётся два файла: 
  - `.service` — описывает, что именно нужно выполнить.
  - `.timer` — задаёт расписание запуска этого сервиса.

#### Пример: запуск скрипта раз в час
1. Создайте файл сервиса, например `/etc/systemd/system/myscript.service`:
   ```
   [Unit]
   Description=Запуск моего скрипта

   [Service]
   ExecStart=/usr/local/bin/myscript.sh
   Type=oneshot
   ```
   Здесь `Type=oneshot` указывает, что процесс выполняется однократно и завершается.

2. Создайте файл таймера `/etc/systemd/system/myscript.timer`:
   ```
   [Unit]
   Description=Запуск myscript каждый час

   [Timer]
   OnCalendar=hourly
   Persistent=true

   [Install]
   WantedBy=timers.target
   ```
   - `OnCalendar=hourly` — запуск каждый час.
   - `Persistent=true` — если система была выключена в момент срабатывания, задача выполнится при следующем включении.

3. Активируйте таймер:
   ```
   sudo systemctl daemon-reload
   sudo systemctl enable myscript.timer
   sudo systemctl start myscript.timer
   ```

#### Гибкость расписания
`OnCalendar` поддерживает множество форматов, например:
- `OnCalendar=*-*-* 14:30:00` — каждый день в 14:30.
- `OnCalendar=Mon *-*-* 09:00:00` — каждый понедельник в 9:00.

### 2. Реакция на события в системе
Systemd позволяет запускать процессы в ответ на события, такие как обращение к файлу, открытие сетевого порта или подключение устройства. Для этого используются юниты `.path`, `.socket` и `.device`.

#### a) Запуск процесса при обращении к файлу (systemd.path)
Юнит `.path` отслеживает изменения в файлах или каталогах (аналог `inotify`).

Пример: запуск скрипта при изменении файла `/var/log/example.log`.

1. Создайте сервис `/etc/systemd/system/logcheck.service`:
   ```
   [Unit]
   Description=Проверка логов

   [Service]
   ExecStart=/usr/local/bin/checklog.sh
   Type=oneshot
   ```

2. Создайте файл `/etc/systemd/system/logcheck.path`:
   ```
   [Unit]
   Description=Отслеживание изменений в example.log

   [Path]
   PathModified=/var/log/example.log

   [Install]
   WantedBy=multi-user.target
   ```
   - `PathModified` — срабатывает при изменении файла.

3. Активируйте:
   ```
   sudo systemctl daemon-reload
   sudo systemctl enable logcheck.path
   sudo systemctl start logcheck.path
   ```

#### b) Реакция на сетевой порт (systemd.socket)
Юнит `.socket` позволяет запускать сервис при обращении к определённому сетевому порту.

Пример: запуск скрипта при подключении к порту 12345.

1. Создайте сервис `/etc/systemd/system/portlisten.service`:
   ```
   [Unit]
   Description=Обработка подключения к порту

   [Service]
   ExecStart=/usr/local/bin/portscript.sh
   StandardInput=socket
   Type=oneshot
   ```

2. Создайте сокет `/etc/systemd/system/portlisten.socket`:
   ```
   [Unit]
   Description=Слушаем порт 12345

   [Socket]
   ListenStream=12345

   [Install]
   WantedBy=sockets.target
   ```

3. Активируйте:
   ```
   sudo systemctl daemon-reload
   sudo systemctl enable portlisten.socket
   sudo systemctl start portlisten.socket
   ```

#### c) Реакция на подключение съёмного носителя (systemd.device)
Для реакции на подключение устройств можно использовать юнит `.device` в сочетании с правилами `udev` и сервисом.

Пример: запуск скрипта при подключении USB-накопителя.

1. Создайте сервис `/etc/systemd/system/usbdetect.service`:
   ```
   [Unit]
   Description=Обработка подключения USB
   BindsTo=dev-disk-by-uuid-XXXX-XXXX.device
   After=dev-disk-by-uuid-XXXX-XXXX.device

   [Service]
   ExecStart=/usr/local/bin/usbscript.sh
   Type=oneshot
   ```

2. Узнайте UUID устройства через `lsblk -f` и подставьте его в `BindsTo` и `After`.

3. Активируйте:
   ```
   sudo systemctl daemon-reload
   sudo systemctl enable usbdetect.service
   ```

### Преимущества использования systemd
- Единый подход к управлению задачами и зависимостями.
- Логирование через `journalctl` (`journalctl -u myscript.timer`).
- Возможность перезапуска, ограничения ресурсов и интеграции с другими юнитами.

### Итог
Systemd предоставляет мощные инструменты для автоматизации: `.timer` для расписания, `.path` для файлов, `.socket` для сети и `.device` для устройств. Настройка требует понимания структуры юнитов, но даёт гибкость и надёжность в управлении процессами. Если нужно больше примеров или помощь с конкретным сценарием — дайте знать!

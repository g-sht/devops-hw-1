# Отчёт по домашней работе

## Проблема 1. Неверный `ExecStart` в unit-файле

### Что было обнаружено

Сразу после `setup.sh` сервис не запускался: `systemctl status` показывал, что unit загружен и включён (`enabled`), но основной процесс завершается с `status=203/EXEC`, а systemd по `Restart=on-failure` каждые 2 секунды пытается запустить его заново.

### Как проводилась диагностика

```bash
systemctl status homework-app.service
journalctl -u homework-app.service -n 30 --no-pager
systemctl show homework-app.service -p ExecStart --value
systemctl cat homework-app.service
ls -l /opt/linux-devops-homework
```

- Код `203/EXEC` означает, что systemd не смог выполнить `exec()` указанного в `ExecStart` файла — до запуска самого приложения дело не дошло, поэтому причину надо искать в unit-файле, а не в коде.
- В журнале была запись о том, что исполняемый файл `/opt/linux-devops-homework/app.py` не найден (`No such file or directory`).
- `systemctl show -p ExecStart` показывает точную команду, которую использует systemd (а не то, что лежит в репозитории): `path=/opt/linux-devops-homework/app.py`. `systemctl cat` подтвердил, что значение взято из `/etc/systemd/system/homework-app.service`.
- `ls -l /opt/linux-devops-homework` показал, что файла `app.py` там нет: приложение называется `server.py`.

### В чём была причина

В unit-файле `ExecStart` указывал на несуществующий файл `/opt/linux-devops-homework/app.py`. Приложение называется `server.py`, а по требованию задания (и проверке №1 в `check.sh`) должно запускаться как `/opt/linux-devops-homework/app/server.py`.

### Что было изменено

Файл `systemd/homework-app.service` в репозитории:

```diff
-ExecStart=/opt/linux-devops-homework/app.py
+ExecStart=/opt/linux-devops-homework/app/server.py
```

Изменение применено к системе (в Git не отражается):

```bash
sudo install -m 0644 systemd/homework-app.service /etc/systemd/system/homework-app.service
sudo systemctl daemon-reload
```

### Почему было выбрано это решение

Ошибка `203/EXEC` прямо указывает на путь из `ExecStart`, поэтому исправлять нужно именно его. Путь `/opt/linux-devops-homework/app/server.py` выбран потому, что его требует конечное состояние: `check.sh` проверяет и то, что systemd использует именно этот путь, и то, что он записан в unit-файле репозитория. Правка сделана в репозитории и затем скопирована в `/etc/systemd/system`, чтобы оба места совпадали. `daemon-reload` обязателен — без него systemd продолжает использовать старую версию unit-файла.

### Как проверялся результат

```bash
systemctl show homework-app.service -p ExecStart --value
sudo systemctl restart homework-app.service
systemctl status homework-app.service
```

`systemctl show` стал показывать новый путь, то есть правка unit-файла применилась. При этом сервис по-прежнему падал с `203/EXEC` — но уже по новому пути. Это следующая проблема.

---

## Проблема 2. Скрипт приложения лежит не там, где его ожидает unit

### Что было обнаружено

После исправления `ExecStart` сервис всё ещё завершался с `status=203/EXEC`, в журнале — файл `/opt/linux-devops-homework/app/server.py` не найден.

### Как проводилась диагностика

```bash
journalctl -u homework-app.service -n 30 --no-pager
ls -l /opt/linux-devops-homework /opt/linux-devops-homework/app
```

`ls` показал, что каталога `/opt/linux-devops-homework/app` не существует, а `server.py` лежит прямо в `/opt/linux-devops-homework/`.

### В чём была причина

`setup.sh` устанавливает приложение в `/opt/linux-devops-homework/server.py`, а требуемый `ExecStart` указывает на `/opt/linux-devops-homework/app/server.py`. Файл существует, но не по тому пути.

### Что было изменено

Скрипт вручную перенесён в ожидаемое место 

```bash
sudo install -d -m 0755 /opt/linux-devops-homework/app
sudo mv /opt/linux-devops-homework/server.py /opt/linux-devops-homework/app/server.py
```

Владелец (`root:root`) и права (`0755`) файла при переносе сохранились: пользователь `homework` может его читать и исполнять, но не изменять.

### Почему было выбрано это решение

Вариантов два: поменять путь в `ExecStart` на фактическое расположение файла или перенести файл туда, куда указывает `ExecStart`. Первый не подходит — `check.sh` требует именно `/opt/linux-devops-homework/app/server.py`, а менять `setup.sh` запрещено. Поэтому перенесён файл. Каталог создан с правами `0755` и владельцем `root`, чтобы сервисный пользователь мог только читать и запускать код.

### Как проверялся результат

```bash
ls -l /opt/linux-devops-homework/app
sudo systemctl restart homework-app.service
systemctl status homework-app.service
journalctl -u homework-app.service -n 30 --no-pager
```

Ошибка `203/EXEC` исчезла: теперь интерпретатор запускается, но процесс завершается с `status=1/FAILURE`, а в журнале появился Python traceback. То есть проблема запуска устранена, и следующая ошибка уже внутри приложения.

---

## Проблема 3. Нет прав на запись в каталог данных

### Что было обнаружено

Приложение стартовало и сразу падало с `status=1/FAILURE`. В журнале — traceback, заканчивающийся `PermissionError: [Errno 13] Permission denied: '/var/lib/linux-devops-homework/startup.log'`.

### Как проводилась диагностика

```bash
journalctl -u homework-app.service -n 30 --no-pager
systemctl show homework-app.service -p User -p Group
id homework
namei -l /var/lib/linux-devops-homework
stat /var/lib/linux-devops-homework
```

- Следующая ошибка появилась уже не в статусе systemd, а в журнале сервиса (stderr приложения): оно при старте открывает на дозапись `startup.log` в каталоге данных.
- Сервис работает от `homework:homework` (`User=`/`Group=` в unit-файле).
- `stat` и `namei -l` показали, что `/var/lib/linux-devops-homework` принадлежит `root:root` с правами `0700`. Родительские каталоги `/var` и `/var/lib` доступны на проход всем, значит мешает именно последний каталог.

### В чём была причина

Каталог данных создан `setup.sh` как `root:root 0700`: доступ есть только у root. Пользователь `homework` не может ни войти в каталог, ни создать в нём файл.

### Что было изменено

Состояние системы:

```bash
sudo chown homework:homework /var/lib/linux-devops-homework
sudo chmod 0750 /var/lib/linux-devops-homework
```

### Почему было выбрано это решение

Сервис должен работать от непривилегированного пользователя, поэтому правильно дать доступ именно ему, а не расширять права для всех или запускать приложение от root (и то и другое запрещено условием). Передача каталога в собственность `homework:homework` с правами `0750` даёт владельцу полный доступ, группе — чтение, остальным — ничего. Это минимально достаточные права и ровно то состояние, которое требуется по заданию.

### Как проверялся результат

```bash
stat -c '%U:%G %a' /var/lib/linux-devops-homework
sudo systemctl restart homework-app.service
systemctl status homework-app.service
sudo cat /var/lib/linux-devops-homework/startup.log
```

`stat` показывает `homework:homework 750`, сервис перешёл в `active (running)`, в `startup.log` появилась запись о старте — запись в каталог работает.

---

## Проблема 4. Приложение слушает только loopback

### Что было обнаружено

Сервис активен, `/health` на `127.0.0.1` отвечает, но `check.sh` сообщал, что порт 8080 доступен только через loopback.

### Как проводилась диагностика

```bash
sudo ss -lntp | grep 8080
cat /etc/linux-devops-homework/app.conf
journalctl -u homework-app.service -n 10 --no-pager
```

- `ss -lntp` показывает слушающие TCP-сокеты вместе с процессом-владельцем: сокет был `127.0.0.1:8080`, владелец — `python3` с PID сервиса.
- В журнале приложение само пишет `homework-app: listening on 127.0.0.1:8080`.
- Адрес берётся из `/etc/linux-devops-homework/app.conf`, где было `host = 127.0.0.1`.

### В чём была причина

В конфигурации приложения был задан `host = 127.0.0.1`. Сокет, привязанный к `127.0.0.1`, принимает соединения только с самой машины через интерфейс `lo`; с других хостов сервис недоступен. `0.0.0.0` означает «все локальные IPv4-адреса»: сокет принимает соединения на любом интерфейсе, включая loopback. Поэтому `127.0.0.1:8080` условию не соответствует, а `0.0.0.0:8080` соответствует.

### Что было изменено

Файл `config/app.conf` в репозитории:

```diff
-host = 127.0.0.1
+host = 0.0.0.0
```

Изменение применено к системе:

```bash
sudo install -m 0644 config/app.conf /etc/linux-devops-homework/app.conf
sudo systemctl restart homework-app.service
```

### Почему было выбрано это решение

Адрес привязки определяется только конфигурацией, код приложения менять нельзя и не нужно. Правка сделана в репозитории и скопирована в `/etc`, чтобы сданный файл совпадал с рабочим. Приложение читает конфигурацию один раз при старте, поэтому нужен `restart` (а `daemon-reload` — нет, unit-файл не менялся).

### Как проверялся результат

```bash
sudo ss -lntp | grep 8080
curl -i http://127.0.0.1:8080/health
```

`ss` показывает сокет `0.0.0.0:8080`, `curl` возвращает `HTTP 200` и `{"status": "ok"}`.

---

## Дополнительные наблюдения

- PPID процесса — `1`: сервис запущен напрямую systemd, а не из чьей-то оболочки.
- `/proc/<PID>/status`: в строках `Uid:`/`Gid:` все четыре значения (real, effective, saved, fs) равны UID/GID пользователя `homework` (сверяется с `id homework`) — процесс действительно работает не от root.
- `/proc/<PID>/cmdline`: `/usr/bin/python3 /opt/linux-devops-homework/app/server.py` — ядро запустило интерпретатор из shebang, скрипт взят из пути в `ExecStart`.
- `/proc/<PID>/fd/`: stdout и stderr — сокеты журнала systemd (поэтому вывод приложения виден в `journalctl`), плюс слушающий TCP-сокет. Файл `startup.log` среди дескрипторов отсутствует: приложение открывает его только на время записи при старте.

## Итоговый ход диагностики

1. `systemctl status` → `203/EXEC`: systemd не может запустить файл из `ExecStart`. `systemctl show -p ExecStart` и `ls` показали, что `app.py` не существует → исправлен путь в unit-файле, `daemon-reload`.
2. Снова `203/EXEC`, но уже по новому пути: `server.py` установлен в корень `/opt/linux-devops-homework`, а не в `app/` → файл перенесён вручную.
3. Процесс начал запускаться и падать с `1/FAILURE`; в `journalctl` — `PermissionError` на `startup.log`. `stat`/`namei` показали `root:root 0700` на каталоге данных → `chown homework:homework`, `chmod 0750`.
4. Сервис стал `active`, но `ss -lntp` показал `127.0.0.1:8080` → в конфигурации `host` заменён на `0.0.0.0`, сервис перезапущен.

После каждого шага сервис перезапускался, и проверялось, что исчезла именно наблюдавшаяся ошибка (сменился код завершения или сообщение в журнале), прежде чем переходить к следующей.

## Итоговая проверка

```bash
sudo ./scripts/check.sh
```

```text
Linux DevOps Homework Checker

[PASS] systemd ExecStart points to the application
[PASS] service runs as homework:homework
[PASS] state directory ownership and permissions are correct
[PASS] service is enabled
[PASS] service is active
[PASS] running process has the expected UID
[PASS] TCP/8080 listens on 0.0.0.0
[PASS] GET /health returns the expected response
[PASS] fixed systemd unit is saved in the repository
[PASS] fixed application configuration is saved in the repository

Result: 10 passed, 0 failed
```

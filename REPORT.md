# Отчёт по домашней работе
## Проблема 1. ExecStart указывал на несуществующий файл
### Что было обнаружено
После развёртывания стенда сервис `homework-app.service` у меня не запускался. Команда `systemctl status homework-app.service` показывала `Active: activating (auto-restart) (Result: exit-code)`, а процесс завершался с кодом `status=203/EXEC`. В журнале у меня был виден бесконечный цикл перезапуска (`restart counter is at 64`).
### Как проводилась диагностика
```bash
systemctl status homework-app.service --no-pager
journalctl -u homework-app.service -n 30 --no-pager
systemctl cat homework-app.service
ls -la /opt/linux-devops-homework/
head -1 /opt/linux-devops-homework/server.py
```
Существенные результаты этих проверок:
1.`status=203/EXEC`-systemd не смог выполнить то, что указано в `ExecStart=`.
2.`systemctl cat` показал мне строку `ExecStart=/opt/linux-devops-homework/app.py`.
3.`ls -la /opt/linux-devops-homework/` показал мне, что в каталоге лежит только `server.py`, но файла `app.py` нет.
4.`head -1 server.py` показал shebang `#!/usr/bin/env python3`-файл является исполняемым Python-скриптом.
### В чём была причина
Параметр `ExecStart=` в unit-файле указывал на несуществующий файл `/opt/linux-devops-homework/app.py`. systemd пытался его запустить, но получал `203/EXEC`, падал и перезапускался каждые 2 секунды из-за `Restart=on-failure`.
### Что было изменено
Первоначально в файле `systemd/homework-app.service` строка
```
ExecStart=/opt/linux-devops-homework/app.py
```
была мной заменена на
```
ExecStart=/opt/linux-devops-homework/server.py
```
Затем исправленный unit скопировала в системный каталог и сервис был перезапущен:
```bash
cp systemd/homework-app.service /etc/systemd/system/homework-app.service
systemctl daemon-reload
systemctl restart homework-app.service
```
На четвёртом шагу (смотреть проблему 4) окончательный путь привела к требуемому `/opt/linux-devops-homework/app/server.py`, а сам файл переместила в подкаталог `app/`.
### Почему было выбрано это решение
`203/EXEC` означает, что systemd не смог выполнить команду, указанную в `ExecStart`. Единственный способ исправить-привести `ExecStart` в соответствие с реально существующим файлом. Файл `server.py` имеет shebang `#!/usr/bin/env python3` и бит `+x`, поэтому запускается напрямую, без явного указания интерпретатора. Запуск от `root` и `chmod 777` запрещены правилами задания.
### Как проверялся результат
```bash
systemctl status homework-app.service --no-pager
journalctl -u homework-app.service -n 30 --no-pager -l
```
Ошибка `203/EXEC` исчезла, сервис перешёл в состояние `active (running)`. В журнале появилась новая ошибка-`PermissionError` при записи `startup.log`. Этого я ожидала, что устранение одной проблемы открыло следующую.
---
## Проблема 2. Нет прав на запись в каталог данных
### Что было обнаружено
После исправления `ExecStart` приложение у меня стартовало, но завершалось с кодом `status=1/FAILURE`. В журнале у меня был traceback Python:
```
PermissionError: [Errno 13] Permission denied: '/var/lib/linux-devops-homework/startup.log'
```
### Как проводилась диагностика
```bash
journalctl -u homework-app.service -n 30 --no-pager -l
ls -ld /var/lib/linux-devops-homework
stat -c '%U %G %a' /var/lib/linux-devops-homework
```
Существенные результаты этих проверок:
1.Каталог `/var/lib/linux-devops-homework` имел владельца и группу `root root` и права `700` (`drwx------`).
2.В unit-файле указано `User=homework` и `Group=homework`.
Таким образом, сервис работал под `homework`, а каталог принадлежал `root` и не давал прав на запись никому, кроме владельца.
### В чём была причина
`setup.sh` создал каталог данных от имени `root` с правами `0700`. Сервис запускается под пользователем `homework` и при попытке открыть `/var/lib/linux-devops-homework/startup.log` на запись получал `Permission denied`, из-за чего приложение падало.
### Что было изменено
Права и владелец каталога данных я изменила командами:
```bash
chown homework:homework /var/lib/linux-devops-homework
chmod 0750 /var/lib/linux-devops-homework
systemctl restart homework-app.service
```
Проверка:
```bash
stat -c '%U %G %a' /var/lib/linux-devops-homework
```
Результат: `homework homework 750`.
### Почему было выбрано это решение
По условию 3 из README каталог `/var/lib/linux-devops-homework` должен принадлежать `homework:homework` и иметь права `0750`. Права `0750` (`rwxr-x---`) дают владельцу полный доступ (включая запись), группе-чтение и вход, остальным-ничего. Это минимально необходимый набор. `chmod 777` и запуск сервиса от `root` запрещены правилами.
### Как проверялся результат
```bash
ls -ld /var/lib/linux-devops-homework
systemctl status homework-app.service --no-pager
journalctl -u homework-app.service -n 20 --no-pager -l
```
Сервис перешёл в `active (running)`, `PermissionError` больше не появляется, в логе-`listening on 127.0.0.1:8080`.Следующая проблема, что сервис слушает только loopback.
---
## Проблема 3. Приложение слушало только loopback
### Что было обнаружено
Сервис работал, `/health` локально отвечал `200 {"status": "ok"}`, но:
```bash
ss -tlnp | grep 8080
```
показывало мне
```
LISTEN 0 5 127.0.0.1:8080 0.0.0.0:* users:(("python3",pid=1363,fd=3))
```
По условию 6 из README сервис должен слушать `0.0.0.0:8080`-все IPv4-интерфейсы.
### Как проводилась диагностика
```bash
ss -tlnp | grep 8080
journalctl -u homework-app.service -n 20 --no-pager -l
cat config/app.conf
cat /etc/linux-devops-homework/app.conf
curl -i http://127.0.0.1:8080/health
```
Существенные результаты этих проверок:
1.В логе: `listening on 127.0.0.1:8080`.
2.В обоих конфигах-`host = 127.0.0.1`.
3.`curl` локально отвечает `200 {"status": "ok"}`-значит, что само приложение работает корректно, проблема только в адресе привязки.
### В чём была причина
В конфиге приложения был указан bind-адрес `127.0.0.1`-это loopback-интерфейс, доступный только с самой машины. Такой сервис не соответствует требованию слушать на всех IPv4-интерфейсах. Для этого нужен `0.0.0.0`.
### Что было изменено
В файле `config/app.conf` строка
```
host = 127.0.0.1
```
заменена на
```
host = 0.0.0.0
```
Затем конфиг скопирован в системный каталог и сервис перезапущен:
```bash
cp config/app.conf /etc/linux-devops-homework/app.conf
systemctl restart homework-app.service
```
Правились оба файла: `config/app.conf`-чтобы изменение попало в Git (условие 8), `/etc/linux-devops-homework/app.conf`-потому что приложение читает именно его.
### Почему было выбрано это решение
`0.0.0.0` означает привязку к IPv4-адресам всех доступных сетевых интерфейсов. `127.0.0.1`-только loopback. Условие 6 задания требует `0.0.0.0`.
### Как проверялся результат
```bash
ss -tlnp | grep 8080
curl -i http://127.0.0.1:8080/health
journalctl -u homework-app.service -n 5 --no-pager -l
```
Результат:
1.`ss`->`LISTEN 0.0.0.0:8080`
2.`curl`->`HTTP/1.0 200 OK`, `{"status": "ok"}`
3.в логе->`listening on 0.0.0.0:8080`
---
## Проблема 4. Структура приложения и unit не соответствовали ожидаемому состоянию
### Что было обнаружено
После закрытия трёх предыдущих проблем я запустила итоговую проверку:
```bash
sudo ./scripts/check.sh
```
Она показала два `FAIL`:
```
[FAIL] systemd ExecStart is incorrect
[FAIL] repository still contains the original broken unit
```
При этом сервис работал, слушал `0.0.0.0:8080` и отвечал на `/health`. Из двух FAIL следовало, что проверка ожидает другой путь в `ExecStart` и другую структуру каталогов приложения.
### Как проводилась диагностика
```bash
ls -la /opt/linux-devops-homework/
cat systemd/homework-app.service
cat scripts/check.sh
```
Существенные результаты этих проверок:
1.`check.sh` вернул два `FAIL`: по `ExecStart` и по unit-файлу в репозитории.
2.`ls -la /opt/linux-devops-homework/` показал мне, что `server.py` лежит в корне каталога, а не в подкаталоге `app/`.
3.В unit-файле указан `ExecStart=/opt/linux-devops-homework/server.py`-без подкаталога `app/`.
4.В `check.sh` ожидается строка `ExecStart=/opt/linux-devops-homework/app/server.py`-и именно её сравнивает проверка для репозиторного и системного unit-файлов.
### В чём была причина
`check.sh` явно проверяет строку `ExecStart=/opt/linux-devops-homework/app/server.py` и в системном unit, и в репозиторном. Приложение в системе должно быть в подкаталоге `app/`. На тот момент ни путь в unit, ни структура каталогов этому не соответствовали.
### Что было изменено
Создан подкаталог `app/`, файл я перенесла:
```bash
mkdir -p /opt/linux-devops-homework/app
mv /opt/linux-devops-homework/server.py /opt/linux-devops-homework/app/server.py
```
В `systemd/homework-app.service` строку я привела к виду:
```
ExecStart=/opt/linux-devops-homework/app/server.py
```
Затем:
```bash
cp systemd/homework-app.service /etc/systemd/system/homework-app.service
systemctl daemon-reload
systemctl restart homework-app.service
```
### Почему было выбрано это решение
`check.sh` однозначно указывает ожидаемое состояние: путь `/opt/linux-devops-homework/app/server.py` и совпадение системного и репозиторного unit-файлов. Файл имеет shebang и бит `+x`, поэтому корректно запускается без явного `python3` в `ExecStart`.
### Как проверялся результат
```bash
systemctl status homework-app.service --no-pager
ss -tlnp | grep 8080
curl -i http://127.0.0.1:8080/health
sudo ./scripts/check.sh
```
Сервис `active (running)`, `ss`->`0.0.0.0:8080`, `curl`->`200 {"status": "ok"}`, `check.sh`->`Result: 10 passed, 0 failed`.
---
## Дополнительные наблюдения
**Пользователь, от имени которого работает сервис.** В unit-файле указаны:
```
User=homework
Group=homework
```
Итоговая проверка `check.sh` подтвердила мне, что сервис работает от имени `homework:homework`, а UID процесса соответствует ожидаемому.
**Состояние процесса.** После исправления unit-файла сервис находится в состоянии `active (running)`. В выводе `systemctl status homework-app.service` видно, что приложение запущено как Python-процесс с путём `/opt/linux-devops-homework/app/server.py`.
**Сетевой сокет.** Команда `ss -tlnp | grep 8080` показала, что приложение слушает `0.0.0.0:8080`. Это означает, что сервер принимает подключения на всех IPv4-интерфейсах. До исправления конфигурации он слушал только `127.0.0.1:8080`, поэтому был доступен только локально.
**Проверка HTTP.** Команда `curl -i http://127.0.0.1:8080/health` вернула `HTTP 200 OK` и ожидаемый ответ `{"status": "ok"}`. Это подтвердило, что приложение работает после всех исправлений.
**Права на каталог данных.** Изначально `/var/lib/linux-devops-homework` принадлежал `root:root` и имел права `700`. После исправления владелец и группа стали `homework:homework`, а права-`750`. Это позволило приложению создавать `startup.log`, не предоставляя лишних прав другим пользователям.
**Маршрутизация.** Для дополнительного анализа сетевой конфигурации можно использовать команду `ip route get 1.1.1.1`. Она позволяет определить, через какой интерфейс и шлюз система отправила бы пакет к указанному адресу. В рамках основной проверки задания это не требовалось, поэтому данный тест отдельно не использовался для получения итогового результата.
---
## Итоговое содержимое изменённых файлов
### systemd/homework-app.service
```
[Unit]
Description=Linux DevOps Homework Service
After=network.target
[Service]
Type=simple
User=homework
Group=homework
ExecStart=/opt/linux-devops-homework/app/server.py
Restart=on-failure
RestartSec=2
[Install]
WantedBy=multi-user.target
```
### config/app.conf
```
[server]
host = 0.0.0.0
port = 8080
```
---
## Итоговый ход диагностики

1. `systemctl status homework-app.service`->`status=203/EXEC`, рестарт-цикл (restart counter растёт).
2. `systemctl cat homework-app.service` + `ls -la /opt/linux-devops-homework/`->`ExecStart` указывает на `app.py`, которого нет, что в каталоге только `server.py`.
3. Правка unit: `ExecStart`->`server.py`. После `daemon-reload` + `restart` сервис запустился, но появилась новая ошибка.
4. `journalctl -u homework-app.service`->`PermissionError` на `startup.log`. `ls -ld` и `stat` каталога-> `root root 700`.
5. `chown homework:homework`, `chmod 0750`, `restart`-> сервис снова запустился, но `ss` показал `127.0.0.1:8080`.
6. `cat config/app.conf` и `/etc/linux-devops-homework/app.conf`-> `host = 127.0.0.1`. Правка на `0.0.0.0`, копирование в `/etc/...`, `restart`->`ss` показывает `0.0.0.0:8080`.
7. `check.sh`->два `FAIL` по `ExecStart` и по unit в репозитории. Приложение перенесено в `app/`, путь в unit приведён к `/opt/linux-devops-homework/app/server.py`, unit синхронизирован с системным.
8. `sudo ./scripts/check.sh`->`Result: 10 passed, 0 failed`.
Каждая правка открывала следующую проблему. Но в итоге всё заработало.
---
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

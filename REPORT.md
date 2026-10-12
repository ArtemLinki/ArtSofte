
## Проблема 1. Сервис не запускался

### Что было обнаружено

Сервис homework-app находился в состоянии activating (auto-restart) и постоянно перезапускался.

### Как проводилась диагностика

Укажите команды и действия, которыми вы получили информацию о проблеме, а также существенные результаты этих проверок.

```bash
systemctl status homework-app.service
systemctl cat homework-app
```

Вывод показал:
```text
Loaded: loaded (/etc/systemd/system/homework-app.service; enabled; preset: enabled)
Active: activating (auto-restart) (Result: exit-code)
Process: 7600 ExecStart=/opt/linux-devops-homework/app.py (code=exited, status=203/EXEC)
Main PID: 7600 (code=exited, status=203/EXEC)
```

### В чём была причина

В ExecStart был указан несуществующий файл /opt/linux-devops-homework/app.py, поэтому systemd не мог запустить приложение, ошибка status=203/EXEC

### Что было изменено

Отредактировал unit-файл homework-app.service, а именно поправил путь в ExecStart, чтобы он указывал на реально существующий файл

### Почему было выбрано это решение

Сначала я посмотрел, что именно пытается загрузить systemd, и обнаружил, что такого файла там нет, но есть другой с другим именем. Я предположил, что в ExecStart неправильно указано имя. Попытался изменить системный unit, но не получилось из-за нехватки прав, поэтому поменял имя в homework-app.service в репозитории, там прав хватало

### Как проверялся результат

```bash
systemctl status homework-app.service
```

Вывод дал уже другую ошибку:
```text
homework/server.py (code=exited, status=1/FAILURE)
```

То есть ошибка запуска ушла, и появилась ошибка уже внутри самого приложения

## Проблема 2. Падение приложения: PermissionError на startup.log

### Что было обнаружено

```text
[FAIL] state directory is root:root mode 700; expected homework:homework mode 750
PermissionError: [Errno 13] Permission denied: '/var/lib/linux-devops-homework/startup.log'
```

Приложение запускалось, но сразу завершалось с status=1/FAILURE и перезапускалось по кругу.

### Как проводилась диагностика


```bash
journalctl -u homework-app -n 30
namei -l /var/lib/linux-devops-homework
stat /var/lib/linux-devops-homework
```

Выяснилось, что каталог /var/lib/linux-devops-homework принадлежит root:root, а надо homework.

### В чём была причина

В unit-файле не было директивы StateDirectory, которая заставила бы systemd создать каталог с владельцем homework.

### Что было изменено

В homework-app.service добавил:
```ini
StateDirectory=linux-devops-homework
StateDirectoryMode=0750
```

### Почему было выбрано это решение

systemd делает владельцем каталога пользователя из User и выставляет права из StateDirectoryMode. Так как этой директивы не было, каталог остался принадлежать root. 
### Как проверялся результат

```bash
journalctl -u homework-app -n 30
```

PermissionError больше не выводится. Чекер показал отсутствие ошибки по правам на state-каталог.

## Проблема 3. Сервис слушал только 127.0.0.1:8080

### Что было обнаружено

Сервис слушал только 127.0.0.1:8080, а надо, чтобы он слушал на всех IPv4-интерфейсах, то есть 0.0.0.0:8080.

### Как проводилась диагностика

```bash
cat /etc/linux-devops-homework/app.conf
```

В конфиге приложения host был выставлен в 127.0.0.1.

### В чём была причина

Приложение было настроено слушать только loopback интерфейс.

### Что было изменено

В config/app.conf поменял:
```ini
host = 0.0.0.0
```

### Почему было выбрано это решение

В задании явно указано, что приложение должно слушать порт 8080 на 0.0.0.0. Значение 0.0.0.0 включает в себя и loopback, поэтому проверка GET http://127.0.0.1:8080/health продолжает работать.

### Как проверялся результат

Чекер показал отсутствие ошибки: [PASS] TCP/8080 listens on 0.0.0.0.

## Дополнительные наблюдения

## Итоговый ход диагностики

1. 203/EXEC, неверный путь в ExecStart.
2. 1/FAILURE с PermissionError, каталог состояния принадлежал root, понадобилась директива StateDirectory=.
3. Сервис слушал только loopback, понадобилась правка host в конфиге на 0.0.0.0.

## Итоговая проверка

Вставьте полный вывод успешного запуска:

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
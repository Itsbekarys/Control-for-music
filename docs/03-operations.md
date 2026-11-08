# Как я обслуживаю сервер

## Перед командами

Эти команды относятся к серверу, на котором уже есть полный проект RESONANCE. В этом репозитории лежит документация, а не установочный комплект приложения.

В примерах проект находится в `/home/ubuntu/resonance-monitor`. На другой машине заменяю путь и пользователя. Имена служб сначала сверяю с фактической конфигурацией.

## 1. Сначала выясняю, что вообще запущено

```bash
systemctl list-unit-files 'resonance-*'
systemctl show resonance-monitor resonance-alerts -p ExecStart -p WorkingDirectory
systemctl status resonance-monitor resonance-alerts --no-pager
sudo journalctl -u resonance-monitor -u resonance-alerts -n 100 --no-pager
sudo ss -lntp
```

Для серверного API ожидаю приложение за Nginx, с внутренним портом на loopback. В обсуждавшейся конфигурации это `127.0.0.1:8080`.

Telegram-служба в новой схеме выключена. Но главное, чтобы активный обработчик тоже не вызывал старую отправку сообщений. Проверяю код и ExecStart, а не только знакомое имя службы.

## 2. Сверяю фактическую политику и свежесть отчётов

Базу для диагностики открываю в режиме read-only. Так опечатка в пути не создаст новую пустую monitor.db.

```bash
cd /home/ubuntu/resonance-monitor
.venv/bin/python - <<'PY'
import sqlite3
import time
from contextlib import closing
from pathlib import Path

path = Path('monitor.db').resolve()
with closing(sqlite3.connect(path.as_uri() + '?mode=ro', uri=True)) as db:
    db.row_factory = sqlite3.Row
    print('Целостность:', db.execute('PRAGMA quick_check').fetchone()[0])
    print('Политика:', dict(db.execute('SELECT * FROM music_policy WHERE id=1').fetchone()))
    for row in db.execute('SELECT hostname, last_seen FROM computers'):
        age = None if row['last_seen'] is None else round(time.time() - row['last_seen'])
        print(row['hostname'], 'секунд с последнего отчёта:', age)
PY
```

Вывод содержит имена компьютеров. Его использую для своей диагностики, а перед публикацией обезличиваю. Токены и весь payload в общий лог не выгружаю.

## 3. Проверяю HTTPS и доступность API

Снаружи:

```bash
curl -I https://monitor.example.com/login
```

На Windows-клиенте:

```powershell
Resolve-DnsName monitor.example.com
Test-NetConnection monitor.example.com -Port 443
```

Эти проверки подтверждают отдельные части цепочки. Успешный TCP 443 ещё не означает успешную авторизацию агента.

Если диагностирую heartbeat, смотрю HTTP-код в журнале агента:

| Код | Что проверяю |
| --- | --- |
| 400 | JSON, типы полей, диапазоны значений |
| 401 | Bearer-токен и его соответствие компьютеру |
| 403 | hostname и назначение учётной записи компьютера |
| 404 | Домен, путь API и настройки прокси |
| 502 | Запущен ли Waitress и совпадает ли внутренний порт |

Это подсказки для диагностики, а не исчерпывающий список причин каждого кода.

## 4. Как выглядит запуск веб-части

Ниже пример unit для полного проекта. Он не устанавливает зависимости, не создаёт config.json и не включает отсутствующие исходники.

```ini
[Unit]
Description=RESONANCE Monitor web application
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/resonance-monitor
ExecStart=/home/ubuntu/resonance-monitor/.venv/bin/waitress-serve --listen=127.0.0.1:8080 --url-scheme=https app:app
Restart=on-failure
RestartSec=5
UMask=0077
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

Здесь `app:app` означает объект app в модуле app.py. Параметр url-scheme подходит для этой схемы с HTTPS на Nginx и доступом к Waitress только через loopback. Синтаксис есть в [документации Waitress](https://docs.pylonsproject.org/projects/waitress/en/stable/runner.html).

Обработчик состояний запускается отдельной службой. Его фактическое имя файла узнаю через ExecStart. В локальных материалах новая версия сохранена как `alerts_new.py`, на сервере она могла быть установлена под другим именем.

## 5. Nginx перед приложением

Внутри уже настроенного HTTPS-сервера Nginx проксирование может выглядеть так:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_connect_timeout 10s;
    proxy_read_timeout 60s;
}
```

Это фрагмент location, не полный конфиг Nginx. server_name, сертификат, ключ, HTTP-редирект и продление сертификата настраиваются отдельно.

Внешний API и панель использую по HTTPS. В доступном app.py сессионная cookie помечена Secure, поэтому обычный HTTP-вход может вести себя иначе, чем рабочая HTTPS-схема.

После изменения прокси:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 6. Резервная копия SQLite

Просто копировать живой файл базы вслепую не хочу. Для отдельного согласованного снимка использую SQLite backup API, поддерживаемый [модулем sqlite3](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.backup).

Минимальный пример ниже сохраняет только базу. Он не заменяет полный бэкап проекта:

```bash
cd /home/ubuntu/resonance-monitor
.venv/bin/python - <<'PY'
import os
import sqlite3
from contextlib import closing
from datetime import datetime, timezone
from pathlib import Path

os.umask(0o077)
source_path = Path('monitor.db').resolve()
destination = Path.home() / 'resonance-backups'
destination.mkdir(mode=0o700, exist_ok=True)
stamp = datetime.now(timezone.utc).strftime('%Y%m%dT%H%M%S%fZ')
backup_path = destination / f'monitor-{stamp}.db'

with closing(sqlite3.connect(source_path.as_uri() + '?mode=ro', uri=True)) as source:
    with closing(sqlite3.connect(backup_path)) as target:
        source.backup(target)
        result = target.execute('PRAGMA integrity_check').fetchone()[0]
        if result != 'ok':
            raise RuntimeError('Копия не прошла проверку целостности')

print('Снимок базы готов:', backup_path)
PY
```

В полном бэкапе мне нужны база, код, зависимости, config.json, история файлов при её использовании, настройки systemd и Nginx. Секреты и сертификаты храню защищённо и восстанавливаю по отдельному проверенному плану.

На пилоте настроено ежедневное создание проверяемого архива примерно в 04:00 по UTC+5, с небольшим случайным сдвигом. Последние 30 архивов сохраняются. Это состояние из предыдущей проверки, перед изменениями я снова смотрю timer и журнал:

```bash
systemctl list-timers 'resonance-backup*' --all
sudo journalctl -u resonance-backup.service -n 50 --no-pager
```

Копия на том же VPS помогает при ошибке приложения, но не при потере всего сервера. Поэтому следующий шаг у меня: защищённая копия вне VPS и проверка полного восстановления.

## 7. План переноса на Debian и Docker

Перенос пока описываю как план. Полного проверенного Docker-комплекта в этом репозитории нет.

Главные вещи, которые сохраняю:

1. Адрес, который уже прописан у агентов.
2. Пути `/api/heartbeat` и `/api/audio-tabs`.
3. monitor.db с идентичностью компьютеров и хешами токенов.
4. Клиентские токены и секреты приложения.
5. Рабочий HTTPS и доверие клиентов к сертификату.

Сначала смотрю, используется домен или прямой IP. Если домен, можно перенести сервер и переключить DNS. Если в клиентах зашит IP, одной сменой DNS ничего не исправить: потребуется сохранить адрес или обновить клиентскую конфигурацию.

На новом Debian Docker устанавливаю по [официальной инструкции](https://docs.docker.com/engine/install/debian/), без предположения, что имена пакетов одинаковы во всех версиях ОС.

Особенность текущего кода: путь к monitor.db в нескольких модулях вычисляется относительно файла Python. Одна переменная `DATABASE=/data/monitor.db` сама это не изменит. Сначала надо централизовать путь к базе в коде, затем подключать постоянное хранилище и проверять все модули.

Другой нюанс: контейнер веб-приложения не заменяет фоновый обработчик состояний. Обоим процессам нужен согласованный доступ к одной базе и одинаковой политике.

Мой порядок миграции:

1. Создаю согласованную копию и фиксирую версии зависимостей.
2. Поднимаю отдельную копию приложения без отправки реальных уведомлений.
3. Проверяю вход, токены и API тестовым агентом.
4. Настраиваю HTTPS и фоновые проверки.
5. В окно переключения останавливаю запись на старом экземпляре.
6. Переношу финальный снимок базы, запускаю новый экземпляр и переключаю адрес.
7. Проверяю возвращение heartbeat и расширения.

DNS-кэши могут некоторое время вести клиентов на старый адрес. Если нужен переход без потери отчётов, старый вход на этот период должен проксировать запросы на новый backend, либо миграция должна предусматривать другой согласованный способ доставки.

Откат тоже планирую заранее. Просто вернуть старый DNS после новых записей на новом сервере недостаточно: базы уже могли разойтись. В таком случае сначала останавливаю запись и выбираю согласованный источник данных.

## 8. Чего не кладу в публичный GitHub

* monitor.db, её WAL/SHM-файлы и бэкапы.
* config.json с рабочими секретами.
* Токены агентов, расширения и Telegram-бота.
* SSH-ключи и приватные ключи сертификатов.
* Реальные адреса магазинов, имена ПК и диагностические выгрузки.

Для примеров использую отдельные синтетические значения. Если секрет уже попал в историю Git, удаление строки в следующем коммите его не отзывает: ключ надо заменить.

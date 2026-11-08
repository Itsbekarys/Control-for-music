# ��� � ���������� ������

## ����� ���������

��� ������� ��������� � �������, �� ������� ��� ���� ������ ������ RESONANCE. � ���� ����������� ����� ������������, � �� ������������ �������� ����������.

� �������� ������ ��������� � `/home/ubuntu/resonance-monitor`. �� ������ ������ ������� ���� � ������������. ����� ����� ������� ������ � ����������� �������������.

## 1. ������� �������, ��� ������ ��������

```bash
systemctl list-unit-files 'resonance-*'
systemctl show resonance-monitor resonance-alerts -p ExecStart -p WorkingDirectory
systemctl status resonance-monitor resonance-alerts --no-pager
sudo journalctl -u resonance-monitor -u resonance-alerts -n 100 --no-pager
sudo ss -lntp
```

��� ���������� API ������ ���������� �� Nginx, � ���������� ������ �� loopback. � ������������� ������������ ��� `127.0.0.1:8080`.

Telegram-������ � ����� ����� ���������. �� �������, ����� �������� ���������� ���� �� ������� ������ �������� ���������. �������� ��� � ExecStart, � �� ������ �������� ��� ������.

## 2. ������ ����������� �������� � �������� �������

���� ��� ����������� �������� � ������ read-only. ��� �������� � ���� �� ������� ����� ������ monitor.db.

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
    print('�����������:', db.execute('PRAGMA quick_check').fetchone()[0])
    print('��������:', dict(db.execute('SELECT * FROM music_policy WHERE id=1').fetchone()))
    for row in db.execute('SELECT hostname, last_seen FROM computers'):
        age = None if row['last_seen'] is None else round(time.time() - row['last_seen'])
        print(row['hostname'], '������ � ���������� ������:', age)
PY
```

����� �������� ����� �����������. ��� ��������� ��� ����� �����������, � ����� ����������� �����������. ������ � ���� payload � ����� ��� �� ��������.

## 3. �������� HTTPS � ����������� API

�������:

```bash
curl -I https://monitor.example.com/login
```

�� Windows-�������:

```powershell
Resolve-DnsName monitor.example.com
Test-NetConnection monitor.example.com -Port 443
```

��� �������� ������������ ��������� ����� �������. �������� TCP 443 ��� �� �������� �������� ����������� ������.

���� ������������ heartbeat, ������ HTTP-��� � ������� ������:

| ��� | ��� �������� |
| --- | --- |
| 400 | JSON, ���� �����, ��������� �������� |
| 401 | Bearer-����� � ��� ������������ ���������� |
| 403 | hostname � ���������� ������� ������ ���������� |
| 404 | �����, ���� API � ��������� ������ |
| 502 | ������� �� Waitress � ��������� �� ���������� ���� |

��� ��������� ��� �����������, � �� ������������� ������ ������ ������� ����.

## 4. ��� �������� ������ ���-�����

���� ������ unit ��� ������� �������. �� �� ������������� �����������, �� ������ config.json � �� �������� ������������� ���������.

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

����� `app:app` �������� ������ app � ������ app.py. �������� url-scheme �������� ��� ���� ����� � HTTPS �� Nginx � �������� � Waitress ������ ����� loopback. ��������� ���� � [������������ Waitress](https://docs.pylonsproject.org/projects/waitress/en/stable/runner.html).

���������� ��������� ����������� ��������� �������. ��� ����������� ��� ����� ����� ����� ExecStart. � ��������� ���������� ����� ������ ��������� ��� `alerts_new.py`, �� ������� ��� ����� ���� ����������� ��� ������ ������.

## 5. Nginx ����� �����������

������ ��� ������������ HTTPS-������� Nginx ������������� ����� ��������� ���:

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

��� �������� location, �� ������ ������ Nginx. server_name, ����������, ����, HTTP-�������� � ��������� ����������� ������������� ��������.

������� API � ������ ��������� �� HTTPS. � ��������� app.py ���������� cookie �������� Secure, ������� ������� HTTP-���� ����� ����� ���� �����, ��� ������� HTTPS-�����.

����� ��������� ������:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 6. ��������� ����� SQLite

������ ���������� ����� ���� ���� ������� �� ����. ��� ���������� �������������� ������ ��������� SQLite backup API, �������������� [������� sqlite3](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.backup).

����������� ������ ���� ��������� ������ ����. �� �� �������� ������ ����� �������:

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
            raise RuntimeError('����� �� ������ �������� �����������')

print('������ ���� �����:', backup_path)
PY
```

� ������ ������ ��� ����� ����, ���, �����������, config.json, ������� ������ ��� � �������������, ��������� systemd � Nginx. ������� � ����������� ����� ��������� � �������������� �� ���������� ������������ �����.

�� ������ ��������� ���������� �������� ������������ ������ �������� � 04:00 �� UTC+5, � ��������� ��������� �������. ��������� 30 ������� �����������. ��� ��������� �� ���������� ��������, ����� ����������� � ����� ������ timer � ������:

```bash
systemctl list-timers 'resonance-backup*' --all
sudo journalctl -u resonance-backup.service -n 50 --no-pager
```

����� �� ��� �� VPS �������� ��� ������ ����������, �� �� ��� ������ ����� �������. ������� ��������� ��� � ����: ���������� ����� ��� VPS � �������� ������� ��������������.

## 7. ���� �������� �� Debian � Docker

������� ���� �������� ��� ����. ������� ������������ Docker-��������� � ���� ����������� ���.

������� ����, ������� ��������:

1. �����, ������� ��� �������� � �������.
2. ���� `/api/heartbeat` � `/api/audio-tabs`.
3. monitor.db � ������������� ����������� � ������ �������.
4. ���������� ������ � ������� ����������.
5. ������� HTTPS � ������� �������� � �����������.

������� ������, ������������ ����� ��� ������ IP. ���� �����, ����� ��������� ������ � ����������� DNS. ���� � �������� ����� IP, ����� ������ DNS ������ �� ���������: ����������� ��������� ����� ��� �������� ���������� ������������.

�� ����� Debian Docker ������������ �� [����������� ����������](https://docs.docker.com/engine/install/debian/), ��� �������������, ��� ����� ������� ��������� �� ���� ������� ��.

����������� �������� ����: ���� � monitor.db � ���������� ������� ����������� ������������ ����� Python. ���� ���������� `DATABASE=/data/monitor.db` ���� ��� �� �������. ������� ���� �������������� ���� � ���� � ����, ����� ���������� ���������� ��������� � ��������� ��� ������.

������ �����: ��������� ���-���������� �� �������� ������� ���������� ���������. ����� ��������� ����� ������������� ������ � ����� ���� � ���������� ��������.

��� ������� ��������:

1. ������ ������������� ����� � �������� ������ ������������.
2. �������� ��������� ����� ���������� ��� �������� �������� �����������.
3. �������� ����, ������ � API �������� �������.
4. ���������� HTTPS � ������� ��������.
5. � ���� ������������ ������������ ������ �� ������ ����������.
6. �������� ��������� ������ ����, �������� ����� ��������� � ���������� �����.
7. �������� ����������� heartbeat � ����������.

DNS-���� ����� ��������� ����� ����� �������� �� ������ �����. ���� ����� ������� ��� ������ �������, ������ ���� �� ���� ������ ������ ������������ ������� �� ����� backend, ���� �������� ������ ��������������� ������ ������������� ������ ��������.

����� ���� �������� �������. ������ ������� ������ DNS ����� ����� ������� �� ����� ������� ������������: ���� ��� ����� ���������. � ����� ������ ������� ������������ ������ � ������� ������������� �������� ������.

## 8. ���� �� ����� � ��������� GitHub

* monitor.db, � WAL/SHM-����� � ������.
* config.json � �������� ���������.
* ������ �������, ���������� � Telegram-����.
* SSH-����� � ��������� ����� ������������.
* �������� ������ ���������, ����� �� � ��������������� ��������.

��� �������� ��������� ��������� ������������� ��������. ���� ������ ��� ����� � ������� Git, �������� ������ � ��������� ������� ��� �� ��������: ���� ���� ��������.

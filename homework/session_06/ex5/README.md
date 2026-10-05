# Bai 5: Thiet ke va Cau hinh dich vu Systemd cho ung dung Java

## 1. Thong tin
- Repo: `session06-homework-ex5`
- Nop: `homework/session_06/ex5/java-app.service` + `homework/session_06/ex5/README.md`
- Moi truong: Ubuntu 22.04, systemd 249

## 2. Cac lenh da thuc hien

### Buoc 1: Tao user he thong khong login
```bash
sudo useradd -r -s /usr/sbin/nologin java-runner
id java-runner
# uid=998(java-runner) gid=998(java-runner) groups=998(java-runner)
```

Giai thich: `-r` = system account (uid < 1000), `-s /usr/sbin/nologin` = chan SSH truc tiep, giam attack surface. App bi hack cung khong co shell.

### Buoc 2: Tao file service
```bash
sudo nano /etc/systemd/system/java-app.service
# dan noi dung file java-app.service (xem file kem theo)
sudo cat /etc/systemd/system/java-app.service
```

Noi dung (mo phong app that bang sleep, thay bang java -jar khi deploy that):
```ini
[Unit]
Description=Java App Service - Spring Boot demo
After=network.target

[Service]
Type=simple
User=java-runner
Group=java-runner
WorkingDirectory=/opt/java-app
ExecStart=/usr/bin/sleep 1000
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

### Buoc 3: Reload + start + enable
```bash
sudo systemctl daemon-reload
sudo systemctl start java-app.service
sudo systemctl enable java-app.service
sudo systemctl status java-app.service --no-pager
journalctl -u java-app.service -n 20 --no-pager
ps aux | grep sleep
```

## 3. Bang chung ket qua

```bash
$ sudo systemctl status java-app.service
* java-app.service - Java App Service - Spring Boot demo
     Loaded: loaded (/etc/systemd/system/java-app.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2026-10-05 10:30:00 +07; 1min ago
   Main PID: 3456 (sleep)
      Tasks: 1 (limit: 4618)
     Memory: 176.0K
        CPU: 1ms
     CGroup: /system.slice/java-app.service
             `-3456 /usr/bin/sleep 1000

$ journalctl -u java-app.service -n 10 --no-pager
Oct 05 10:30:00 ubuntu-vps systemd[1]: Started Java App Service - Spring Boot demo.

$ ps aux | grep sleep
java-runner  3456  0.0  0.0   8076   640 ?  Ss   10:30   0:00 /usr/bin/sleep 1000
# Chay dung user java-runner -> dat yeu cau bao mat
```

Test tu restart khi sap:
```bash
sudo kill -9 3456
sleep 12
sudo systemctl status java-app.service
# Active: active (running) voi PID moi -> Restart=on-failure + RestartSec=10 hoat dong
```

## 4. Giai thich
- `[Unit] After=network.target`: cho network len xong moi start app.
- `[Service] User=java-runner`: khong chay root. Neu app co lo hong RCE, attacker chi co quyen java-runner.
- `Restart=on-failure`: chi restart khi exit code != 0 (crash). Khong restart khi stop chu dong (`systemctl stop`).
- `RestartSec=10`: doi 10s truoc khi restart de tranh loop spam CPU.
- `WantedBy=multi-user.target`: tu start khi boot may.
- Nang cao that cho Java: them `Environment=JAVA_OPTS="-Xmx512m"`, `SuccessExitStatus=143` (vi java tra 143 khi SIGTERM), `LimitNOFILE=65536`.

# Сессионный проект — Операционные системы Unix (Fedora 43)

> Туториал для выполнения проекта с нуля на виртуальной машине (Mac + Parallels)

---

## Подготовка: виртуальная машина

### Что скачать
- **Parallels Desktop**: https://www.parallels.com
- **Fedora Server 43 ISO**: https://fedoraproject.org/server/download
  - Apple Silicon (M1/M2/M3) → выбирайте `aarch64`
  - Intel Mac → выбирайте `x86_64`

### Настройка ВМ в Parallels
1. Создайте новую ВМ → выберите ISO Fedora
2. RAM: **2048 MB**, CPU: **2 ядра**
3. Диск 1: **64 GB** (система) — создаётся автоматически
4. Сеть: **Shared Network**
5. После установки добавьте Диск 2: **5 GB**
   - Выключите ВМ → настройки → Hardware → **+** → Hard Disk → 5 GB

### Установка Fedora
1. Запустите ВМ → выберите **Install Fedora Server**
2. Software Selection → **Minimal Install**
3. Задайте пароль пользователя
4. После установки перезагрузитесь

### Включение копипаста между Маком и Fedora
Parallels → настройки ВМ → Options → Sharing → включите Shared Clipboard

---

## Подключение по SSH (удобнее работать с Мака)

Внутри Fedora:
```bash
sudo dnf install -y openssh-server
sudo systemctl enable --now sshd
ip addr show    # запомните IP
```

На Маке в Terminal:
```bash
ssh parallels@<IP_федоры>
```

---

## Раздел 1 — Управление сетью (10 баллов)

### 1.1 Статический IP и hostname (5 баллов)

Узнайте имя подключения:
```bash
nmcli con show
# Запомните NAME — например "Profile 1"
```

Настройте статический IP:
```bash
sudo nmcli con mod "Profile 1" ipv4.addresses 192.168.1.100/24
sudo nmcli con mod "Profile 1" ipv4.gateway 192.168.1.1
sudo nmcli con mod "Profile 1" ipv4.dns 8.8.8.8
sudo nmcli con mod "Profile 1" ipv4.method manual
sudo nmcli con up "Profile 1"
```

> ⚠️ После последней команды SSH отключится — переподключитесь по новому IP

Установите hostname:
```bash
sudo hostnamectl set-hostname mephi-2026.domain.local
```

Проверьте:
```bash
ip addr show
hostnamectl | grep hostname
ip route show
```

### 1.2 Проверка сетевой связности (5 баллов)

```bash
ping -c 4 8.8.8.8 > /tmp/network_check.txt
ping -c 4 192.168.1.1 >> /tmp/network_check.txt
cat /tmp/network_check.txt
```

> ⚠️ В Parallels Shared Network шлюз 192.168.1.1 может быть недоступен — это нормально для ВМ

---

## Раздел 2 — Программное обеспечение (10 баллов)

### 2.1 Установка пакетов (5 баллов)

```bash
sudo dnf install -y nginx tcpdump libcap-ng-utils
```

Проверьте:
```bash
nginx -v
tcpdump --version
getcap --version
```

### 2.2 Скачать RPM и установить вручную (5 баллов)

```bash
sudo dnf download --destdir /tmp tcpdump
ls /tmp/tcpdump*.rpm
sudo rpm -ivh --reinstall /tmp/tcpdump-*.rpm
```

---

## Раздел 3 — Файловые системы и сервисы (10 баллов)

### 3.1 Создание раздела и монтирование (5 баллов)

Убедитесь что второй диск виден:
```bash
lsblk
# Должен быть /dev/sdb
```

Создайте раздел:
```bash
sudo fdisk /dev/sdb
```

Внутри fdisk введите по очереди:
```
n
Enter
Enter
Enter
Enter
w
```

Форматирование и монтирование:
```bash
sudo mkfs.ext4 -L MEPHI_DATA /dev/sdb1
sudo mkdir -p /data/mephi-web
echo 'LABEL=MEPHI_DATA /data/mephi-web ext4 defaults 0 2' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h /data/mephi-web
```

### 3.2 Сервис nginx (5 баллов)

```bash
sudo systemctl enable --now nginx
sudo systemctl status nginx
journalctl -u nginx --since "5 minutes ago" > /tmp/nginx_recent_logs.txt
```

---

## Раздел 4 — Управление доступом (10 баллов)

### 4.1 Пользователи, группы, права DAC (5 баллов)

```bash
sudo useradd mephi-admin
echo 'P@ssw0rd2026' | sudo passwd --stdin mephi-admin
sudo groupadd mephi-devs
sudo usermod -aG mephi-devs mephi-admin
sudo chown mephi-admin:mephi-devs /data/mephi-web
sudo chmod 2775 /data/mephi-web
ls -ld /data/mephi-web
# Должно быть: drwxrwsr-x ... mephi-admin mephi-devs
```

### 4.2 SELinux и Capabilities (5 баллов)

```bash
# Проверьте режим SELinux
getenforce
# Должно быть: Enforcing

# Если не Enforcing — включите
sudo setenforce 1
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config

# Установите пакет для управления контекстами
sudo dnf install -y policycoreutils-python-utils

# Разрешите nginx читать файлы из /data/mephi-web
sudo semanage fcontext -a -t httpd_sys_content_t "/data/mephi-web(/.*)?"
sudo restorecon -Rv /data/mephi-web

# Проверьте контекст
ls -Zd /data/mephi-web

# Настройте capabilities для tcpdump
sudo setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump
getcap /usr/sbin/tcpdump
# Должно быть: /usr/sbin/tcpdump cap_net_admin,cap_net_raw=eip

# Проверьте что обычный пользователь может запустить
sudo -u mephi-admin tcpdump --help
```

---

## Раздел 5 — Аутентификация (10 баллов)

### 5.1 Запрет входа root (5 баллов)

```bash
# Запрет через SSH
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo grep PermitRootLogin /etc/ssh/sshd_config
sudo systemctl restart sshd

# Запрет через локальную консоль
echo 'root' | sudo tee /etc/deniedusers
sudo chmod 600 /etc/deniedusers
echo 'auth required pam_listfile.so item=user sense=deny file=/etc/deniedusers onerr=succeed' | sudo tee -a /etc/pam.d/login
```

### 5.2 Создание страницы и проверка (5 баллов)

Настройте nginx на директорию /data/mephi-web:
```bash
sudo sed -i 's|root.*html;|root /data/mephi-web;|' /etc/nginx/nginx.conf
sudo nginx -t
sudo systemctl reload nginx
```

Создайте страницу (замените 12345 на ваш номер зачётной книжки):
```bash
echo "Hello from Student: 12345" | sudo tee /data/mephi-web/index.html
```

Проверьте:
```bash
curl http://localhost
# Ожидаемый результат: Hello from Student: 12345
```

---

## Раздел 6 — Артефакты для GitHub

### 6.1 Сохраните все файлы

```bash
history > ~/project_history.txt
ping -c 4 192.168.1.1 > ~/network_check.txt
ping -c 4 8.8.8.8 >> ~/network_check.txt
journalctl -u nginx --since "5 minutes ago" > ~/nginx_recent_logs.txt
cp /etc/fstab ~/fstab.txt
getenforce > ~/selinux_status.txt
ls -Zd /data/mephi-web > ~/file_contexts.txt
getcap /usr/sbin/tcpdump > ~/tcpdump_capabilities.txt
stat /data/mephi-web > ~/permissions.txt
id mephi-admin > ~/users_groups.txt
getent group mephi-devs >> ~/users_groups.txt
curl -s http://localhost > ~/curl_output.txt
cp /data/mephi-web/index.html ~/index.html
cp /tmp/tcpdump-*.rpm ~/tcpdump.rpm
```

### 6.2 Скриншот

Введите в терминале и сделайте скриншот (Cmd+Shift+4 на Маке):
```bash
curl http://localhost
```

Сохраните как `mephi-nginx-screenshot.png`

### 6.3 Скопируйте файлы на Мак

На Маке в Terminal:
```bash
mkdir ~/Desktop/mephi-project
scp "parallels@<IP_федоры>:~/*" ~/Desktop/mephi-project/
```

### 6.4 Создайте репозиторий на GitHub

1. Зайдите на https://github.com → New repository
2. Название: **mephi-session-project-2026**
3. Тип: **Public**
4. Add file → Upload files → загрузите все файлы из папки mephi-project
5. Commit changes

---

## Итоговая проверка

```bash
echo "=== HOSTNAME ===" && hostnamectl | grep hostname && \
echo "=== IP ===" && ip addr show | grep "inet 192" && \
echo "=== NGINX ===" && systemctl is-enabled nginx && \
echo "=== DISK ===" && df -h /data/mephi-web && \
echo "=== SELINUX ===" && getenforce && \
echo "=== TCPDUMP ===" && getcap /usr/sbin/tcpdump && \
echo "=== WEBSERVER ===" && curl -s http://localhost
```

Ожидаемый результат:
```
=== HOSTNAME ===
   Static hostname: mephi-2026.domain.local
=== IP ===
    inet 192.168.1.100/24 ...
=== NGINX ===
enabled
=== DISK ===
/dev/sdb1  4.9G  ...  /data/mephi-web
=== SELINUX ===
Enforcing
=== TCPDUMP ===
/usr/sbin/tcpdump cap_net_admin,cap_net_raw=eip
=== WEBSERVER ===
Hello from Student: 12345
```

## Чек-лист перед сдачей

- [ ] `hostnamectl` показывает mephi-2026.domain.local
- [ ] `ip addr` показывает статический IP
- [ ] `df -h /data/mephi-web` показывает примонтированный раздел
- [ ] `systemctl is-enabled nginx` выводит `enabled`
- [ ] `getenforce` выводит `Enforcing`
- [ ] `getcap /usr/sbin/tcpdump` выводит capabilities
- [ ] `curl http://localhost` выводит `Hello from Student: <ВАШ_НОМЕР>`
- [ ] root не может войти через SSH
- [ ] GitHub репозиторий публичный и содержит все файлы

# Проект CloudClass: Техническая документация по узлу №1

## 1. Роль узла и архитектура

**Роль в инфраструктуре:** Маршрутизатор, файловое хранилище, балансировщик нагрузки, источник метрик мониторинга  
**Операционная система:** Debian 12 (Bookworm) Minimal  
**Физический интерфейс:** `enx00e04c2247b0` (USB-адаптер Ethernet)  
**Сетевая архитектура:** Единый физический интерфейс логически разделён на четыре подсети с использованием IP-алиасов. Изоляция трафика и маршрутизация обеспечиваются правилами `nftables`. Доступ в интернет для виртуальных машин и контейнеров организован через NAT.

**Распределение подсетей:**
- `10.23.68.226/23` — Внешний интерфейс WAN (корпоративная/учебная сеть)
- `10.0.10.1/24` — Зона управления (SSH, административный доступ)
- `10.0.20.1/24` — Зона VDI (трафик удалённых рабочих столов, точка входа HAProxy)
- `10.0.30.1/24` — Зона Docker (веб-порталы, контейнеризованные сервисы)
- `10.0.40.1/24` — Зона хранения (NFS-экспорты, общие ресурсы Samba)

---

## 2. Пошаговая реализация

### 2.1. Подготовка базовой системы
1. Выполнена минимальная установка Debian 12.
2. Обновлены списки пакетов и установлены базовые утилиты:
```bash
apt update && apt upgrade -y
apt install -y curl wget git htop vim net-tools nftables tcpdump firmware-realtek firmware-linux nfs-kernel-server samba haproxy prometheus-node-exporter software-properties-common zfsutils-linux
```
3. Отключён NetworkManager для предотвращения конфликтов с `ifupdown`:
```bash
systemctl disable --now NetworkManager
```
4. Включена переадресация IPv4-пакетов для маршрутизации и NAT:
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

### 2.2. Настройка файловой системы ZFS (бонусное задание)
1. Подключены репозитории `contrib` и `non-free`, необходимые для работы ZFS:
```bash
add-apt-repository contrib non-free non-free-firmware
apt update
```
2. Установлены модули ядра ZFS и пользовательские утилиты:
```bash
apt install -y linux-headers-$(uname -r) zfs-dkms zfsutils-linux
modprobe zfs
```
3. Настроен лимит кэша ARC для предотвращения чрезмерного потребления оперативной памяти (ограничение до 4 ГБ):
```bash
mkdir -p /etc/modprobe.d
echo "options zfs zfs_arc_max=4294967296" | tee /etc/modprobe.d/zfs.conf
update-initramfs -u
reboot
```
4. Создан пул ZFS на неразмеченном пространстве диска (`/dev/sda3`):
```bash
zpool create tank /dev/sda3
zfs set compression=lz4 tank
zfs set atime=off tank
zfs create tank/iso tank/backups tank/profiles
chmod 755 /tank/iso /tank/backups
chmod 1777 /tank/profiles
```
5. Настроена автоматическая ротация снимков через планировщик cron:
Создан файл `/etc/cron.d/zfs-auto-snapshot` с расписанием ежечасных, ежедневных и еженедельных снимков. Подтверждено, что снимки потребляют 0 байт до момента изменения или удаления блоков данных.

### 2.3. Настройка сети
Отредактирован файл `/etc/network/interfaces` для назначения статического внешнего IP-адреса и четырёх внутренних алиасов:
```conf
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug enx00e04c2247b0
iface enx00e04c2247b0 inet static
    address 10.23.68.226/23
    gateway 10.23.68.1
    dns-nameservers 77.88.8.88 77.88.8.2

    post-up ip addr add 10.0.10.1/24 dev enx00e04c2247b0
    post-up ip addr add 10.0.20.1/24 dev enx00e04c2247b0
    post-up ip addr add 10.0.30.1/24 dev enx00e04c2247b0
    post-up ip addr add 10.0.40.1/24 dev enx00e04c2247b0

    post-up /usr/local/sbin/tc-vdi.sh
```
Конфигурация применена командой `ifup enx00e04c2247b0` после предварительной очистки существующих адресов, что гарантировало корректное выполнение хуков `post-up`.

### 2.4. Ограничение трафика (tc / HTB)
Создан скрипт `/usr/local/sbin/tc-vdi.sh` для ограничения трафика подсети VDI до 3 Мбит/с, предотвращающего насыщение канала:
```bash
#!/bin/bash
IF="enx00e04c2247b0"
VDI_NET="10.0.20.0/24"
LIMIT="3mbit"

tc qdisc del dev $IF root 2>/dev/null
tc qdisc add dev $IF root handle 1: htb default 10
tc class add dev $IF parent 1: classid 1:1 htb rate 100mbit ceil 100mbit
tc class add dev $IF parent 1:1 classid 1:10 htb rate $LIMIT ceil 5mbit burst 15k cburst 15k
tc filter add dev $IF parent 1:0 protocol ip u32 match ip dst $VDI_NET flowid 1:10
tc filter add dev $IF parent 1:0 protocol ip u32 match ip src $VDI_NET flowid 1:10
```
Скрипт сделан исполняемым, работоспособность проверена командой `tc qdisc show dev enx00e04c2247b0`.

### 2.5. Межсетевой экран и NAT (nftables)
Стандартная конфигурация `/etc/nftables.conf` заменена на правила, обеспечивающие изоляцию подсетей, фильтрацию портов и маскарадинг:
- Разрешены установленные и связанные соединения, а также loopback-интерфейс.
- Доступ к управлению ограничен адресом `10.0.10.1`.
- Доступ к NFS (111, 2049) и SMB (139, 445) разрешён только из внутренних подсетей.
- Доступ к RDP (3389) открыт на адресе `10.0.20.1`.
- Настроен `masquerade` на интерфейсе `enx00e04c2247b0` для исходящего интернет-трафика.
- Правила применены командой `nft -f /etc/nftables.conf`, служба включена в автозагрузку.

### 2.6. Службы хранения (NFS и Samba)
**Конфигурация NFS (`/etc/exports`):**
```conf
/tank/iso 10.0.0.0/16(ro,sync,no_subtree_check,no_root_squash)
/tank/backups 10.0.0.0/16(rw,sync,no_subtree_check,no_root_squash)
/tank/profiles 10.0.0.0/16(rw,sync,no_subtree_check,all_squash,anonuid=65534,anongid=65534)
```
**Конфигурация Samba (`/etc/samba/smb.conf`):**
Добавлена шара `[profiles]` с параметрами `force user = nobody` и `force group = nogroup` для обеспечения совместимости прав доступа между виртуальными машинами на базе Linux и Windows.

### 2.7. Балансировщик нагрузки RDP (HAProxy)
Стандартный файл `/etc/haproxy/haproxy.cfg` заменён на конфигурацию, работающую в TCP-режиме:
- Параметр `mode http` изменён на `mode tcp` для сохранения целостности протокола RDP.
- Настроен `frontend rdp_balancer`, привязанный к `10.0.20.1:3389`.
- Настроен `backend vdi_servers` с алгоритмом `balance source` для сохранения сессий и опцией `option tcp-check` для мониторинга доступности.
- Целевые серверы: `10.0.20.11:3389` (Linux VDI) и `10.0.20.12:3389` (Windows VDI).
- Конфигурация проверена командой `haproxy -c -f /etc/haproxy/haproxy.cfg`, служба перезапущена.

### 2.8. Агент мониторинга
Включён `prometheus-node-exporter` для экспорта метрик системы (загрузка ЦП, ОЗУ, дисковых операций, статус пула ZFS, сетевая статистика) на порт 9100. Данные собираются экземпляром Prometheus на узле №3.

---

## 3. Журнал устранения неисправностей

| Ошибка / Симптом | Причина | Решение |
|------------------|---------|---------|
| Ошибка установки `zfsutils-linux` | Пакеты ZFS находятся в репозиториях `contrib`/`non-free`, отключённых по умолчанию в Debian 12. | Выполнена команда `add-apt-repository contrib non-free non-free-firmware`, обновлён кэш пакетов перед установкой. |
| Файл `/etc/modprobe.d/zfs.conf` не найден | Файл не создаётся автоматически; конструкция `sudo echo "..." > файл` не работает из-за перенаправления вывода в непривилегированной оболочке. | Использована команда `mkdir -p /etc/modprobe.d` и конвейер `echo \| sudo tee` для корректного создания и записи файла. |
| Таймеры `zfs-auto-snapshot` отсутствуют | Пакет в Debian 12 не активирует systemd-таймеры автоматически; используется механизм cron. | Создан файл `/etc/cron.d/zfs-auto-snapshot` с расписанием, перезапущена служба `cron`. |
| Ручной запуск `zfs-auto-snapshot frequent` завершается ошибкой | Синтаксис CLI требует указания пути к датасету или глобального флага; утилита интерпретировала "frequent" как имя датасета. | Cron выполняет задачу корректно. Снимки проверены через `zfs list -t snapshot`. Отмечено, что потребление 0B является нормой до изменения данных. |
| `systemctl restart networking` не применяет алиасы | Служба networking в Debian выполняет мягкий перезапуск и пропускает хуки `post-up`, если интерфейс уже активен. | Адреса очищены командой `ip addr flush`, интерфейс переведён в состояние DOWN, после чего выполнен `ifup` для принудительного применения хуков. |
| Ошибка `ifup`: "File exists" и "tc-vdi.sh not found" | Ошибка RTNETLINK из-за существующих назначений IP; `ifup` прерван из-за отсутствия исполняемого скрипта в `post-up`. | Выполнены `ip addr flush` и `ip link set down`. Скрипт `/usr/local/sbin/tc-vdi.sh` воссоздан через heredoc, установлены права на выполнение, повторно выполнен `ifup`. |
| `tc qdisc show` возвращает пустой вывод | Команда выполнена без прав root или отфильтрована некорректно. | Проверка выполнена через `sudo tc qdisc show dev enx00e04c2247b0`. Подтверждена активность дисциплины `htb`. |
| Логи HAProxy: "backend vdi_servers has no server available!" | Целевые ВМ (`10.0.20.11`, `10.0.20.12`) ещё не развёрнуты. Механизм health-check корректно фиксирует их статус как DOWN. | Подтверждено прослушивание порта `10.0.20.1:3389` через `ss -tlnp`. Зарегистрировано как ожидаемое поведение до запуска ВМ на узле №2. |

---

## 4. Итоговый чек-лист проверки

Выполните следующие команды для валидации состояния узла №1:

```bash
# Сетевые интерфейсы и алиасы
ip -br addr show enx00e04c2247b0

# Доступ в интернет и NAT
ping -c 3 8.8.8.8

# Статус пула ZFS и компрессия
zpool status tank
zfs get compression tank

# Экспорты NFS
showmount -e localhost

# Валидация конфигурации Samba
testparm -s

# Статус прослушивателя HAProxy
ss -tlnp | grep 3389

# Проверка ограничения трафика
tc qdisc show dev enx00e04c2247b0

# Доступность эндпоинта метрик
curl -s http://localhost:9100/metrics | head -n 5
```

Все компоненты должны возвращать успешный статус без критических ошибок.

---

## 5. Структура репозитория

Файлы конфигурации и документация хранятся в системе контроля версий:

```
cloudclass-coursework/
├── nodes/
│   └── node1-debian-router/
│       ├── interfaces              # Настройка сети с алиасами
│       ├── nftables.conf           # Правила фаервола, NAT и изоляция подсетей
│       ├── tc-vdi.sh               # Скрипт ограничения трафика HTB для зоны VDI
│       ├── exports                 # Определения экспортов NFS
│       ├── smb-profiles.conf       # Конфигурация шары Samba
│       ├── haproxy.cfg             # Конфигурация TCP-балансировщика
│       ├── zfs-cron                # Расписание автоматических снимков
│       └── README.md               # Роль узла, шаги развёртывания и проверка
├── docs/
│   ├── network-topology.md         # Диаграмма логической архитектуры
│   └── ip-allocation-table.md      # Таблица распределения подсетей и IP
── .gitignore
```

---

## 6. Заключение

Узел №1 успешно развёрнут в качестве центрального компонента маршрутизации, хранения и балансировки нагрузки инфраструктуры CloudClass. Реализация полностью соответствует требованиям задания: логическая изоляция подсетей через IP-алиасы, фильтрация пакетов и NAT посредством nftables, управление пропускной способностью через tc/HTB, централизованное хранение через ZFS/NFS/Samba, а также распределение RDP-сессий через HAProxy. Бонусное задание по настройке ZFS функционирует в полном объёме с включённой компрессией и автоматической ротацией снимков. Все выявленные ошибки конфигурации и runtime-проблемы задокументированы и устранены. Узел готов к интеграции с узлом №2 (хост Proxmox VDI) и узлом №3 (веб-портал Docker).

<br>
<br>

<br>
<br>

----

# CloudClass Project: Technical Documentation for Node 1

## 1. Project Role & Architecture Overview

**Node Role:** Router, Storage Server, Load Balancer, Monitoring Source  
**Operating System:** Debian 12 (Bookworm) Minimal  
**Physical Interface:** `enx00e04c2247b0` (USB-to-Ethernet adapter)  
**Network Architecture:** Single physical interface segmented into four logical subnets using IP aliases. Traffic isolation and routing are enforced via `nftables`. NAT provides internet access for internal virtual machines and containers.

**Subnet Allocation:**
- `10.23.68.226/23` - External WAN interface (corporate/educational network)
- `10.0.10.1/24` - Management Zone (SSH, administrative access)
- `10.0.20.1/24` - VDI Zone (Remote desktop traffic, HAProxy entry point)
- `10.0.30.1/24` - Docker Zone (Web portals, containerized services)
- `10.0.40.1/24` - Storage Zone (NFS exports, Samba shares)

---

## 2. Step-by-Step Implementation

### 2.1. Base System Preparation
1. Installed Debian 12 with minimal package selection.
2. Updated package lists and installed core utilities:
   ```bash
   apt update && apt upgrade -y
   apt install -y curl wget git htop vim net-tools nftables tcpdump firmware-realtek firmware-linux nfs-kernel-server samba haproxy prometheus-node-exporter software-properties-common zfsutils-linux
   ```
3. Disabled NetworkManager to prevent conflicts with `ifupdown`:
   ```bash
   systemctl disable --now NetworkManager
   ```
4. Enabled IPv4 packet forwarding for routing and NAT:
   ```bash
   echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
   sysctl -p
   ```

### 2.2. ZFS File System Configuration (Bonus Task)
1. Enabled `contrib` and `non-free` repositories required for ZFS:
   ```bash
   add-apt-repository contrib non-free non-free-firmware
   apt update
   ```
2. Installed ZFS kernel modules and userspace tools:
   ```bash
   apt install -y linux-headers-$(uname -r) zfs-dkms zfsutils-linux
   modprobe zfs
   ```
3. Configured ARC memory limit to prevent excessive RAM consumption (capped at 4GB):
   ```bash
   mkdir -p /etc/modprobe.d
   echo "options zfs zfs_arc_max=4294967296" | tee /etc/modprobe.d/zfs.conf
   update-initramfs -u
   reboot
   ```
4. Created ZFS pool on unallocated disk space (`/dev/sda3`):
   ```bash
   zpool create tank /dev/sda3
   zfs set compression=lz4 tank
   zfs set atime=off tank
   zfs create tank/iso tank/backups tank/profiles
   chmod 755 /tank/iso /tank/backups
   chmod 1777 /tank/profiles
   ```
5. Configured automatic snapshot rotation via cron:
   Created `/etc/cron.d/zfs-auto-snapshot` with hourly, daily, and weekly schedules. Verified snapshots consume 0 bytes until data blocks are modified or deleted.

### 2.3. Network Configuration
Configured `/etc/network/interfaces` to assign a static external IP and four internal alias IPs:
```conf
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug enx00e04c2247b0
iface enx00e04c2247b0 inet static
    address 10.23.68.226/23
    gateway 10.23.68.1
    dns-nameservers 77.88.8.88 77.88.8.2

    post-up ip addr add 10.0.10.1/24 dev enx00e04c2247b0
    post-up ip addr add 10.0.20.1/24 dev enx00e04c2247b0
    post-up ip addr add 10.0.30.1/24 dev enx00e04c2247b0
    post-up ip addr add 10.0.40.1/24 dev enx00e04c2247b0

    post-up /usr/local/sbin/tc-vdi.sh
```
Applied configuration using `ifup enx00e04c2247b0` after flushing existing addresses to ensure `post-up` hooks executed correctly.

### 2.4. Traffic Shaping (tc / HTB)
Created `/usr/local/sbin/tc-vdi.sh` to limit VDI subnet traffic to 3 Mbit/s, preventing bandwidth saturation:
```bash
#!/bin/bash
IF="enx00e04c2247b0"
VDI_NET="10.0.20.0/24"
LIMIT="3mbit"

tc qdisc del dev $IF root 2>/dev/null
tc qdisc add dev $IF root handle 1: htb default 10
tc class add dev $IF parent 1: classid 1:1 htb rate 100mbit ceil 100mbit
tc class add dev $IF parent 1:1 classid 1:10 htb rate $LIMIT ceil 5mbit burst 15k cburst 15k
tc filter add dev $IF parent 1:0 protocol ip u32 match ip dst $VDI_NET flowid 1:10
tc filter add dev $IF parent 1:0 protocol ip u32 match ip src $VDI_NET flowid 1:10
```
Made executable and verified with `tc qdisc show dev enx00e04c2247b0`.

### 2.5. Firewall & NAT (nftables)
Replaced default configuration in `/etc/nftables.conf` with rules enforcing subnet isolation, port filtering, and NAT masquerading:
- Allowed established/related connections and loopback.
- Restricted management access to `10.0.10.1`.
- Allowed NFS (111, 2049) and SMB (139, 445) only from internal subnets.
- Allowed RDP (3389) on `10.0.20.1`.
- Configured `masquerade` on `enx00e04c2247b0` for outbound internet traffic.
- Applied with `nft -f /etc/nftables.conf` and enabled service.

### 2.6. Storage Services (NFS & Samba)
**NFS Configuration (`/etc/exports`):**
```conf
/tank/iso 10.0.0.0/16(ro,sync,no_subtree_check,no_root_squash)
/tank/backups 10.0.0.0/16(rw,sync,no_subtree_check,no_root_squash)
/tank/profiles 10.0.0.0/16(rw,sync,no_subtree_check,all_squash,anonuid=65534,anongid=65534)
```
**Samba Configuration (`/etc/samba/smb.conf`):**
Added `[profiles]` share with `force user = nobody` and `force group = nogroup` to ensure cross-platform file permission compatibility between Linux and Windows VDI instances.

### 2.7. RDP Load Balancer (HAProxy)
Replaced default `/etc/haproxy/haproxy.cfg` with TCP-mode configuration:
- Changed `mode http` to `mode tcp` to preserve RDP protocol integrity.
- Configured `frontend rdp_balancer` binding to `10.0.20.1:3389`.
- Configured `backend vdi_servers` with `balance source` for session persistence and `option tcp-check` for health monitoring.
- Target servers: `10.0.20.11:3389` (Linux VDI) and `10.0.20.12:3389` (Windows VDI).
- Verified with `haproxy -c -f /etc/haproxy/haproxy.cfg` and restarted service.

### 2.8. Monitoring Agent
Enabled `prometheus-node-exporter` to expose system metrics (CPU, RAM, disk I/O, ZFS pool status, network statistics) on port 9100 for collection by the Prometheus instance on Node 3.

---

## 3. Troubleshooting & Error Resolution Log

| Error / Symptom | Root Cause | Resolution |
|-----------------|------------|------------|
| `apt install zfsutils-linux` failed | ZFS packages reside in `contrib`/`non-free` repositories, disabled by default in Debian 12. | Ran `add-apt-repository contrib non-free non-free-firmware` and updated package cache before installation. |
| `/etc/modprobe.d/zfs.conf` not found | File does not exist by default; `sudo echo "..." > file` fails due to shell redirection running as non-root. | Used `mkdir -p /etc/modprobe.d` and piped echo through `sudo tee` to create and write the file correctly. |
| `zfs-auto-snapshot` timers not found | Debian 12 package does not automatically enable systemd timers; relies on cron. | Created `/etc/cron.d/zfs-auto-snapshot` with proper scheduling entries and restarted `cron` service. |
| Manual `zfs-auto-snapshot frequent` failed | CLI syntax requires global flag or dataset path; utility misinterpreted "frequent" as a dataset name. | Cron executes correctly with proper environment. Verified snapshots via `zfs list -t snapshot`. Noted that 0B usage is normal until data changes. |
| `systemctl restart networking` did not apply aliases | Debian's networking service performs a soft restart and skips `post-up` hooks if interface is already UP. | Flushed IP addresses, brought interface down, and used `ifup enx00e04c2247b0` to force full hook execution. |
| `ifup` failed with "File exists" and "tc-vdi.sh not found" | RTNETLINK error due to existing IP assignments; `ifup` aborted because `post-up` script was missing/empty. | Ran `ip addr flush` and `ip link set down`. Recreated `/usr/local/sbin/tc-vdi.sh` using `cat << 'EOF'` to guarantee exact content, set executable permissions, then ran `ifup`. |
| `tc qdisc show` returned empty | Command run without `sudo` or filtered incorrectly with `grep`. | Verified with `sudo tc qdisc show dev enx00e04c2247b0`. Confirmed `htb` qdisc is active. |
| HAProxy logs: "backend vdi_servers has no server available!" | Target VMs (`10.0.20.11`, `10.0.20.12`) were not yet provisioned. HAProxy health checks correctly reported them as DOWN. | Confirmed HAProxy was listening on `10.0.20.1:3389` via `ss -tlnp`. Documented as expected behavior until Node 2 VMs are online. |

---

## 4. Final Verification Checklist

Execute the following commands to validate Node 1 status:

```bash
# Network interfaces & aliases
ip -br addr show enx00e04c2247b0

# Internet connectivity & NAT
ping -c 3 8.8.8.8

# ZFS pool health & compression
zpool status tank
zfs get compression tank

# NFS exports
showmount -e localhost

# Samba configuration validation
testparm -s

# HAProxy listener status
ss -tlnp | grep 3389

# Traffic shaping verification
tc qdisc show dev enx00e04c2247b0

# Metrics endpoint availability
curl -s http://localhost:9100/metrics | head -n 5
```

All components must return successful states without critical errors.

---

## 5. Repository Structure

Configuration files and documentation are version-controlled in the project repository:

```
cloudclass-coursework/
├── nodes/
│   └── node1-debian-router/
│       ├── interfaces              # Network configuration with aliases
│       ├── nftables.conf           # Firewall rules, NAT, and subnet isolation
│       ├── tc-vdi.sh               # HTB traffic shaping script for VDI subnet
│       ├── exports                 # NFS export definitions
│       ├── smb-profiles.conf       # Samba share configuration
│       ├── haproxy.cfg             # TCP load balancer configuration
│       ├── zfs-cron                # Automated snapshot schedule
│       └── README.md               # Node role, deployment steps, and verification
├── docs/
│   ├── network-topology.md         # Logical architecture diagram
│   ── ip-allocation-table.md      # Subnet and IP assignment reference
└── .gitignore
```

---

## 6. Conclusion

Node 1 has been successfully deployed as the central routing, storage, and load-balancing component of the CloudClass infrastructure. The implementation satisfies all core requirements: logical subnet isolation via IP aliases, stateful packet filtering and NAT via nftables, bandwidth management via tc/HTB, centralized storage via ZFS/NFS/Samba, and RDP session distribution via HAProxy. The bonus ZFS task is fully operational with compression enabled and automated snapshot retention configured. All encountered configuration and runtime errors have been documented and resolved. The node is ready for integration with Node 2 (Proxmox VDI host) and Node 3 (Docker web portal).

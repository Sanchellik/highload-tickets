# 2026-10-12 — базовая настройка хоста Beelink (шаг 0.2, часть 1)

**Цель:** подготовить Beelink как хост виртуализации (KVM/libvirt) и собрать данные о железе, чтобы спланировать VM. Сеть хоста (`br-lab`, шаг 0.3) сознательно не трогали.

## Железо (факты)

| Что | Значение |
|---|---|
| ОС | Ubuntu 26.04 LTS (resolute), Desktop; хост `sanch-GTi15`, пользователь `sanch` |
| CPU | Intel Core Ultra 9 285H, 16 ядер, без Hyper-Threading |
| Типы ядер | P: CPU 0–5 (у каждого свой L2) · E-кластер A: 6–9 (общий L2) · E-кластер B: 10–13 (общий L2) · LP-E: 14–15 |
| RAM | 91 GiB, swap 8 GiB (`/swap.img`) |
| Диск 1 (система) | Crucial CT1000E100SSD8, 931 ГБ, серийник `2533EADB2F9…`: `/boot/efi` 1 ГБ, `/` 372 ГБ, `/home` 558 ГБ |
| Диск 2 (свободен) | Crucial CT1000E100SSD8, 931 ГБ, серийник `2533EADB2CDC`: 71 ч работы, износ 0 %, ошибок нет |

Как проверяли: `lscpu`, `/sys/devices/cpu_core/cpus`, `/sys/devices/cpu_atom/cpus`, `/sys/devices/system/cpu/cpuN/cache/index2/shared_cpu_list`, `free -h`, `df -h`, `lsblk`, `smartctl -a`.

## Что сделано

| Действие | Команда | Зачем | Статус |
|---|---|---|---|
| Обновление и базовые пакеты | `apt full-upgrade`; `openssh-server cpu-checker htop btop git vim tmux linux-tools-* lm-sensors` | SSH — чтобы управлять безголовым стендом с Mac; `cpupower` из linux-tools; `sensors` — следить за перегревом под нагрузкой | сделано |
| Виртуализация | `apt install qemu-system-x86 libvirt-daemon-system libvirt-clients virtinst cloud-image-utils` | гипервизор для VM k3s. Пакета `qemu-kvm` в 26.04 нет — это `qemu-system-x86` (QEMU 10.2.1); вариант `-hwe` не нужен: железо поддерживает KVM в ядре | сделано |
| Доступ к libvirt без sudo | `usermod -aG libvirt,kvm $USER` + перелогин | `virsh` без root. Без перелогина — `Permission denied` на сокете | сделано |
| URI libvirt по умолчанию | `export LIBVIRT_DEFAULT_URI=qemu:///system` в `~/.bashrc` | работать с системными VM, а не с пользовательскими (три слеша обязательны) | сделано |
| Governor `performance` | systemd-юнит `/etc/systemd/system/cpu-performance.service` (`cpupower frequency-set -g performance`, enabled) + `powerprofilesctl set performance` | стабильные частоты → воспроизводимые замеры; переживает перезагрузку. Проверка: 16 × `performance` | сделано |
| Запрет сна | `systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target` | стенд не должен засыпать во время прогонов | команда дана, результат не проверен |
| SSH с Mac | `ssh sanch@<IP>` | работа без монитора | не проверено |

## Решения

**Раскладка ядер (предварительно, до создания VM):**

| VM | Ядра хоста | RAM | Почему |
|---|---|---|---|
| node-2 (сервисы, измеряем) | 6–9 (E-кластер A) | 12 ГБ | целый кластер со своим L2 — без шумных соседей |
| node-3 (Postgres, измеряем) | 10–13 (E-кластер B) | 16 ГБ | то же; одинаковые ядра с node-2 → сравнимые цифры |
| node-1 (k3s server + мониторинг) | 0–2 (P) | 12 ГБ | мониторинг не на измеряемых нодах |
| node-4 (Kafka, позже) | 3–4 (P) | 16 ГБ | резерв |
| хост | 5 (P) + 14–15 (LP-E) | ~8 ГБ | ОС, libvirt |

Без переподписки CPU и RAM; запас RAM ~25 ГБ.

**Диск 2 — под диски VM** (изоляция I/O Postgres от ОС). Отложено до отдельной аккуратной сессии.

## Что выяснили по диску 2

- Не смонтирован, нет в fstab, swap и LVM; `wipefs -n` и `blkid -p` сигнатур не видят.
- Первые 2 МБ — нули, но дальше (смещение `0x204000`) остатки FAT32 — EFI-раздел от прежней установки **Proxmox** (больше не нужен, подтверждено). Записано 353 ГБ.
- **Имена NVMe меняются между загрузками:** в комментариях `/etc/fstab` указано, что при установке `/` был на `nvme0n1p2`, сейчас система на `nvme1n1`. Поэтому к диску 2 обращаемся только по серийнику: `/dev/disk/by-id/nvme-CT1000E100SSD8_2533EADB2CDC`; в fstab — по UUID.

## Дальше

1. Диск 2 — проверка пути по серийнику и `dmsetup ls` (только чтение) → `blkdiscard` → GPT + ext4 с меткой `vms` → `/srv/vms` в fstab по UUID с `nofail` → пул libvirt `vms`. Каждая команда — через путь по серийнику с проверкой серийника перед записью.
2. Проверить сон и SSH с Mac.
3. Шаг 0.3 — сеть `br-lab`.
4. Шаг 0.4 — VM по раскладке выше; перенести шаги отсюда в `infra/host/` скриптом.

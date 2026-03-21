# Kubernetes: Установка кластера через kubeadm

## Содержание

1. [Требования к виртуальным машинам](#1-требования-к-виртуальным-машинам)
2. [Подготовка всех узлов](#2-подготовка-всех-узлов)
3. [Установка containerd](#3-установка-containerd)
4. [Установка kubeadm, kubelet, kubectl](#4-установка-kubeadm-kubelet-kubectl)
5. [Инициализация master-ноды](#5-инициализация-master-ноды)
6. [Подключение worker-нод](#6-подключение-worker-нод)
7. [Установка CNI (Flannel)](#7-установка-cni-flannel)
8. [Проверка работы кластера](#8-проверка-работы-кластера)
9. [Развёртывание приложений](#9-развёртывание-приложений)
10. [Настройка Ingress-контроллера](#10-настройка-ingress-контроллера)
11. [Внутренний DNS и домены *.NN.sirius](#11-внутренний-dns-и-домены-nnsirius)

---

## 1. Требования к виртуальным машинам

### Минимальные параметры

| Нода          | CPU  | RAM  | Диск  |
|---------------|------|------|-------|
| k8s-master    | 2    | 4 GB | 40 GB |
| k8s-worker-1  | 2    | 4 GB | 40 GB |
| k8s-worker-2  | 2    | 4 GB | 40 GB |
| dns-server    | 1    | 2 GB | 20 GB |

### Пример /etc/hosts (добавить на все ВМ)

```
192.168.56.10   k8s-master
192.168.56.11   k8s-worker-1
192.168.56.12   k8s-worker-2
192.168.56.20   dns-server
```

### Создание ВМ через Vagrant (пример Vagrantfile)

```ruby
# Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  {
    "k8s-master"   => "192.168.56.10",
    "k8s-worker-1" => "192.168.56.11",
    "k8s-worker-2" => "192.168.56.12",
  }.each do |name, ip|
    config.vm.define name do |node|
      node.vm.hostname = name
      node.vm.network "private_network", ip: ip
      node.vm.provider "virtualbox" do |vb|
        vb.cpus   = 2
        vb.memory = 4096
      end
    end
  end
end
```

Запуск:
```bash
vagrant up
vagrant ssh k8s-master
```

---

## 2. Подготовка всех узлов

> Выполнять на **всех трёх** нодах (master + два worker).

### 2.1 Отключить swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/' /etc/fstab
```

### 2.2 Настроить модули ядра

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### 2.3 Настроить параметры sysctl

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

## 3. Установка containerd

> Выполнять на **всех трёх** нодах.

```bash
# Установка зависимостей
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Добавление Docker GPG-ключа
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Добавление репозитория Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Установка containerd
sudo apt-get update
sudo apt-get install -y containerd.io

# Генерация конфигурации по умолчанию
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Включить SystemdCgroup (обязательно для kubeadm)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 4. Установка kubeadm, kubelet, kubectl

> Выполнять на **всех трёх** нодах.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

---

## 5. Инициализация master-ноды

> Выполнять **только на k8s-master**.

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.56.10 \
  --pod-network-cidr=10.244.0.0/16 \
  --node-name=k8s-master
```

После успешной инициализации вы увидите в конце вывод вида:
```
Your Kubernetes control-plane has initialized successfully!
...
kubeadm join 192.168.56.10:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

**Сохраните эту команду** — она понадобится для подключения worker-нод.

### Настройка kubectl для текущего пользователя

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Проверка:
```bash
kubectl get nodes
```

---

## 6. Подключение worker-нод

> Выполнять на **k8s-worker-1** и **k8s-worker-2**.

Используйте команду `kubeadm join`, полученную на предыдущем шаге:

```bash
sudo kubeadm join 192.168.56.10:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

Если токен истёк (через 24 часа), сгенерируйте новый на master-ноде:
```bash
kubeadm token create --print-join-command
```

---

## 7. Установка CNI (Flannel)

> Выполнять **только на k8s-master**.

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Подождите 1–2 минуты и проверьте, что все ноды перешли в статус `Ready`:

```bash
kubectl get nodes -o wide
```

Пример ожидаемого вывода:
```
NAME           STATUS   ROLES           AGE   VERSION
k8s-master     Ready    control-plane   5m    v1.29.x
k8s-worker-1   Ready    <none>          3m    v1.29.x
k8s-worker-2   Ready    <none>          3m    v1.29.x
```

---

## 8. Проверка работы кластера

```bash
# Все поды системных namespace должны быть Running
kubectl get pods -A

# Проверить события
kubectl get events -A --sort-by='.lastTimestamp'

# Тестовый деплой
kubectl create deployment nginx-test --image=nginx --replicas=2
kubectl expose deployment nginx-test --port=80 --type=NodePort
kubectl get svc nginx-test
# Зайдите в браузере: http://192.168.56.11:<NodePort>

# Удалить тестовый деплой
kubectl delete deployment nginx-test
kubectl delete svc nginx-test
```

---

## 9. Развёртывание приложений

Применяйте манифесты в следующем порядке.

### PostgreSQL (сначала — база данных)

```bash
kubectl apply -f manifests/postgresql/namespace.yaml
kubectl apply -f manifests/postgresql/secret.yaml
kubectl apply -f manifests/postgresql/pvc.yaml
kubectl apply -f manifests/postgresql/statefulset.yaml
kubectl apply -f manifests/postgresql/service.yaml
```

### Сервис авторизации

```bash
# Сначала соберите и запушьте образ:
# docker build -t <your-registry>/auth-service:latest auth-service/python/
# docker push <your-registry>/auth-service:latest

kubectl apply -f manifests/auth-service/namespace.yaml
kubectl apply -f manifests/auth-service/deployment.yaml
kubectl apply -f manifests/auth-service/service.yaml
```

### GitLab

```bash
kubectl apply -f manifests/gitlab/namespace.yaml
kubectl apply -f manifests/gitlab/pvc.yaml
kubectl apply -f manifests/gitlab/deployment.yaml
kubectl apply -f manifests/gitlab/service.yaml
```

> **Важно:** GitLab требует минимум 4 GB RAM на ноде. Убедитесь, что у worker-нод достаточно ресурсов.

### Grafana

```bash
kubectl apply -f manifests/grafana/namespace.yaml
kubectl apply -f manifests/grafana/deployment.yaml
kubectl apply -f manifests/grafana/service.yaml
```

---

## 10. Настройка Ingress-контроллера

### Установка nginx-ingress

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/baremetal/deploy.yaml
```

Дождитесь готовности:
```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

Узнайте NodePort Ingress-контроллера:
```bash
kubectl get svc -n ingress-nginx
```

Для работы Ingress на 80/443 портах без NodePort установите MetalLB (L2-балансировщик для bare-metal):

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

Затем создайте пул адресов:
```yaml
# metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.56.200-192.168.56.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
```

```bash
kubectl apply -f metallb-config.yaml
```

### Применение Ingress-правил

```bash
kubectl apply -f manifests/ingress/ingress.yaml
```

---

## 11. Внутренний DNS и домены *.NN.sirius

### Почему нужен отдельный DNS-сервер?

Kubernetes Ingress работает на основе HTTP Host-заголовка. Чтобы браузер отправлял правильный заголовок, доменное имя должно резолвиться в IP Ingress-контроллера.

### Вариант A: dnsmasq (рекомендуется для быстрого старта)

Установите на отдельной ВМ (или на хост-машине):

```bash
sudo apt-get install -y dnsmasq
```

Добавьте в `/etc/dnsmasq.conf`:
```
# Все запросы к *.42.sirius резолвятся в IP Ingress
address=/.42.sirius/192.168.56.200

# Если MetalLB не используется — укажите NodePort IP:
# address=/.42.sirius/192.168.56.11
```

Перезапустите:
```bash
sudo systemctl restart dnsmasq
```

На каждой ВМ и хост-машине укажите адрес DNS-сервера в `/etc/resolv.conf`:
```
nameserver 192.168.56.20
```

### Вариант B: BIND9

```bash
sudo apt-get install -y bind9 bind9utils
```

`/etc/bind/named.conf.local`:
```
zone "42.sirius" {
    type master;
    file "/etc/bind/db.42.sirius";
};
```

`/etc/bind/db.42.sirius`:
```
$TTL    604800
@       IN  SOA     ns1.42.sirius. admin.42.sirius. (
                    2024010101  ; Serial
                    604800      ; Refresh
                    86400       ; Retry
                    2419200     ; Expire
                    604800 )    ; Negative Cache TTL
;
@       IN  NS      ns1.42.sirius.
ns1     IN  A       192.168.56.20
*       IN  A       192.168.56.200
```

```bash
sudo systemctl restart bind9
```

### Вариант C: FreeIPA (корпоративный стандарт)

Разверните 4-ю ВМ на **Rocky Linux 9** и установите FreeIPA:

```bash
sudo dnf install -y ipa-server ipa-server-dns
sudo ipa-server-install \
  --domain=42.sirius \
  --realm=42.SIRIUS \
  --ds-password=<пароль> \
  --admin-password=<пароль> \
  --mkhomedir \
  --setup-dns \
  --forwarder=8.8.8.8
```

FreeIPA предоставляет:
- DNS с wildcard-записями
- LDAP-каталог пользователей
- Kerberos-аутентификацию
- Web-интерфейс управления

Добавление wildcard DNS-записи через веб-интерфейс:
```
Зона: 42.sirius
Запись: * → A → 192.168.56.200
```

### Проверка DNS

```bash
# С любой машины в сети
nslookup gitlab.42.sirius 192.168.56.20
dig grafana.42.sirius @192.168.56.20

# Тест в браузере
curl -H "Host: grafana.42.sirius" http://192.168.56.200/
```

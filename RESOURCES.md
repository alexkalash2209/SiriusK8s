# Полезные ресурсы для курса Kubernetes

## Официальная документация

- [Kubernetes Documentation](https://kubernetes.io/docs/) — основной справочник
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/) — описание всех объектов
- [Docker Documentation](https://docs.docker.com/) — справочник по Docker
- [containerd](https://containerd.io/) — container runtime, используемый Kubernetes

## Подготовка к сертификации

### CKA (Certified Kubernetes Administrator)
- [Программа экзамена](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)
- [killer.sh](https://killer.sh/) — симулятор экзамена (2 бесплатные попытки при покупке)
- [CKA Exercises](https://github.com/chadmcrowell/CKA-Exercises) — практические задачи

### CKAD (Certified Kubernetes Application Developer)
- [Программа экзамена](https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/)
- [CKAD Exercises](https://github.com/dgkanatsios/CKAD-exercises) — 150+ задач с решениями

### CKS (Certified Kubernetes Security Specialist)
- [Программа экзамена](https://training.linuxfoundation.org/certification/certified-kubernetes-security-specialist/)
- [CKS Study Guide](https://github.com/walidshaari/Certified-Kubernetes-Security-Specialist) — подготовка к CKS

## Безопасность контейнеров

- [Trivy](https://aquasecurity.github.io/trivy/) — сканер образов и IaC
- [Falco](https://falco.org/) — runtime-детектирование угроз
- [OPA/Gatekeeper](https://open-policy-agent.github.io/gatekeeper/) — policy enforcement
- [Kyverno](https://kyverno.io/) — policy engine для Kubernetes
- [Docker Bench Security](https://github.com/docker/docker-bench-security) — CIS Benchmark аудит
- [Hadolint](https://github.com/hadolint/hadolint) — линтер Dockerfile
- [cosign](https://github.com/sigstore/cosign) — подпись контейнерных образов

## Сетевые плагины (CNI)

- [Calico](https://www.tigera.io/project-calico/) — сети + NetworkPolicy + eBPF
- [Cilium](https://cilium.io/) — eBPF-based networking + observability
- [Flannel](https://github.com/flannel-io/flannel) — простой overlay network
- [Weave Net](https://www.weave.works/oss/net/) — mesh networking

## Service Mesh

- [Istio](https://istio.io/) — service mesh с mTLS, traffic management
- [Linkerd](https://linkerd.io/) — лёгкий service mesh
- [Envoy Proxy](https://www.envoyproxy.io/) — L7 proxy (основа Istio)

## Мониторинг и логирование

- [Prometheus](https://prometheus.io/) — мониторинг и алертинг
- [Grafana](https://grafana.com/) — визуализация метрик
- [Loki](https://grafana.com/oss/loki/) — агрегация логов (стек Grafana)
- [Jaeger](https://www.jaegertracing.io/) — distributed tracing
- [OpenTelemetry](https://opentelemetry.io/) — стандарт observability

## CI/CD и GitOps

- [ArgoCD](https://argo-cd.readthedocs.io/) — GitOps для Kubernetes
- [Flux](https://fluxcd.io/) — альтернативный GitOps-инструмент
- [GitLab CI/CD](https://docs.gitlab.com/ee/ci/) — CI/CD пайплайны
- [GitHub Actions](https://github.com/features/actions) — CI/CD в GitHub
- [Tekton](https://tekton.dev/) — cloud-native CI/CD

## Инструменты разработчика

- [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) — шпаргалка по kubectl
- [Lens](https://k8slens.dev/) — IDE для Kubernetes
- [k9s](https://k9scli.io/) — TUI для управления кластером
- [Helm](https://helm.sh/) — пакетный менеджер для Kubernetes
- [Kustomize](https://kustomize.io/) — customize YAML без шаблонов
- [kind](https://kind.sigs.k8s.io/) — Kubernetes in Docker (для тестирования)
- [k3s](https://k3s.io/) — лёгкий дистрибутив Kubernetes

## Книги

- "Kubernetes in Action" — Marko Lukša (Manning)
- "Kubernetes Patterns" — Bilgin Ibryam, Roland Huß (O'Reilly)
- "Container Security" — Liz Rice (O'Reilly)
- "Production Kubernetes" — Josh Rosso et al. (O'Reilly)
- "Kubernetes Up & Running" — Brendan Burns et al. (O'Reilly)

## YouTube-каналы

- [TechWorld with Nana](https://www.youtube.com/c/TechWorldwithNana) — K8s туториалы
- [That DevOps Guy](https://www.youtube.com/c/MarcelDempers) — практический DevOps
- [Pavlenkoat](https://www.youtube.com/@pavlenkoat) — DevOps на русском
- [Docker Official](https://www.youtube.com/c/DockerIo) — официальный канал Docker

## Практические площадки

- [Play with Kubernetes](https://labs.play-with-k8s.com/) — бесплатный кластер в браузере
- [Katacoda](https://www.katacoda.com/) — интерактивные сценарии
- [Killercoda](https://killercoda.com/) — K8s sandbox-среды
- [SadServers](https://sadservers.com/) - задачки по устранению проблем в серверах

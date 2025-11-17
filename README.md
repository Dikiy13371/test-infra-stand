# Test Infra Stand

Этот репозиторий содержит полностью рабочий тестовый DevOps‑стенд, включающий:

- **k3d Kubernetes кластер (1 control-plane + 2 workers)**
- **Gitea** (Git, Docker Registry, Gitea Actions CI/CD)
- **Argo CD** (GitOps деплой всех манифестов)
- **Demo‑app** (Nginx приложение + Deployment + Service + Ingress)
- **kube‑prometheus‑stack** (Prometheus + Alertmanager + Grafana)
- Полностью автоматический pipeline: **push → build image → push to Gitea registry → ArgoCD auto-sync → deploy**

---

# 🏗 Архитектура стенда

```
                          ┌───────────────────────────────────────────┐
                          │                 Gitea                     │
                          │-------------------------------------------│
                          │ Git repos                                 │
                          │ Docker Registry (<gitea-ip>:3000/v2/)     │
                          │ Gitea Actions (CI)                        │
                          └───────────────────────────────────────────┘
                                            │ Push + Image Push
                                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                             Argo CD                                 │
│──────────────────────────────────────────────────────────────────── │
│ GitOps синхронизация из репозитория infra-manifests                 │
│ Автодеплой demo-app + мониторинга                                   │
└─────────────────────────────────────────────────────────────────────┘
                                            │ Apply manifests
                                            ▼
            ┌───────────────────────────────────────────────────┐
            │                  k3d cluster                      │
            │────────────────────────────────────────────────── │
            │  - k3d-infra-server-0 (control-plane)             │
            │  - k3d-infra-agent-0                              │
            │  - k3d-infra-agent-1                              │
            │                                                   │
            │  Ingress NGINX → demo-app                         │
            │  Prometheus + Grafana (kube-prometheus-stack)     │
            └───────────────────────────────────────────────────┘
```

---

# 📁 Структура репозитория

```
test-infra-stand/
 ├── infra-manifests/              # GitOps для Argo CD
 │   ├── application-demo-app.yaml
 │   ├── application-monitoring.yaml
 │   └── apps/
 │       └── demo-app/
 │           ├── deployment.yaml
 │           ├── service.yaml
 │           └── ingress.yaml
 │
 ├── gitea-runner/                 # Gitea Actions runner
 │   └── docker-compose.yml
 │
 ├── demo-app-src/                 # Исходники demo-приложения
 │   ├── Dockerfile
 │   ├── index.html
 │   └── .gitea/workflows/build.yml
 │
 └── README.md                     # Этот файл
```

---

# 🚀 1. Разворачивание Kubernetes (k3d)

```bash
k3d cluster create infra   --servers 1   --agents 2   --api-port 6445   -p "80:80@loadbalancer"   --k3s-arg "--disable=traefik@server:0"
```

Проверка:

```bash
kubectl cluster-info
kubectl get nodes -o wide
```

---

# 🧰 2. Установка Gitea в Docker

`docker-compose.yml`:

```yaml
version: "3"

services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    ports:
      - "3000:3000"
      - "2222:22"
    volumes:
      - ./data:/data
    restart: always
```

Запуск:

```bash
docker compose up -d
```

---

# 📦 3. Включение Gitea Docker Registry

В `app.ini`:

```
[packages.container.registry]
ENABLED = true
```

Адрес реестра:

```
http://<gitea-ip>:3000
```

Логин:

```bash
docker login http://<gitea-ip>:3000
```

---

# 🔧 4. CI/CD — Gitea Actions

`.gitea/workflows/build.yml`:

```yaml
name: Build Docker image

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: docker
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build and Push Image
        run: |
          docker build -t <gitea-ip>:3000/gitea/demo-app-src/demo:latest .
          docker push <gitea-ip>:3000/gitea/demo-app-src/demo:latest
```

---

# 🚀 5. Argo CD (GitOps)

Добавляем приложение:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
spec:
  project: default
  source:
    repoURL: https://github.com/<your>/test-infra-stand.git
    path: infra-manifests/apps/demo-app
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: apps
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

ArgoCI сам подтянет все манифесты.

---

# 📊 6. Мониторинг — kube-prometheus-stack

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 65.x.x
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Grafana доступна по:

```
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

---

# 🎉 Готово!

Получился полноценный DevOps стенд:

- Git → Build → Push → Deploy → Monitor  
- GitOps через ArgoCD  
- CI/CD через Gitea Actions  
- Docker Registry встроенный  
- Полная Observability  
- k3d кластер с несколькими нодами  

Используй как основы для CV, pet‑project или дальнейшей автоматизации.

---

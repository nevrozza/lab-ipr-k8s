# Мессенджер в Kubernetes

Репозиторий содержит конфигурационные манифесты для развертывания микросервисного мессенджера в Kubernetes-кластере. Проект спроектирован с разделением сред разработки (`dev`) и эксплуатации (`prod`) с использованием Kustomize, а процесс деплоя автоматизирован по принципам GitOps с помощью Argo CD.

## Архитектурная схема взаимодействия
<p align="center">
<img width="547" height="500" alt="image" src="https://github.com/user-attachments/assets/aee4ba9c-953f-4163-bb88-3d88588b9d1c" />
</p>

---

## Быстрый старт (Локальный запуск)

### Требования
* Kubernetes кластер (Minikube / Docker Desktop / Kind).
* Наличие метки `workload=app` на узле для запуска сервиса сообщений:
  ```bash
  NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
  kubectl label node $NODE_NAME workload=app --overwrite
  ```

### 1. Запуск среды разработки (Dev) через Kustomize
Локальная среда оптимизирована для работы без внешних зависимостей (S3 CSI эмулируется через быстрый `emptyDir` для обхода ограничений прав доступа хоста):
```bash
# Развертывание всей инфраструктуры
kubectl apply -k k8s/overlays/dev

# Проверка статуса подов (дождитесь Running для всех компонентов)
kubectl get pods -n messager-app -w
```

### 2. Доступ к приложению
Пробросьте порт для веб-интерфейса фронтенда:
```bash
kubectl port-forward svc/frontend 8080:80 -n messager-app
```
Откройте в браузере: `http://localhost:8080`.

---

## Развертывание через Argo CD (GitOps)

Проект поддерживает автоматическую синхронизацию состояния кластера с репозиторием. Манифесты разделены на две среды.

### 1. Установка Argo CD в кластер:
```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -

# Установка манифестов в режиме server-side (для обхода ограничений на размер)
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Запуск отслеживания сред:
* **Для локальной среды разработки (Dev):**
  ```bash
  kubectl apply -f argocd/application-dev.yaml
  ```
* **Для production среды (Prod):**
  ```bash
  kubectl apply -f argocd/application-prod.yaml
  ```

Далее:
```
# Узнать пароль от логина 'admin'
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Пробросить порт
kubectl port-forward svc/argocd-server -n argocd 8085:443
```
Вуаля – Argo CD доступен локально в браузере: `http://localhost:8085`.

---

## Навигация по детальной документации (docs/)
* [**docs/01-architecture-and-database.md**](./docs/01-architecture-and-database.md) — Сетевая связность, детальные схемы баз данных и решение проблемы коллизии миграций.
* [**docs/02-node-affinity-and-storage.md**](./docs/02-node-affinity-and-storage.md) — Настройка распределения нагрузок (Affinity), работа с хранилищем файлов (S3 CSI) и решение проблем с правами локальной ФС.
* [**docs/03-kustomize-and-gitops.md**](./docs/03-kustomize-and-gitops.md) — Разделение сред через Kustomize, автоматизация через Argo CD и скриншоты работающей системы.

---

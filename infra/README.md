# telegram-support-infra

Инфраструктурный репозиторий: **только PostgreSQL**.  
Код приложения и его манифесты — в отдельном репозитории `telegram-support-bot`.

---

## Контракт для приложения

| Параметр | dev | prod |
|---|---|---|
| Хост | `postgres-0.postgres.telegram-demo.svc.cluster.local` | `postgres-0.postgres.telegram-prod.svc.cluster.local` |
| Порт | `5432` | `5432` |
| База данных | `support_bot` | `support_bot` |
| Пользователь | `support_user` | `support_user` |
| Пароль | из Secret `postgres-secret` (ключ `postgres-password`) | из Secret `postgres-secret` (ключ `postgres-password`) |

Полная строка подключения (DATABASE_URL):
```
postgresql://support_user:<password>@postgres-0.postgres.<namespace>.svc.cluster.local:5432/support_bot
```

> **Важно:** пароль не хранится в Git в открытом виде.  
> В dev используется значение из `values-dev.yaml` (только для локальной разработки).  
> В prod пароль подаётся через CI-переменную или Sealed Secrets.

---

## Порядок деплоя

### Вариант A — Helm

```bash
# 1. Убедитесь, что есть StorageClass
kubectl get storageclass

# 2. Установка / обновление (dev)
helm upgrade --install telegram-support-db ./k8s/helm/postgres-infra \
  --namespace telegram-demo --create-namespace \
  -f ./k8s/helm/postgres-infra/values-dev.yaml

# 3. Проверка
kubectl get pods,pvc -n telegram-demo -l app=postgres
kubectl exec -n telegram-demo postgres-0 -- psql -U support_user -d support_bot -c '\l'
```

### Вариант B — Kustomize

```bash
kubectl apply -k k8s/kustomization/overlays/dev
kubectl get pods,pvc -n telegram-demo -l app=postgres
```

---

## Удаление

```bash
helm uninstall telegram-support-db -n telegram-demo
# PVC не удаляются автоматически — данные сохраняются
kubectl get pvc -n telegram-demo
# При необходимости удалить данные:
kubectl delete pvc -n telegram-demo -l app=postgres
```

> **Нюанс:** PVC переживают удаление StatefulSet и Helm-релиза.  
> Это сделано намеренно для защиты данных. Удаляйте PVC осознанно.

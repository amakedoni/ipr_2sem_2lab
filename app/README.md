# telegram-support-app — манифесты приложения

Приложение: Telegram Support Bot.  
База данных — в отдельном репозитории `telegram-support-infra`.

---

## Структура

```
k8s/
  kustomization/
    base/             # Deployment, Service, ConfigMap (без БД)
    overlays/
      dev/            # namespace: telegram-demo, 1 реплика, DEBUG
      prod/           # namespace: telegram-prod, 2 реплики, INFO
  helm/
    telegram-support-app/
      values.yaml           # базовые значения
      values-dev.yaml       # dev-переопределения
      values-prod.yaml      # prod-переопределения
      templates/            # Secret, ConfigMap, Deployment, Service
```

> **Манифестов PostgreSQL здесь нет.** DATABASE_URL задаётся через overlay/values  
> и совпадает с контрактом из `telegram-support-infra/README.md`.

---

## Порядок деплоя

### 0. Сначала — инфраструктура

```bash
# Убедитесь, что БД уже запущена:
kubectl get pods -n telegram-demo -l app=postgres
```

### 1a. Kustomize (dev)

```bash
# Заполните bot-token в overlays/dev/secret.yaml, затем:
kubectl apply -k k8s/kustomization/overlays/dev

# Проверка
kubectl get pods -n telegram-demo -l app=telegram-support-bot
kubectl logs -n telegram-demo -l app=telegram-support-bot
```

### 1b. Helm (dev)

```bash
helm template telegram-app ./k8s/helm/telegram-support-app \
  --namespace telegram-demo \
  -f ./k8s/helm/telegram-support-app/values-dev.yaml \
  --set telegram.botToken="YOUR_TOKEN"

helm upgrade --install telegram-app ./k8s/helm/telegram-support-app \
  --namespace telegram-demo --create-namespace \
  -f ./k8s/helm/telegram-support-app/values-dev.yaml \
  --set telegram.botToken="YOUR_TOKEN"
```

### Откат / удаление (Helm)

```bash
helm rollback telegram-app 1 -n telegram-demo
helm uninstall telegram-app -n telegram-demo
```

---

## Секреты

- `bot-token` — никогда не коммитить в Git; передавать через `--set` или CI.
- `database-url` — формируется из контракта инфраструктуры; пароль берётся из CI.

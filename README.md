# Лабораторная работа №6 — Kustomize и Helm

## Структура

```
lab6/
├── infra/   # PostgreSQL: Helm + Kustomize, dev/prod
└── app/     # Бот: Kustomize + Helm, dev/prod, без манифестов БД
```

---

## Шаг 1 — Проверить кластер и StorageClass

```bash
kubectl cluster-info
kubectl get storageclass
```

---

## Шаг 2 — Инфраструктура (PostgreSQL)

### Helm

```bash
cd infra

helm template telegram-support-db ./k8s/helm/postgres-infra \
  --namespace telegram-demo \
  -f ./k8s/helm/postgres-infra/values-dev.yaml

helm upgrade --install telegram-support-db ./k8s/helm/postgres-infra \
  --namespace telegram-demo --create-namespace \
  -f ./k8s/helm/postgres-infra/values-dev.yaml
```

### Kustomize

```bash
kubectl kustomize k8s/kustomization/overlays/dev
kubectl apply -k k8s/kustomization/overlays/dev
```

### Проверка

```bash
kubectl get pods,pvc -n telegram-demo -l app=postgres
kubectl exec -n telegram-demo postgres-0 -- pg_isready -U support_user -d support_bot
kubectl exec -it -n telegram-demo postgres-0 -- psql -U support_user -d support_bot -c '\l'
kubectl get svc -n telegram-demo postgres -o yaml | grep clusterIP
kubectl run dns-test --rm -it --image=busybox --restart=Never -n telegram-demo -- \
  nslookup postgres-0.postgres.telegram-demo.svc.cluster.local
```

---

## Шаг 3 — Приложение

### Kustomize

```bash
cd app

kubectl kustomize k8s/kustomization/overlays/dev
kubectl apply -k k8s/kustomization/overlays/dev
```

### Helm (просмотр шаблонов)

```bash
helm template telegram-app ./k8s/helm/telegram-support-app \
  --namespace telegram-demo \
  -f ./k8s/helm/telegram-support-app/values-dev.yaml \
  --set telegram.botToken="1234567890:AAFakeTokenForTestingPurposesOnly123"
```

### Проверка

```bash
kubectl get pods -n telegram-demo -l app=telegram-support-bot
kubectl logs -n telegram-demo -l app=telegram-support-bot
kubectl get secret app-secret -n telegram-demo \
  -o jsonpath='{.data.database-url}' | base64 -d
kubectl get deployment -n telegram-demo -l app=telegram-support-bot
```

---

## Шаг 4 — Проверить разделение

```bash
kubectl get all,pvc,secret -n telegram-demo
kubectl kustomize app/k8s/kustomization/overlays/dev | grep -c "Deployment"
```

---

## Шаг 5 — Helm: история и rollback (инфраструктура)

```bash
helm history telegram-support-db -n telegram-demo
helm rollback telegram-support-db 1 -n telegram-demo
kubectl get pvc -n telegram-demo
```

---

## Шаг 6 — Очистка

```bash
kubectl delete -k app/k8s/kustomization/overlays/dev
helm uninstall telegram-support-db -n telegram-demo
kubectl delete pvc -n telegram-demo --all
kubectl delete namespace telegram-demo
```

---


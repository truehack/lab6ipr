# Лабораторная работа №6: Kustomize и Helm, разделение приложения и инфраструктуры

<img width="100" height="100" alt="Kubernetes_logo_without_workmark svg" src="https://github.com/user-attachments/assets/db1e5d5a-4fc4-4228-ad12-1b9f6b6d1e26" />



## Клонирование репозитория
- git clone https://github.com/truehack/lab6ipr.git
- cd lab6ipr

## Проверка структуры проекта
# Проверка наличия инфраструктуры
- ls -la telegram-support-infra/k8s/helm/postgres-infra/
- ls -la telegram-support-infra/k8s/kustomization/

## Развёртывание инфраструктуры (PostgreSQL)
- cd lab6ipr

# Установка PostgreSQL через Helm
helm upgrade --install postgres-infra ./k8s/helm/postgres-infra \
  --namespace telegram-demo --create-namespace \
  --set storage.storageClassName=hostpath \
  --set storage.size=5Gi

## Проверка работы PostgreSQL
- kubectl get pods -n telegram-demo
- kubectl get pvc -n telegram-demo
- kubectl exec -n telegram-demo postgres-infra-postgres-0 -- psql -U bot_user -d support_bot -c "SELECT 1 as test;"
- kubectl get secret -n telegram-demo postgres-infra-postgres-secret -o jsonpath="{.data.DATABASE_URL}" | base64 --decode

## Создание секрет для приложения
kubectl create secret generic app-secret -n telegram-demo \
  --from-literal=DATABASE_URL="postgresql://bot_user:dev-password@postgres:5432/support_bot" \
  --dry-run=client -o yaml | kubectl apply -f -

## Создание тестового приложения
kubectl apply -n telegram-demo -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: app
        image: alpine
        command: ["sleep", "infinity"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DATABASE_URL
EOF

## Очистка
- helm uninstall postgres-infra -n telegram-demo
- kubectl delete namespace telegram-demo

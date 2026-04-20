# Лабораторная работа №6: Kustomize и Helm, разделение приложения и инфраструктуры

---
<div align="center">
  <h3>☸️ Kubernetes + 📦 Helm + ⚙️ Kustomize</h3>
  <br/>
</div>

---




# Клонирование репозитория
- git clone https://github.com/truehack/lab6ipr.git
- cd lab6ipr

## Работа с репозиторием
```
# Создайте namespace
kubectl create namespace telegram-demo

# Создайте Secret
kubectl create secret generic postgres-secret -n telegram-demo \
  --from-literal=POSTGRES_PASSWORD=dev-password

# Создайте ConfigMap
kubectl create configmap postgres-config -n telegram-demo \
  --from-literal=POSTGRES_DB=support_bot \
  --from-literal=POSTGRES_USER=bot_user

# Создайте StatefulSet и Service
kubectl apply -n telegram-demo -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15.3
        envFrom:
        - configMapRef:
            name: postgres-config
        - secretRef:
            name: postgres-secret
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: hostpath
      resources:
        requests:
          storage: 5Gi
EOF
```

### Проверка
```
kubectl get pods -n telegram-demo
kubectl get pvc -n telegram-demo
kubectl exec -n telegram-demo postgres-0 -- psql -U bot_user -d support_bot -c "SELECT 1 as test;"
```


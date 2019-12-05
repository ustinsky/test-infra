#Домашняя работа 4 (kuber-database)
## 1. kind
### 1.1 Установка 
```
https://kind.sigs.k8s.io/docs/user/quick-start#installation
```

### 1.2 Создаем кластер
```
kind create cluster
export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
```
### 1.3 Создаем файл minio-statefulset.yaml
```
https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Kuberenetes-volumes/minio-statefulset.yaml
```

### 1.4 Создаем файл minio-headless-service.yaml
```
https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Kuberenetes-volumes/minio-headless-service.yaml
```

## 2. Проверка работы
### 2.1 Используя команды
```
kubectl get statefulsets
kubectl get pods
kubectl get pvc
kubectl get pv
kubectl describe <resource> <resource_name>
```
### 2.2 Используя minio/mc
```
https://github.com/minio/mc
```
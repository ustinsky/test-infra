   
# Домашняя работа 2 (kuber-security)
## 1. task01
###  1.1 Создать Service Account bob, дать ему роль admin в рамках всего кластера
1) nano 01-sa-bob.yaml
```
        apiVersion: v1
        kind: ServiceAccount
        metadata:
            name: bob
```
        
2) nano 02-rb-bob.yaml
```
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
            name: crb-bob
        subjects:
        -   kind: ServiceAccount
            name: bob
            apiGroup: rbac.authorization.k8s.io
        roleRef:
            kind: ClusterRole
            name: cluster-admin
            apiGroup: rbac.authorization.k8s.io
```
    
### 1.2 Создать Service Account dave без доступа к кластеру
3) nano 03-sa-dave.yaml
```
        apiVersion: v1
        kind: ServiceAccount
        metadata:
            name: dave
```

## 2. task02

### 2.1 Создать Namespace prometheus   
```
    nano 01-ns-prometheus.yaml
    kind: Namespace 
    apiVersion: v1
    metadata:
        name: prometheus    
```

### 2.2 Создать Service Account carol в этом Namespace
```
    nano 02-sa-carol.yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: carol
      namespace: prometheus
```

### 2.3 Дать всем Service Account в prometheus возможность делать get, list, watch в отношении Pods всего кластера
        
1) nano 03-role-pod-reader.yaml
```
        apiVersion: rbac.authorization.k8s.io/v1
        kind: Role
        metadata:
        name: pod-reader
        rules:
        - apiGroups: [""] 
          resources: ["pods"]
          verbs: ["get", "watch", "list"]
```

2) nano 04-rb-prometheus.yaml
```
        apiVersion: rbac.authorization.k8s.io/v1
        kind: RoleBinding
        metadata:
            name: rb-prometheus
        subjects:
        -   kind: Group
            name: system:serviceaccounts:prometheus
            apiGroup: rbac.authorization.k8s.io
        roleRef:
            kind: Role
            name: pod-reader
            apiGroup: rbac.authorization.k8s.io
```

## 3. task03
### 3.1 Создать namespace dev
```
        nano 01-ns-dev.yaml
        kind: Namespace 
        apiVersion: v1
        metadata:
            name: dev
```

### 3.2 Создать Service Account jane в namespace dev
```
        nano 02-sa-jane.yaml
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: jane
          namespace: dev
```

### 3.3 Дать jane роль admin в рамках namespace dev
```
        nano 03-rb-jane.yaml
        apiVersion: rbac.authorization.k8s.io/v1
        kind: RoleBinding
        metadata:
            name: rb-jane
            namespace: dev
        subjects:
        -   kind: ServiceAccount
            name: jane
            namespace: dev
        roleRef:
            kind: ClusterRole
            name: admin
            apiGroup: rbac.authorization.k8s.io
```

### 3.4 Создать Service Account ken в namespace dev
```
        nano 04-sa-ken.yaml
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: ken
          namespace: dev
```

### 3.5 Дать ken роль view в рамках namespace dev
```
        nano 05-rb-ken.yaml
        apiVersion: rbac.authorization.k8s.io/v1
        kind: RoleBinding
        metadata:
            name: rb-ken
            namespace: dev
        subjects:
        -   kind: ServiceAccount
            name: ken
            namespace: dev
        roleRef:
            kind: ClusterRole
            name: view
            apiGroup: rbac.authorization.k8s.io
```
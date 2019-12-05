# Домашняя работа 3 (kuber-network)
    
## 1. Работа с тестовым веб-приложением
### 1.1. Добавление проверок Pod
#### 1.1.1 Добавить в описание пода
```
                readinessProbe:
                  httpGet:
                    path: /index.html
                    port: 80
```

#### 1.1.2 Запустить под командой 
```
kubectl apply -f web-pod.yaml
```

#### 1.1.3 Проверяем запуск
```
kubectl get pod/web
```
            
#### 1.1.4 Смотрим события запуска
```
kubectl describe pod/web
```
            
#### 1.1.5 Добавим liveness проверку
```
                livenessProbe:
                    tcpSocket: { port: 8000 }
```

#### 1.1.6 Вопрос
1. Почему следующая конфигурация валидна, но не имеет смысла?
```
                    livenessProbe:
                        exec:
                            command:
                                - 'sh'
                                - '-c'
                                - 'ps aux | grep my_web_server_process'
```

Ответ:
```
В команде надо еще убрать вывод самой команды grep. 'ps aux | grep my_web_server_process | grep -v grep'
И даже в этом случае наличие процесса в списке процессов не гарантирует его корректную работу. Процесс может подвиснуть, 
переити в deadlock. Тогда в списке процессов он будет, проверку пройдет, но работать не будет. 
```
                
2. Бывают ли ситуации, когда она все-таки имеет смысл?

Ответ:
```
Возможно это используется для каких то задач, которые конечны (процесс завершиться после выполнения какой либо задачи). 
Далее для того чтобы отреагировала диагностика пода и убрала контейнер с завершенным процессом.
```

### 1.2. Создание объекта Deployment
#### 1.2.1 Создаем файл web-deploy.yaml
``` 
                apiVersion: apps/v1
                kind: Deployment
                metadata:
                  name: web                 # Название нашего объекта Deployment
                spec:
                  replicas: 1               # Начнем с одного пода
                  selector:                 # Укажем, какие поды относятся к нашему Deployment:
                    matchLabels:            # - это поды с меткой
                    app: web                # app и ее значением web
                  template:                 # Теперь зададим шаблон конфигурации пода
```

#### 1.2.2 Удаляем старый под
```
kubectl delete pod/web --grace-period=0 --force
```
            
#### 1.2.3 Применяем Deployment
```
kubectl apply -f web-deploy.yaml
```

#### 1.2.4 Посмотрим что получилось
```
kubectl describe deployment web
```

#### 1.2.5 Добавляем стратегию
```
                strategy:
                    type: RollingUpdate
                    rollingUpdate:
                        maxUnavailable: 0
                        maxSurge: 100%
```

### 1.3. Добавление сервисов в кластер (ClusterIP)
#### 1.3.1 Создаем сервис web-svc-cip.yaml
```
                apiVersion: v1
                kind: Service
                metadata:
                  name: web-svc-cip
                spec:
                selector:
                  app: web
                type: ClusterIP
                ports:
                - protocol: TCP
                  port: 80
                  targetPort: 8000
```

#### 1.3.2 Применяем
```
kubectl apply -f web-svc-cip.yaml
```

#### 1.3.3 Проверяем
```
kubectl get services
```

#### 1.3.4 Проверяем цепочку
```
iptables --list -nv -t nat
```

Я так понимаю работает так
1. цепочка OUTPUT кидает на цепочку KUBE-SERVICES
2. KUBE-SERVICES видя IP-destination кидает на цепочку KUBE-SVC-...
3. KUBE-SVC-... распределяет нагрузку и кидает на одну из KUBE-SEP-...
4. KUBE-SEP-... делает DNAT на конкретную машину
 
## 1.4. Включение режима балансировки IPVS
### 1.4.1 Включим IPVS
```
kubectl --namespace kube-system edit configmap/kube-proxy
```

ищем KubeProxyConfig
меняем ```mode: ""``` => ```mode: "ipvs"```


#### 1.4.2 Перезагружаем kube-proxy
```
kubectl --namespace kube-system delete pod --selector='k8s-app=kube-proxy'
```

#### 1.4.3 Создадим файл /tmp/iptables.cleanup
```
*nat
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
COMMIT
*filter
COMMIT
*mangle
COMMIT
```
            
#### 1.4.4 Применим конфигурацию
```
iptables-restore /tmp/iptables.cleanup
```

#### 1.4.5 Через 30 секунд kube-proxy восстановит правила для подов
```
iptables --list -nv -t nat
```

#### 1.4.6 IPVS
```            
toolbox
dnf install -y ipvsadm && dnf clean all
dnf install -y ipset && dnf clean all
ipvsadm --list -n
```

## 2. Доступ к приложению извне кластера 

### 2.1. Установка MetaILB в Layer2 режиме
####  2.1.1 Установка
```
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.0/manifests/metallb.yaml
kubectl --namespace metallb-system get all
```
#### 2.1.2 Настройка балансировщика metallb-config.yaml
```
                apiVersion: v1
                kind: ConfigMap
                metadata:
                    namespace: metallb-system
                    name: config
                data:
                    config: |
                        address-pools:
                            - name: default
                              protocol: layer2
                              addresses:
                                - "172.17.255.1-172.17.255.255" 
```

### 2.2. Добавление сервиса LoadBalancer
#### 2.2.1 Создаем файл
```
                apiVersion: v1
                kind: Service
                metadata:
                    name: web-svc-lb
                spec:
                    selector:
                        app: web
                    type: LoadBalancer
                    ports:
                        - protocol: TCP
                        port: 80
                        targetPort: 8000
```

#### 2.2.2 Применяем и Проверяем
```
kubectl apply -f web-svc-lb.yaml
kubectl --namespace metallb-system logs pod/controller-7757586ff4-d8ds2
kubectl describe svc web-svc-lb
```
#### 2.2.3 Создаем статический маршрут в основной ОС
```
#minikube ip
192.168.99.103
#ip route add 172.17.255.0/24 via 192.168.99.103
```

#### 2.2.4 DNS через MataILB
1. Создаем манифест coredns-svc-lb.yaml
```
                    ---
                    apiVersion: v1
                    kind: Service
                    metadata:
                    name: coredns-svc-lb-udp
                    annotations:
                        metallb.universe.tf/allow-shared-ip: coredns
                    namespace: kube-system
                    spec:
                    selector:
                        k8s-app: kube-dns
                    type: LoadBalancer
                    loadBalancerIP: 172.17.255.2
                    ports:
                    - protocol: UDP
                        port: 53
                        targetPort: 53
                    ---
                    apiVersion: v1
                    kind: Service
                    metadata:
                    name: coredns-svc-lb-tcp
                    annotations:
                        metallb.universe.tf/allow-shared-ip: coredns
                    namespace: kube-system
                    spec:
                    selector:
                        k8s-app: kube-dns
                    type: LoadBalancer
                    loadBalancerIP: 172.17.255.2
                    ports:
                    - protocol: TCP
                        port: 53
                        targetPort: 53
```

2. Проверяем работу
```
nslookup web-svc-lb.default.svc.cluster.local 172.17.255.2
```

### 2.3. Установка ingress-контроллера и прокси ingress-nginx
#### 2.3.1 Установка
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
```
            
#### 2.3.2 Создаем файл nginx-lb.yaml
```
                kind: Service
                apiVersion: v1
                metadata:
                    name: ingress-nginx
                    namespace: ingress-nginx
                labels:
                    app.kubernetes.io/name: ingress-nginx
                    app.kubernetes.io/part-of: ingress-nginx
                spec:
                    externalTrafficPolicy: Local
                    type: LoadBalancer
                    selector:
                        app.kubernetes.io/name: ingress-nginx
                        app.kubernetes.io/part-of: ingress-nginx
                    ports:
                        - { name: http, port: 80, targetPort: http }
                        - { name: https, port: 443, targetPort: https }
```
            
#### 2.3.3 Применим
```
kubectl apply -f nginx-lb.yaml
kubectl get services -n ingress-nginx
```

#### 2.3.4 Создание Headless сервиса web-svc-headless.yaml
```
                apiVersion: v1
                kind: Service
                metadata:
                    name: web-svc
                spec:
                    selector:
                        app: web
                    type: ClusterIP
                    clusterIP: None
                    ports:
                        - protocol: TCP
                          port: 80
                          targetPort: 8000
```


### 2.4. Создание правил Ingress
#### 2.4.1 Создаем файл web-ingress.yaml
```
                apiVersion: networking.k8s.io/v1beta1
                kind: Ingress
                metadata:
                    name: web
                    annotations:
                        nginx.ingress.kubernetes.io/rewrite-target: /
                spec:
                    rules:
                        - http:
                            paths:
                                - path: /web
                                  backend:
                                    serviceName: web-svc
                                    servicePort: 8000
```

#### 2.4.2 Проверяем
```
kubectl apply -f web-ingress.yaml
kubectl describe ingress/web
```

#### 2.4.3 Canary проверка
```
curl -kL http://<IngressIP>/webapp
curl -kL --header 'testapp: true'  http://<IngressIP>/webapp
```

## 3 Для запуска:
```
kubectl apply -f web-deploy.yaml
kubectl apply -f web-svc-cip.yaml
kubectl --namespace kube-system edit configmap/kube-proxy (ищем KubeProxyConfig и меняем mode: "" => mode: "ipvs")
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.0/manifests/metallb.yaml
kubectl apply -f metallb-config.yaml
kubectl apply -f web-svc-lb.yaml
minikube ip
ip route add 172.17.255.0/24 via <IP>
kubectl apply -f coredns-svc-lb.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
kubectl apply -f nginx-lb.yaml
kubectl apply -f web-svc-headless.yaml
kubectl apply -f web-ingress.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```
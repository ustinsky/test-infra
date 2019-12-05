# ustinsky_platform
ustinsky Platform repository

Домашняя работа 1 (kuber-intro)

1.    Установка kubectl <br>
    1.1   Скачиаем
    ~~~~
        curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
    ~~~~
    1.2   Даем права
    ~~~~
        chmod +x ./kubectl
    ~~~~
    1.3 Переносим в /usr/local/bin
    ~~~~
        sudo mv ./kubectl /usr/local/bin/kubectl
    ~~~~  
    1.4 Настраиваем автодополнение
    ~~~~
        source <(kubectl completion bash) # setup autocomplete in bash into the current shell, bash-completion package should be installed first.
        echo "source <(kubectl completion bash)" >> ~/.bash_profile
    ~~~~
    
2.  Установка minikube
    2.1 Проверяем наличие поддержки аппаратной виртуализации
    ~~~~
        grep -E --color 'vmx|svm' /proc/cpuinfo
    ~~~~
    2.2 Устанавливаем VirtualBox

    2.3 Скачиваем minikube
    ~~~~
        curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube
    ~~~~
    2.4 Устанавливаем minikube
    ~~~~
        sudo install minikube /usr/local/bin
    ~~~~
3. Установка kind (https://kind.sigs.k8s.io/)
    1. Ставим
        GO111MODULE="on" go get sigs.k8s.io/kind@v0.5.1 && kind create cluster
    2. Опредеяем настройки kubectl
        export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
    3. Проверяем кластер
        kubectl cluster-info

4. Запуск minikube 
    minikube start

5. Просмотр текущей конфигурации
    5.1 kubectl 
        kubectl config view
    5.2 Подключение к кластеру
        kubectl cluster-info

6.  Kubernetes Dashboard
    6.1 Применяем
        kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
    6.2 Доступ (https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
        6.2.1 dashboard-service-account.yaml

            apiVersion: v1
            kind: ServiceAccount
            metadata:
                name: admin-user
                namespace: kube-system
        
        
        6.2.2 dashboard-role-binding.yaml

            apiVersion: rbac.authorization.k8s.io/v1
            kind: ClusterRoleBinding
            metadata:
                name: admin-user
            roleRef:
                apiGroup: rbac.authorization.k8s.io
                kind: ClusterRole
                name: cluster-admin
            subjects:
            -   kind: ServiceAccount
                name: admin-user
                namespace: kube-system

        6.2.3 Смотрим токен
            kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}') 

    6.3 Проксируем
        kubectl proxy

    6.4 Заходим на http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/ 
        используя токен

7.  k9s
    7.1 Скачиваем 
        wget https://github-production-release-asset-2e65be.s3.amazonaws.com/167596393/06956700-be26-11e9-8765-7e323b0dce92?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20190826%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20190826T144828Z&X-Amz-Expires=300&X-Amz-Signature=11380ce16989a1de7bc454f6c13d3ab9784884bddde989f26cbba0dfad3aecd4&X-Amz-SignedHeaders=host&actor_id=18228849&response-content-disposition=attachment%3B%20filename%3Dk9s_0.8.2_Linux_x86_64.tar.gz&response-content-type=application%2Foctet-stream

    7.2 Распаковываем и запускаем

8.  Заходим на VM minikube
    8.1 Заходим на VM по SSH
        minikube ssh
    8.2 Проверяем устойчивость к отказам 
        docker rm -f $(docker ps -a -q)

9.  Проверяем устойчивость через kubectl
    9.1 Получаем список pod-ов для namespace kube-system
        kubectl get pods -n kube-system
    9.2 Удаляем все поды для этого namespace
        kubectl delete pod --all -n kube-system

10. Проверка что кластер находится в рабочем состоянии
    kubectl get componentstatuses
    kubectl get cs 

11. Почему происходит восcтановление системных подов после удаления
    kubelet запущен как сервис systemd. Он занимается процессом запуска pod-ов.
    Если выполнить 
    sudo systemctl stop kubelet; docker rm -f $(docker ps -a -q)
    то кластер не восстанавливается автоматически

    для запуска кластера
    sudo systemctl start kubelet;
    после чего кластер запускает необходимые контейнеры

    core-dns реализован как Deployment с параметром replicas: 2

    так же можно сломать кластер командой 
    (kill -9 `pgrep -f docker`) &

    Не посмотрел внимательно !!!??? 
    При аварийном запуске имеется процесс coredns, при нормальной работе coredns - это несколько контейнеров

    ??? (coredns и api-server имеют разные причины восстановления)

    // сломал ( (kill -9 `pgrep -f docker`) & )
    systemd-+-VBoxService---6*[{VBoxService}]
        |-bash---sleep
        |-2*[coredns---9*[{coredns}]]
        |-dbus-daemon
        |-etcd---11*[{etcd}]
        |-getty
        |-kube-apiserver---10*[{kube-apiserver}]
        |-kube-proxy---7*[{kube-proxy}]
        |-kube-scheduler---9*[{kube-scheduler}]
        |-11*[pause]
        |-rpc.mountd
        |-rpcbind
        |-sshd---sshd---sshd---bash---pstree
        |-storage-provisi---6*[{storage-provisi}]
        |-systemd-journal
        |-systemd-logind
        |-systemd-network
        |-systemd-resolve
        `-systemd-udevd

        =======================
    systemd-+-VBoxService---6*[{VBoxService}]
        |-dbus-daemon
        |-dockerd-+-containerd-+-2*[containerd-shim-+-pause]
        |         |            |                    `-10*[{containerd-shim}]]
        |         |            |-7*[containerd-shim-+-pause]
        |         |            |                    `-9*[{containerd-shim}]]
        |         |            |-containerd-shim-+-bash---sleep
        |         |            |                 `-10*[{containerd-shim}]
        |         |            |-containerd-shim-+-etcd---11*[{etcd}]
        |         |            |                 `-10*[{containerd-shim}]
        |         |            |-containerd-shim-+-kube-controller---8*[{kube-controller}]
        |         |            |                 `-10*[{containerd-shim}]
        |         |            |-containerd-shim-+-kube-apiserver---10*[{kube-apiserver}]
        |         |            |                 `-9*[{containerd-shim}]
        |         |            |-containerd-shim-+-kube-scheduler---9*[{kube-scheduler}]
        |         |            |                 `-9*[{containerd-shim}]
        |         |            |-containerd-shim-+-kube-proxy---8*[{kube-proxy}]
        |         |            |                 `-10*[{containerd-shim}]
        |         |            |-containerd-shim-+-storage-provisi---7*[{storage-provisi}]
        |         |            |                 `-9*[{containerd-shim}]
        |         |            |-containerd-shim-+-coredns---10*[{coredns}]
        |         |            |                 `-9*[{containerd-shim}]
        |         |            |-containerd-shim-+-coredns---9*[{coredns}]
        |         |            |                 `-10*[{containerd-shim}]
        |         |            `-29*[{containerd}]
        |         `-25*[{dockerd}]
        |-getty
        |-kubelet---16*[{kubelet}]
        |-rpc.mountd
        |-rpcbind
        |-sshd---sshd---sshd---bash---pstree
        |-systemd-journal
        |-systemd-logind
        |-systemd-network
        |-systemd-resolve
        `-systemd-udevd


12. Создать Dockerfile в котором будет описан образ:
    1. Запускающий web-сервер на порту 8000
    2. Отдающий содержимое директории /app
    3. Работающий с UID 1001

13. Dockerfile: 
    1. разместить в kubernetes-intro/web 
    2. Собрать образ и разместить его в DockerHub

14. Написать манифест web-pod.yaml
    apiVersion: v1      # Версия API 
    kind: Pod           # Объект, который создаем
    metadata:
        name:           # Название Pod
        labels:         # Метки в формате key: value
            key: value
    spec:               # Описание Pod
        containers:     # Описание контейнеров внутри Pod
            - name:     # Название контейнера
              image:    # Образ из которого создается контейнер

15. Применить манифест и разметить в kubernetes-intro
    15.1 Применяем
        kubectl apply -f web-pod.yaml
    15.2 Проверяем работу 
        kubectl get pods

16. Получаем от kubernetes манифест уже запущенного pod-а 
    kubectl get pod web -o yaml

17. Получаем текущее состояние и события pod-а 
    kubectl describe pod web

18. Успешная старт pod-а дает следующие сообщения:
    1. scheduler определил где запускать pod
    2. kubelet скачал необходимый образ и запустил контейнер

19. Имитация неудачного старта 
    19.1 Добавить в web-pod.yaml несуществующий тэг и применить
        kubectl apply -f web-pod.yaml
    19.2 Проверить вывод команды 
        kubectl get pods
        kubectl describe pod web

20. Init контейнеры:
    image: busybox:1.31.0
    command: ['sh', '-c', 'wget -O- https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Introduction-to-Kubernetes/wget.sh | sh']

21. Volume:
    для контейнера:
        volumeMounts:
            -name: app
             mountPath: /app
    для pod:
        volumes:
            -name: app
             emptyDir: {}

22. Удалить запущенный под из кластера и применить обновленный манифест
    1. kubectl delete pod web
    2. kubectl get pods -w
    3. kubectl apply -f web-pod.yaml && kubectl get pods -w 

23. Проверка работы приложения
    1. kubectl port-forward --address 0.0.0.0 pod/web 8000:8000
    2. открыть в браузере http://localhost:8000/index.html

24. Kube-forwarder 

25. Добавить файлы
    1. .travis.yml
    2. .github/PULL_REQUEST_TEMPLATE.md
    

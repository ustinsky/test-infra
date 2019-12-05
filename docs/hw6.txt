
Домашняя работа 6 (Debug)
1. kubectl debug
    1.1 Установим kubectl по инструкции https://github.com/aylei/kubectl-debug
        export PLUGIN_VERSION=0.1.1
        # linux x86_64
        curl -Lo kubectl-debug.tar.gz https://github.com/aylei/kubectl-debug/releases/download/v${PLUGIN_VERSION}/kubectl-debug_${PLUGIN_VERSION}_linux_amd64.tar.gz
        tar -zxvf kubectl-debug.tar.gz kubectl-debug
        sudo mv kubectl-debug /usr/local/bin/

        # if your kubernetes version is v1.16 or newer
        kubectl apply -f https://raw.githubusercontent.com/aylei/kubectl-debug/master/scripts/agent_daemonset.yml
        # if your kubernetes is old version(<v1.16), you should change the apiVersion to extensions/v1beta1, As follows
        wget https://raw.githubusercontent.com/aylei/kubectl-debug/master/scripts/agent_daemonset.yml
        sed -i '' '1s/apps\/v1/extensions\/v1beta1/g' agent_daemonset.yml
        kubectl apply -f agent_daemonset.yml
        mv agent_daemonset.yml agent_daemonset_old.yml

    1.2 Пробуем запустить strace и получаем ошибку
            
        1.2.1 Запускаем пробный под
            kubectl create ns test1
            kubectl run nginx --image=nginx --port=80 -n test1 --generator=run-pod/v1

        1.2.2 Из другого терминала заходим kubectl debug-ом 
            kubectl debug nginx -n test1 --port-forward

        1.2.3 Пробуем запустить strace
            bash-5.0# strace -p6 -c
            strace: test_ptrace_get_syscall_info: PTRACE_TRACEME: Operation not permitted
            strace: attach: ptrace(PTRACE_ATTACH, 6): Operation not permitted

        1.2.4 Видим ошибку. Нет прав PTRACE_TRACEME

        1.2.5 Ищем контейнер
            $ docker ps | grep ne
            00289bcf1ff1        nicolaka/netshoot:latest   "bash"                   10 seconds ago      Up 9 seconds                                   friendly_mirzakhani
            206bfb03f042        4689081edb10               "/storage-provisioner"   36 minutes ago      Up 36 minutes                                  k8s_storage-provisioner_storage-provisioner_kube-system_83577e77-9923-4777-bd3d-a64e2246850c_0
            dac0e02cb0fb        k8s.gcr.io/pause:3.1       "/pause"                 36 minutes ago      Up 36 minutes                                  k8s_POD_storage-provisioner_kube-system_83577e77-9923-4777-bd3d-a64e2246850c_0

        1.2.6 Смотрим docker capabilities
            $ docker inspect 00 | less 
            ...
             "CapAdd": null
            ...

        1.2.7 Права не добавляются

    1.3 Правим
    
        1.3.1 Меняем в файле agent_daemonset.yaml

            image: aylei/debug-agent:0.0.1
            на 
            image: aylei/debug-agent:latest

        1.3.2 Удаляем старый daemonset
            kubectl delete daemonset debug-agent

        1.3.3 Ставим новый
            kubectl apply -f 01-agent-daemonset.yaml

        1.3.4 Проверяем работу strace
            bash-5.0# strace -p7 -c
            strace: Process 7 attached
            ^Cstrace: Process 7 detached
            % time     seconds  usecs/call     calls    errors syscall
            ------ ----------- ----------- --------- --------- ----------------
            21.19    0.000153          76         2           writev
            21.05    0.000152          38         4           close
            16.20    0.000117          19         6           epoll_wait
            14.54    0.000105          52         2           sendfile
            5.26    0.000038          19         2           stat
            4.85    0.000035          17         2           accept4
            4.16    0.000030           7         4           recvfrom
            3.74    0.000027          13         2           openat
            3.46    0.000025          12         2           write
            2.35    0.000017           8         2           epoll_ctl
            1.94    0.000014           7         2           setsockopt
            1.25    0.000009           4         2           fstat
            ------ ----------- ----------- --------- --------- ----------------
            100.00    0.000722                    32           total

        1.3.5 Проверяем права
            $ docker inspect 22 | less
            ... 
            "CapAdd": [
                    "SYS_PTRACE",    
                    "SYS_ADMIN"
            ]
            ...

        1.3.6 Забавно получается 
            Образ nicolaka/netshoot:latest не меняется. Но меняется запуск. 
            Образ aylei/debug-agent берется другой. CapAdd для debug-agent остается тем же ("CapAdd": null). 
            Но запуск контенера netshoot меняется. Добавляются права SYS_PTRACE

2 iptables-tailer
    2.1 Установка https://github.com/piontec/netperf-operator
        git clone https://github.com/piontec/netperf-operator.git 

    2.2 Запустить манифесты
        kubectl apply -f ./deploy/crd.yaml
        kubectl apply -f ./deploy/rbac.yaml
        kubectl apply -f ./deploy/operator.yaml

    2.3 Запустим пример
        kubectl apply -f ./deploy/cr.yaml
        kubectl describe netperf.app.example.com/example

    2.4 Ставим политику (включаем логирование в iptables) и смотрим как изменился вывод
        kubectl apply -f kit/netperf-calico-policy.yaml
        kubectl delete -f netperf-operator/deploy/cr.yaml
        kubectl apply -f netperf-operator/deploy/cr.yaml
        kubectl describe netperf.app.example.com/example

    2.5 Получаем доступ SSH к ноде GKE
     
        2.5.1 Получаем IP адрес узлов кластера
            gcloud beta compute --project "studied-glow-255313" ssh --zone "us-central1-a" "gke-standard-cluster-3-default-pool-a7bab0cf-g9gm"

        2.5.2 Подключаемся к ноде по SSH и смотрим iptables
            iptables --list -nv | grep DROP - счетчики дропов ненулевы
            iptables --list -nv | grep LOG - счетчики с действием логирования ненулевые
            journalctl -k | grep calico

    2.6 Запустим iptailer
        2.6.1 Применим
            kubectl apply -f kit/iptables-tailer.yaml 
            kubectl describe daemonset kube-iptables-tailer -n kube-system

        2.6.2 Применим ServiceAccount
            kubectl apply -f kit/kit-serviceaccount.yaml
            kubectl apply -f kit/kit-clusterrole.yaml
            kubectl apply -f kit/kit-clusterrolebinding.yaml 
            kubectl describe daemonset kube-iptables-tailer -n kube-system

        2.6.3 Пересоздадим netperf
            kubectl delete -f netperf-operator/deploy/cr.yaml
            kubectl apply -f netperf-operator/deploy/cr.yaml
            kubectl describe netperf.app.example.com/example

        2.6.4 Проверяем
            kubectl get events -A
            kubectl describe pod --selector=app=netperf-operator
            Видим что пакеты дропаются

        2.7 Иcправим
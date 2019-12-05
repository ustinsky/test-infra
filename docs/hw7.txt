# ustinsky_platform
ustinsky Platform repository

Домашняя работа 7 (operators)
1. CRD CR Mysql

    1.1 Создадим 
        kubectl apply -f deploy/crd.yml
        kubectl apply -f deploy/cr.yml

    1.2 Взаимодействие с объектами
        kubectl get crd
        kubectl get mysqls.otus.homework
        kubectl describe mysqls.otus.homework mysql-instance


    1.3 Добавим спецификацию полей (добавляется поле validation)
        kubectl delete mysqls.otus.homework mysql-instance
        kubectl apply -f deploy/crd.yaml
        kubectl apply -f deploy/cr.yaml

    1.4 Запустим оператор
        apt install python3-pip
        pip3 install kopf
        PATH="$PATH:/home/lex/.local/bin/"
        pip3 install kubernetes jinja2
        kopf run mysql-operator.py

    1.5 Почему объект создался, хотя мы создали CR, до того, как запустили контроллер?
        Ответ из документации
        https://kopf.readthedocs.io/en/latest/walkthrough/starting/

        Note that the operator has noticed an object created before the operator was even started, 
        and handled it – since it was not handled before.

        Обратите внимание, что оператор заметил объект, созданный еще до того, как оператор был запущен,
         и обработал его - поскольку он не был обработан ранее.

    1.6 Удалим все ресурсы созданные контроллером
        kubectl delete mysqls.otus.homework mysql-instance
        kubectl delete deployments.apps mysql-instance
        kubectl delete pvc mysql-instance-pvc
        kubectl delete pv mysql-instance-pv
        kubectl delete svc mysql-instance

    1.7 Добавим код и запустим
        kopf run mysql-operator.py
        kubectl apply -f deploy/cr.yaml
        kubectl get pvc

    1.8 Заполним базу данных
        1.8.1 export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
        
        1.8.2 kubectl exec -it $MYSQLPOD -- mysql -u root -potuspassword -e "CREATE TABLE test ( id \
              smallint unsigned not null auto_increment, name varchar(20) not null, constraint \
              pk_example primary key (id) );" otus-database
        
        1.8.3 kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) \
              VALUES ( null, 'some data' );" otus-database 
                      

        1.8.4 kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) \
              VALUES ( null, 'some data-2' );" otus-database

    1.9 Посмотрим содержимое таблицы
        kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database


    1.10 Удалим 
        kubectl delete mysqls.otus.homework mysql-instance
        kubectl get pv
        kubectl get jobs.batch

    1.11 Создадим заново
        kubectl apply -f deploy/cr.yml
        kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
        kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
        export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
        kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database

        #kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
        mysql: [Warning] Using a password on the command line interface can be insecure.
        +----+-------------+
        | id | name        |
        +----+-------------+
        |  1 | some data   |
        |  2 | some data-2 |
        +----+-------------+



2 Создадим docker-контейнер
    2.1 Dockerfile 
        Dockerfile:
            FROM python:3.7
            COPY templates ./templates
            COPY mysql-operator.py ./mysql-operator.py
            RUN pip install kopf kubernetes pyyaml jinja2
            CMD kopf run /mysql-operator.py

    2.2 Отправлем в DockerHub
        docker build ./build/ --tag ustinsky/mysql-operator:v0.0.1
        docker images
        docker tag 07c37 ustinsky/mysql-operator:v0.0.1
        docker login
        docker push ustinsky/mysql-operator

    2.3 Скачаем 
        wget https://gist.githubusercontent.com/Evgenikk/581fa5bba6d924a3438be1e3d31aa468/raw/99429270c474cc434748e1058919e27df01d9a48/service-account.yml
        wget https://gist.githubusercontent.com/Evgenikk/581fa5bba6d924a3438be1e3d31aa468/raw/99429270c474cc434748e1058919e27df01d9a48/role.yml
        wget https://gist.githubusercontent.com/Evgenikk/581fa5bba6d924a3438be1e3d31aa468/raw/99429270c474cc434748e1058919e27df01d9a48/ClusterRoleBinding.yml
        wget https://gist.githubusercontent.com/Evgenikk/581fa5bba6d924a3438be1e3d31aa468/raw/619023d01e49ca3702357d4fded4d054cd523a9a/deploy-operator.yml

    2.4 Переустановим minikube
        minikube delete && minikube start

    2.5 Применим манифесты
        kubectl apply -f deploy/crd.yml
        kubectl apply -f service-account.yml
        kubectl apply -f role.yml
        kubectl apply -f ClusterRoleBinding.yml 
        kubectl apply -f deploy-operator.yml
        kubectl apply -f deploy/cr.yml

    2.6 Проверяем
        $ kubectl get pvc
        NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
        backup-mysql-instance-pvc   Bound    pvc-5f06a495-0624-49d7-aac0-4682b2cfaba2   1Gi        RWO            standard       2m47s
        mysql-instance-pvc          Bound    pvc-bcad5009-d14c-4ef2-aad4-993d6e337313   1Gi        RWO            standard       2m48s

    2.7 Заполним базу данных
        2.7.1 export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
        
        2.7.2 kubectl exec -it $MYSQLPOD -- mysql -u root -potuspassword -e "CREATE TABLE test ( id \
              smallint unsigned not null auto_increment, name varchar(20) not null, constraint \
              pk_example primary key (id) );" otus-database
        
        2.7.3 kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) \
              VALUES ( null, 'some data' );" otus-database 
                      

        2.7.4 kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) \
              VALUES ( null, 'some data-2' );" otus-database

    2.8 Посмотрим содержимое таблицы
        kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database

    2.9 Удалим
        kubectl delete mysqls.otus.homework mysql-instance
        kubectl get pv
        kubectl get jobs.batch

    2.10 И создадим заново
        kubectl apply -f deploy/cr.yml
        export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
        kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database

        #kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
        mysql: [Warning] Using a password on the command line interface can be insecure.
        +----+-------------+
        | id | name        |
        +----+-------------+
        |  1 | some data   |
        |  2 | some data-2 |
        +----+-------------+
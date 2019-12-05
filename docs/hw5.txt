
Домашняя работа 5 
...

 ISCSI
        0. Установка kubernetes через kubespray
            0.1 Скачиваем
                git clone https://github.com/kubernetes-sigs/kubespray.git
            
            0.2 Устанавливаем рекомендованные зависимости
                sudo pip install -r requirements.txt`;

            0.3 Копируем файл пример
                cp -rfp inventory/sample inventory/mycluster

            0.4 Редактируем файл
                nano inventory/mycluster/inventory.ini

            0.5 Применяем, ждем и пользуемся
                ansible-playbook -i inventory/mycluster/inventory.ini --become --become-user=root --user=lex --key-file=~/.ssh/id_rsa cluster.yml 


        1. Устанавливаем targetcli на Ubuntu 18.04
            apt -y install targetcli-fb

        2. Создаем LVM раздел
            parted /dev/sdb
            # mklabel msdos
            # q
            parted -s /dev/sdb unit mib mkpart primary 1 100% set 1 lvm on
            pvcreate /dev/sdb1
            vgcreate vg0 /dev/sdb1
            lvcreate -l 10%FREE -n base vg0
            mkfs.ext4 /dev/vg0/base

        3. Настраиваем targetcli
            targetcli
            /> ls
            /> backstores/block create name=iscsi-disk dev=/dev/vg0/base
            /iscsi create
            /iscsi и ls 
            iqn.2003-01.org.linux-iscsi.iscsi-1.x8664:sn.c67162716271617/tpg1/
            luns/ create /backstores/block/iscsi-disk
            set attribute authentication=0
            acls/
            create wwn=iqn.2019-09.com.example.srv01.initiator01
            cd / 
            ls 
            saveconfig
            exit
            (https://kifarunix.com/how-to-install-and-configure-iscsi-storage-server-on-ubuntu-18-04/)


    4. Настраиваем worker-node
        4.1 apt -y install open-iscsi
            yum install iscsi-initiator-utils

        4.2 настроим конфиг /etc/iscsi/initiatorname.iscsi, 
        внеся туда корректное имя, которое мы использовали ранее `iqn.2019-09.com.example.srv01.initiator01`

        4.3 добавим open-iscsi в автозагрузку и запустим:
            systemctl restart iscsid open-iscsi
            systemctl enable iscsid open-iscsi

    5. Проверяем
        5.1 Запускаем под
            kubectl apply -f kubernetes-storage/iscsi/01-iscsi-pod.yaml

        5.2 Зайдем на pod
            kubectl exec -it iscsi-pod -- /bin/bash

        5.3 Сохраним
            echo "ISCSI TEST!" > /mnt/iscsi-test.txt

        5.4 Создадим snapshot
            lvcreate --snapshot --size 1G  --name ss-01 /dev/vg0/base

        5.5 Перейдем обратно в под и удалим данные
            rm -rf /mnt/iscsi-test.txt

        5.6 Удалим сам pod
            kubectl delete -f kubernetes-storage/iscsi/01-iscsi-pod.yaml

        5.7 Отключим диск ISCSI
            targetcli 
            /> backstores/block delete iscsi-disk 

        5.8 Восстановимся из снапшота 
            lvconvert --merge /dev/vg0/ss-01

        5.9 Восстановим диск ISCSI
            targetcli
            /> backstores/block create name=iscsi-disk dev=/dev/vg0/base
            /> /iscsi/iqn.2003-01.org.linux-iscsi.iscsi-1.x8664:sn.c0904cfa5297/tpg1/
            /> luns/ create /backstores/block/iscsi-disk
            exit

        5.10 Снова запусти и проверим наличие файла
            kubectl apply -f kubernetes-storage/iscsi/01-iscsi-pod.yaml
            kubectl exec -it iscsi-pod -- /bin/bash
            cat /mnt/iscsi-test.txt

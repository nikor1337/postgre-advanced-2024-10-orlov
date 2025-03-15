# Установка, настройка и тестирование Greengage

## Установка: 
### Развернуто 3 хоста в ЯО со следующими параметрами:
- Имена: orlov-gg-master, orlov-gg-smdv, orlov-gg-segment1, orlov-gg-segment2
- OS: Ubuntu 22
- CPU: 8
- RAM: 16 GB
- DiskSize: 40 GB

*Дальнейшие действия выполняются на всех хостах*
### Подключение файловой системы xfs
1. Создать диск на 20 ГБ в ЯО
2. Подключить диск к ВМ в ЯО на вкладке Диски 
3. Выполнить команды:


    sudo apt-get install xfsprogs
    sudo modprobe -v xfs
    sudo fdisk -l  # в выводе найти свой диск, для примера у меня /dev/vdb смонтирован в /data
    sudo mkfs.xfs /dev/vdb
    sudo mkdir /data
    sudo mount -w /dev/vdb /data/
    
### Установка зависимостей:
    sudo apt install bash bzip2 iproute2 iputils-ping krb5-multidev libapr1 libaprutil1 libcurl3-gnutls libcurl4 libuuid1 libxml2 libyaml-0-2 less locales net-tools openssh-client openssh-server openssl perl rsync sed tar zip zlib1g libevent-dev libldap-dev libreadline8 openjdk-17-jdk python2.7 chrony

### Разрешение имен хостов
    sudo nano /etc/hosts
    # закомментить все существующие строки и всавить следуюшие (ip заменить на свои)
    10.94.12.67 orlov-gg-smdv
    10.94.12.4 orlov-gg-master
    10.94.12.98 orlov-gg-segment1
    10.94.12.27 orlov-gg-segment2
    sudo nano /etc/cloud/cloud.cfg
    #  comment row - update_etc_hosts > # - update_etc_hosts
   

### Параметры ядра
    sudo nano /etc/sysctl.conf
    # add new rows
    kernel.core_pipe_limit=0
    net.core.rmem_max=2097152
    net.core.rmem_default=26214400
    net.core.wmem_max=2097152
    net.core.wmem_default=26214400
    net.ipv6.conf.all.disable_ipv6=1             # Если используется IPv4
    net.ipv6.conf.default.disable_ipv6=1         # Если используется IPv4
    kernel.sysrq=1
    kernel.core_uses_pid=1
    kernel.shmmni=4096
    kernel.sem=250 2048000 200 8192
    kernel.msgmnb=65536
    kernel.msgmax=65536
    kernel.msgmni=2048
    net.ipv4.tcp_syncookies=1
    net.ipv4.conf.default.accept_source_route=0
    net.ipv4.tcp_max_syn_backlog=4096
    net.ipv4.conf.all.arp_filter=1
    net.ipv4.ip_local_port_range=10000 65535     # Влияет на значения портов Greengage DB
    net.ipv4.ipfrag_high_thresh=41943040
    net.ipv4.ipfrag_low_thresh=31457280
    net.ipv4.ipfrag_time=60
    net.core.netdev_max_backlog=10000
    vm.overcommit_memory=2
    vm.overcommit_ratio=95
    vm.swappiness=10
    vm.zone_reclaim_mode=0
    vm.dirty_expire_centisecs=500
    vm.dirty_writeback_centisecs=100
    vm.dirty_background_ratio=0
    vm.dirty_ratio=0
    vm.dirty_background_bytes=1610612736
    vm.dirty_bytes=4294967296
    
    sudo sysctl --system
### Лимиты системных ресурсов
    sudo nano /etc/security/limits.conf
    # add new rows
    gpadmin soft nofile 524288
    gpadmin hard nofile 524288
    gpadmin soft nproc 150000
    gpadmin hard nproc 150000
### Удаление объектов IPC
    sudo nano /etc/systemd/logind.conf
    # Edit row
    RemoveIPC=no
    sudo systemctl restart systemd-logind
### Выключение Transparent huge pages (THP)
    sudo nano /etc/systemd/system/disable-thp.service
    # add rows
    [Unit]
    Description=Disable Transparent Huge Pages
     
    [Service]
    Type=oneshot
    ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
    ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'
     
    [Install]
    WantedBy=multi-user.target
    
    # perform commands
    sudo systemctl daemon-reload
    sudo systemctl enable disable-thp.service
    sudo systemctl start disable-thp.service
### Лимиты подключения по SSH
    sudo nano /etc/ssh/sshd_config
    # change param value
    MaxStartups 1000:30:1022
    sudo systemctl restart sshd
### Настройка сети
    sudo sysctl -w net.core.rmem_max=26214400
    sudo sysctl -w net.core.rmem_default=26214400
    sudo sysctl -w net.core.wmem_max=26214400
    sudo sysctl -w net.core.wmem_default=26214400
    sudo ip link set mtu 9000 dev eth0
### Создание пользователя gpadmin
    sudo groupadd gpadmin
    sudo useradd gpadmin -r -m -g gpadmin
    sudo passwd gpadmin # set pass = 1
    sudo adduser gpadmin sudo
    sudo visudo
    # add row
    %sudo    ALL=(ALL)    NOPASSWD: ALL
    sudo chsh -s /bin/bash gpadmin
### Генерация SSH-ключей
    su - gpadmin
    bash
    ssh-keygen -t rsa -b 4096 # 3 раза нажать Enter
### Включение беспарольного SSH
1. На каждом хосте выполнить команду cat ~/.ssh/id_rsa.pub
2. В отдельном текстовом файле создать 4 строки с ssh ключами, пример:
        
       ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDfRxLisclbSWWEiUiLOFoVlofNZWM+ZgMEmFe53FYtb8pW9liOrLpccQC3wOmCNz8IEh620rUlJcdJZHDiXgr7+wUMTBpAlnXZUCQE52vxmhaHHZ00CxsuQskQSCAclRku03NYL/LgaWfkH/xTiWxexZamwm4mlSVfF+M0KT9XugicU6qRXAY7Ukm65kYS6qvJ0W06+w0qxSqPBDKFstFcWkhKOQ+YPQAtcFaZNWWJlZxkVG7ITiX+esUupE6p8I9DLWqIc/3Q37rMiOtPHGaGc7Ev7uZaOHMR2O9K3rBtq+/cfSsDLP1w2+3XJWxFFZLkFzZRwuUPYRcR+XTVrcUA5noht8gfGR3kUntjPAQ1oh3Qu4fiaILRZoIQijEohpnz9NTO15MA96BCIkE40OBrqJ31zRxhrOXzsq6rjf8PABrP8zrq3lDyDw6rUi3Q7hR4pI5sBT8I1ZfA1Z6wfYcP1Kg9RrBGD+45jeKU/iTBCHYRsXPTzbSjMpHAoMae0tcQEtasEiQjAMtRc+mLgZQgDGJYqAyVkg2Uu4g0AqF5GqJVYex2oTr98Xf5N8qSCHmdZaWRGT1vx0V/Yta4hPZFzN2Qk2zyNlUcWh11qReKgz4/wfpu/Vsc3PmY1x+sPNiROGg5hwjHDIU6RiVfG3cF1U8RsznNeU8a78Le3UjZPQ== gpadmin@orlov-gg-master
       ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDfuehKJ7dZdGDyjTf1I790sg8HB6cs/QQAr00e7JzUMeICFbXuiNi1sGAd4+jSg3PasVl2bRZ4YoF9rWf5ZoQYbJWsQurCw7ukjMUd19l1phm/bNTeJVkfK42y+BVW4HDrEfyD/QWDRln1ravJ8eKiu5WwF81//oOvEf2RQ1kyK7aAx/FdyxTNRJZbsKHIFdMZLYEdCHgl6+WfZjLO55Tyg8RG+Xnp1B25XJ6FV08jA8R3jtmatePOebKVgSrylCpUBUsnKdn7eqnbpN6DRklz+ughm643U3XGvwcTKwqG1JFzAm08upff9gi4ZDmKUYCLIvAOtc8W2ADP4IXqLfYbYKCE0GW0vp9zkPyLPtm1p0l9TzOnaTH3xqfl9ZBJdTYDkbhGoiMGbdkPP0/DV+vRNApFPtYyAlaAL7hbvLfJVocSa9NngeRVHN0io40P1i/Vxh25/KABRix0JU/SZ9znXxyq34064kUm20iP00HWiKlSPd7f04m6s+WbjlLxIG8g4MGh8Glq12kwXrLe/N0uPcdgbHYH9s3bMtdtH1Ke8mF1dnAkK42oDgYfFzXNz7siJKX7NUXxsg1Z69PzK9dFZwV+s03isc8bFb/yzG9a4IfMtUu/Xz7zea4iwySL6Tdhkd4zgUCc+UgrWBkwFpD7lVGrdWOhi5iLZVUpQm8m9w== gpadmin@orlov-gg-smdv
       ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCbqDwEfb+b2DDMpSwBOP9tziQSonFbX6YuMthj7lzXcZjBUnYJs0nlIYVxaAWFdov7W/CMbV5qmHiftasQBiVQIeMY7ITz4GzCBlXmKDvkfMTdQdIxGmKENY/qtGo1V4RDcTxncGZrYnZsntu3epthhvGAECJ3XK6foHsDqqZyNdA3G/s7YHWk1HANlRCvG7RqKzvzqO5A0n590pWNw22o4ag4YmK9656DLAPRNqV+4h5a8hAkObPtOF2g0hEqfFN3OjO38Y1k4Ei/ef7/LkPN77+R9cfyCp+NSsa5VVqLK5bfjndEa3tIUd1JJIO12Ehh6/RJcsbKu5GqAiXGwnawOYLj6LdE6ujlFaPVoMv2eshwWKhS9q50+zGYI86+c2hraiwgHyrUdbSWbOM338/aa1o8JsNANblBkFp8sOEhPbpvl3+IKN9zdNq8Y9mG5dRzNEG0hNItUR70VAc7b3Ma0Q7YTOkmPFZIayz/vk5jH0m5FfzAiyC5naV1IrW1A1c6u8OwYWLppPavqRdr8ogKXNYAABHbLmf3nsBemEfsxFNMO/vqyAizC0XQHe8XF5k/KWqvLkTivn1Q029uAEOQkivPvl+C/94nj2Ke1TtZF3ltpc1mTcoCvjLiSWAgLB2DdAF2h0CG0bnk4wSgDqGHcl6J1ISMinKLIEbZDJfdEQ== gpadmin@orlov-gg-segment1
       ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCpk4wnmpmp8tPaQKBAEaZeJ8wFViOt3aDx7c0jeDV5gkxaN7dAFqi4ujbbbcBQH+2uoLXpM3NG8/jTIYx/Ud1MYPI2bBoLmySEE2kLnM9NcEQmSDkGuRkmFrW+R4hgm3Typh/56BuCSAb3iEYxe6i5jZPpUlpLHKQszIEv+rrVX2xJjaMRfL/eiHrsTVXICLnQ+dY1jn3my5yaluWhf0NNwsrvlo//cYX3LYavVYokgz9saluGr5IqIIrWiTzbdiC2UiY/2tvS8v2DqeoagMdXGUr6tf4NmOAHX5uQDDvsQfGukd/kTToVNUXcxQip3biP8mKptroIGaRWnKgWcY3AKhgwTwU5OfUnCCni7q7fgj6vaTbuhAIyM8hGSPHC95H3KRQtwBH7zrSLRxfDD4/SrftOv56AAZLWDsg97ps3P38FvhR6CG+eq26sBnyZ9enMwzws7yEMghnz7XDV6cyqi3IN0dh8u5NOL0EdAamMFXEP00HdXvy8867EFMa7BLAeVD5Fyg9snpwYTzB7nXZ4/Wno0I7WgAc/3JJLW/sx0FvDrE/Z9ZEVF2frF5qLQpk1mb9bgd3FaHYW+TynCy5w+fvxTy9jcJ2JI/4CbqlCVgglqLFaQbtNl7aiKQGFd2uBtMbbzHS3Cy62tpGaPVsGTIlKu0JDvZVmjZdP8aHoyw== gpadmin@orlov-gg-segment2
3. На каждый хост в файл ~/.ssh/authorized_keys вставить эти 4 строки с ssh ключами
4. Для проверки выполнить ssh <hostnmae> - не должен затребовать пароль

### Клонирование репозитория Greengage DB
    cd
    git clone --recurse-submodules https://github.com/GreengageDB/greengage.git
    cd greengage
### Установка зависимостей
    sudo ./README.ubuntu.bash
    sudo ln -s python2 /usr/bin/python
    sudo locale-gen "en_US.UTF-8"
### Установка psutil и pyyaml
    sudo apt install python-pip
    pip2 install psutil pyyaml
### Сборка и установка Greengage DB
    cd ~/greengage
    ./configure --with-perl --with-python --with-libxml --with-gssapi --prefix=/usr/local/gpdb
    make -j$(nproc)
    # expected output: All of Greengage Database successfully made. Ready to install.
    sudo make -j$(nproc) install
    # expected output: Greengage Database installation complete.
    sudo chown -R gpadmin:gpadmin /usr/local/gpdb*
    sudo chgrp -R gpadmin /usr/local/gpdb*
    source /usr/local/gpdb/greengage_path.sh
## На мастер хосте
### Создание хост-файлов со всеми хостами

    cd ~/greengage
    nano hostfile_all_hosts
    # add rows
    orlov-gg-master
    orlov-gg-segment1
    orlov-gg-segment2
    orlov-gg-smdv
### Создание хост-файлов с сегмент-хостами
    cd ~/greengage    
    nano hostfile_all_hosts
    # add rows
    orlov-gg-segment1
    orlov-gg-segment2
## На всех хостах
### Создание хост-файлов со всеми хостами и Включение n-n беспарольного SSH

    cd ~/greengage
    nano hostfile_all_hosts
    # add rows
    orlov-gg-master
    orlov-gg-segment1
    orlov-gg-segment2
    orlov-gg-smdv
    gpssh-exkeys -f hostfile_all_hosts
## На мастер хосте
### Создание каталога для данных на мастере
    sudo mkdir -p /data/master
    sudo chown gpadmin:gpadmin /data/master
### Создание каталога для данных на резервном мастере
    gpssh -h orlov-gg-smdv -e 'sudo -n mkdir -p /data/master'
    gpssh -h orlov-gg-smdv -e 'sudo chown gpadmin:gpadmin /data/master'
### Создание каталогов для данных на сегмент-хостах
    gpssh -f hostfile_segment_hosts -e 'sudo mkdir -p /data/primary'
    gpssh -f hostfile_segment_hosts -e 'sudo mkdir -p /data/mirror'
    gpssh -f hostfile_segment_hosts -e 'sudo chown -R gpadmin /data/*'

# Инициализация СУБД Greengage
## На мастер хосте
    nano init_config
    # add rows
    ARRAY_NAME="Greengage DB cluster"
    SEG_PREFIX=gpseg
    PORT_BASE=6000
    declare -a DATA_DIRECTORY=(/data/primary /data/primary )
    MASTER_HOSTNAME=mdw
    MASTER_DIRECTORY=/data/master
    MASTER_PORT=5432
    TRUSTED_SHELL=ssh
    CHECK_POINT_SEGMENTS=8
    ENCODING=UNICODE
    MIRROR_PORT_BASE=7000
    declare -a MIRROR_DATA_DIRECTORY=(/data/mirror /data/mirror )
## Создание файла конфигурации базы данных
    nano init_config
    # add rows
    ARRAY_NAME="Greengage DB cluster"
    SEG_PREFIX=gpseg
    PORT_BASE=6000
    declare -a DATA_DIRECTORY=(/data/primary /data/primary )
    MASTER_HOSTNAME=orlov-gg-master
    MASTER_DIRECTORY=/data/master
    MASTER_PORT=5432
    TRUSTED_SHELL=ssh
    CHECK_POINT_SEGMENTS=8
    ENCODING=UNICODE
    MIRROR_PORT_BASE=7000
    declare -a MIRROR_DATA_DIRECTORY=(/data/mirror /data/mirror )
## Запуск утилиты инициализации
    gpinitsystem -c init_config -h hostfile_segment_hosts -s orlov-gg-smdv
    # expected output: Greengage Database instance successfully created
    export MASTER_DATA_DIRECTORY=/data/master/gpseg-1

# На мастере и резвервном мастере (orlov-gg-master, orlov-gg-smdv)
## Установка переменных окружения для Greengage DB
    nano .bashrc
    # add rows
    source /usr/local/gpdb/greengage_path.sh
    export MASTER_DATA_DIRECTORY=/data/master/gpseg-1
    export PGPORT=5432
    export PGUSER=gpadmin
    export PGDATABASE=postgres

# На остальных хостах
## Установка переменных окружения для Greengage DB
    nano .bashrc
    # add rows
    source /usr/local/gpdb/greengage_path.sh

# В случае перезагрузки хостов на каждом хосте выполнить следующие действия:
    sudo su - gpadmin
    sudo mount -w /dev/vdb /data1
    sudo sysctl -w net.core.rmem_max=26214400
    sudo sysctl -w net.core.rmem_default=26214400
    sudo sysctl -w net.core.wmem_max=26214400
    sudo sysctl -w net.core.wmem_default=26214400
    sudo ip link set mtu 9000 dev eth0
## На мастер хосте для проверки состояния кластера:
    gpstate
    gpstate -s
# Замеры
## Создаем распределенные таблицы с поколоночным и построчным хранением:
    postgres=# CREATE TABLE events_row (
               event_id bigserial,
               event_time timestamp,
               user_id bigint,
               data jsonb
           ) distributed by (user_id);
    CREATE TABLE
    postgres=# CREATE TABLE events_column (
               event_id bigserial,
               event_time timestamp,
               user_id bigint,
               data jsonb 
           ) WITH (appendoptimized=true, orientation=column) distributed by (user_id);
    CREATE TABLE
    postgres=# CREATE TABLE users (
                   id bigserial, name text
              ) distributed by (id);

### Загружаем данные в таблицы:
    INSERT INTO events_row (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 1000000), '{"action": "login"}'; #10х
    INSERT INTO events_row (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 1000000), '{"action": "logout"}'; #10х
    INSERT INTO events_row (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 1000000), '{"action": "purchase"}'; #10x
    INSERT INTO events_column (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 1000000), '{"action": "login"}'; #10х
    INSERT INTO events_column (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 1000000), '{"action": "logout"}'; #10х
    INSERT INTO events_column (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 1000000), '{"action": "purchase"}'; #10x
    INSERT INTO users (name)                                                            
    SELECT 'user' || generate_series(1, 1000000);  #2x

### Размер таблиц:
      postgres=# \dt+
                                          List of relations
       Schema |     Name      | Type  |  Owner  |       Storage        |  Size   | Description 
      --------+---------------+-------+---------+----------------------+---------+-------------
       public | events_column | table | gpadmin | append only columnar | 1414 MB | 
       public | events_row    | table | gpadmin | heap                 | 2334 MB | 
       public | users          | table | gpadmin | heap                 | 100 MB  | 
### Тестируем OLAP запросы:
#### Таблица с поколоночным хранением:
      postgres=# explain analyze select name, data from users join events_column on users.id = events_column.user_id;
                                                                 QUERY PLAN                                                                  
    ---------------------------------------------------------------------------------------------------------------------------------------------
    Gather Motion 4:1  (slice1; segments: 4)  (cost=0.00..7609.57 rows=30000000 width=35) (actual time=348.296..8171.038 rows=30000000 loops=1)
      ->  Hash Join  (cost=0.00..4097.32 rows=7500000 width=35) (actual time=383.669..4425.010 rows=7519770 loops=1)
            Hash Cond: (events_column.user_id = users.id)
            Extra Text: (seg2)   Hash chain length 1.6 avg, 8 max, using 322700 of 524288 buckets.
            ->  Seq Scan on events_column  (cost=0.00..690.88 rows=7500000 width=33) (actual time=0.123..824.712 rows=7519770 loops=1)
            ->  Hash  (cost=437.60..437.60 rows=250000 width=18) (actual time=381.646..381.646 rows=500946 loops=1)
                  ->  Seq Scan on users  (cost=0.00..437.60 rows=250000 width=18) (actual time=0.010..32.073 rows=500946 loops=1)
    Planning time: 7.651 ms
      (slice0)    Executor memory: 296K bytes.
      (slice1)    Executor memory: 45372K bytes avg x 4 workers, 45372K bytes max (seg0).  Work_mem: 23482K bytes max.
    Memory used:  128000kB
    Optimizer: Pivotal Optimizer (GPORCA)
    Execution time: 9610.229 ms
    (13 rows)
   
    Time: 9618.549 ms

#### Таблица с построчным хранением:
    postgres=# explain analyze select name, data from users join events_row on users.id = events_row.user_id;
                                                                    QUERY PLAN                                                                 
    --------------------------------------------------------------------------------------------------------------------------------------------
    Gather Motion 4:1  (slice1; segments: 4)  (cost=0.00..1221.53 rows=1000000 width=34) (actual time=444.875..8891.729 rows=32000000 loops=1)
      ->  Hash Join  (cost=0.00..1107.80 rows=250000 width=34) (actual time=443.495..5014.848 rows=8021088 loops=1)
            Hash Cond: (events_row.user_id = users.id)
            Extra Text: (seg2)   Hash chain length 1.6 avg, 8 max, using 322700 of 524288 buckets.
            ->  Seq Scan on events_row  (cost=0.00..441.73 rows=250000 width=32) (actual time=0.056..1103.788 rows=8021088 loops=1)
            ->  Hash  (cost=437.60..437.60 rows=250000 width=18) (actual time=443.071..443.071 rows=500946 loops=1)
                  ->  Seq Scan on users  (cost=0.00..437.60 rows=250000 width=18) (actual time=0.014..32.320 rows=500946 loops=1)
    Planning time: 5.651 ms
      (slice0)    Executor memory: 87K bytes.
      (slice1)    Executor memory: 45132K bytes avg x 4 workers, 45132K bytes max (seg0).  Work_mem: 23482K bytes max.
    Memory used:  128000kB
    Optimizer: Pivotal Optimizer (GPORCA)
    Execution time: 10462.952 ms
    (13 rows)
   
    Time: 10469.081 ms

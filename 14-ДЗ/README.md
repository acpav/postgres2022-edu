# Домашнее задание 14
## Репликация

На cloud.yandex.ru создаю четыре одинаковых инстанса (Debian 11, postgresql 13).  
* postgres-edu14-01
* postgres-edu14-02
* postgres-edu14-03
* postgres-edu14-04

На первом, втором и третем инстансе создаю базы testdb01, testdb02 и testdb03 соответственно.  
Создаю пользователей user1..3 с паролем 123  

    create database testDB01;
    create role user1 with password '123';
    grant all privileges ON DATABASE testdb01 TO user1;


На трех инстансах создаю таблицы test и test2 с единственным полем id integer  

На инстансе postgres-edu14-01 создаю публикацию **create publication pb1 for table test;**

И на postgres-edu14-02 публикацию **create publication pb2 for table test2;**

На двух первых инстансах в **pg_hba.conf** добавляю возможность подключения с хостов локальной сети  

    host    all             all             10.128.0.0/24           md5

и в **postgresql.conf** включаю **wal_level = logical** и **listen_addresses = 'localhost, 10.128.0.x'**

На первом инстансе (postgres-edu14-01) создаю подписку

    create subscription sb1 CONNECTION 'host=10.128.0.23 port=5432 user=user2 password=123 dbname=testdb02' publication pb2 with ( copy_data = true);

На втором (postgres-edu14-02) подписку

    create subscription sb2 CONNECTION 'host=10.128.0.6 port=5432 user=user1 password=123 dbname=testdb01' publication pb1 with ( copy_data = true);

На третьем (postgres-edu14-03) подписываюсь к двум таблицам

    create subscription sb1 CONNECTION 'host=10.128.0.23 port=5432 user=user2 password=123 dbname=testdb02' publication pb2 with ( copy_data = true);
    create subscription sb2 CONNECTION 'host=10.128.0.6 port=5432 user=user1 password=123 dbname=testdb01' publication pb1 with ( copy_data = true);

Теперь при добавлении записей в таблицу test или test2 на первом и втором сервере соответственно записи появляются сразу на трех серверах.

На третем и четвертом интансе **wal_level = replica**

На третем инстансе разрешаю подключение для репликации для инстанса postgres-edu14-04  
    
    host    replication     all             10.128.0.30/32          md5

На четвертом инстансе останавливаю службу postgres **sudo systemctl stop postgresql**. Удаляю все из каталога **/var/lib/postgresql/13/main**.  
Выполняю бекап  

    sudo -u postgres pg_basebackup -p 5432 -h postgres-edu14-03 -R -D /var/lib/postgresql/13/main

Теперь на postgres-edu14-04 появилась точная копия БД testdb03

Логической репликация не выполнялась, пока не выдал пользователям user1 и user2 права на запись в таблицы test и test2.  

Во время настройки физ репликации, при очистке каталога **/var/lib/postgresql/13/main** удалил случайно и сам каталог main. После создал его заново и postgres не стартовал пока не установил права на каталог **sudo chmod 0700 /var/lib/postgresql/13/main** (хотя бекап с настройкой репликации прошел успешно).
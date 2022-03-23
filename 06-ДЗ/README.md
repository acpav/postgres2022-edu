# Домашнее задание 06
## Установка и настройка PostgreSQL

Создаю инстанс в облаке Яндекс под именем postgers-edu06.  
Устанавливаю postrgesql 14

Создаю таблицу test и добавляю пару строк.  

Создаю новый диск на 10 Гб, тип HDD, имя назначаю newhdd10  

Присоединяю диск к виртуальной машине postgers-edu06  

С помощью **fdisk** создаю новый раздел на новом диске.  
Создаю файловую систему **sudo mkfs.ext4 /dev/vdb**  
Создаю новый каталог /mnt/data
Добавляю новый диск в /etc/fstab с точкой монтирования /mnt/data  

Перезагружаю инстанс, проверяю что новый диск примонтировался.  
Устанавливаю владельцем каталога /mant/data пользователя postgres.

Останаливаю postgres **sudo systemctl stop postgresql**  
Переношу содержимое /var/lib/postgres/14 в /mnt/data  
Запускаю postgres **sudo systemctl start postgresql**  

Вижу, что postgres не стартовал  

> acpavel@postgers-edu06:~$ sudo -u postgres pg_lsclusters 
> Ver Cluster Port Status Owner     Data directory              Log file  
> 14  main    5432 down   <unknown> /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.lo  

Смотрю статус службы **sudo systemctl status postgresql@14-main.service**

> ● postgresql@14-main.service - PostgreSQL Cluster 14-main  
>     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)  
>     Active: failed (Result: protocol) since Wed 2022-03-23 13:55:53 UTC; 7min ago  
>    Process: 977 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 14-main start (code=exited, status=1/FAILURE)  

> Mar 23 13:55:53 postgers-edu06 systemd[1]: Starting PostgreSQL Cluster 14-main...  
> Mar 23 13:55:53 postgers-edu06 postgresql@14-main[977]: Error: /var/lib/postgresql/14/main is not accessible or does not exist  
> Mar 23 13:55:53 postgers-edu06 systemd[1]: postgresql@14-main.service: Can't open PID file /run/postgresql/14-main.pid (yet?) after start: Operation not per>  
> Mar 23 13:55:53 postgers-edu06 systemd[1]: postgresql@14-main.service: Failed with result 'protocol'.  
> Mar 23 13:55:53 postgers-edu06 systemd[1]: Failed to start PostgreSQL Cluster 14-main.  

Вижу ошибку **Error: /var/lib/postgresql/14/main is not accessible or does not exist**  

Меняю строчку в **postgresql.conf** (**sudo vi /etc/postgresql/14/main/postgresql.conf**)
> data_directory = '/var/lib/postgresql/14/main'          # use data in another directory  
на  
> data_directory = '/mnt/data/14/main'          # use data in another directory  

Postgres стартует  
> acpavel@postgers-edu06:~$ sudo -u postgres pg_lsclusters  
> Ver Cluster Port Status Owner    Data directory    Log file  
> 14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log  

Останавливаю инстанс **postgers-edu06**  
Создаю новый под именем **postgers-edu06-2**  

Отключаю диск от первого инстанса, и подключаю ко второму.

Подключаюсь к **postgers-edu06-2**  и устанавливаю postgers

Очищаю каталог **/var/lib/postgres** и в fstab указываю этот каталог как точку монтирования диска **newhdd10**.  

После перезагрузки на инстансе **postgers-edu06-2** проверяю что данные добавленные на инстансе **postgers-edu06** на месте  

> acpavel@postgres-edu06-2:~$ sudo -u postgres psql  
> psql (14.2 (Ubuntu 14.2-1.pgdg20.04+1))  
> Type "help" for help.  
>  
> postgres=`#` select * from test;  
 > c1   
> ----  
 > 1  
 > 1  
> (2 rows)  
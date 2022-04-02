# Домашнее задание 09
## Работа с журналами

1.
Настроил создание контрольной точки каждые 30 секунд

2.
Получил lsn

    postgres=# SELECT pg_current_wal_insert_lsn();
    pg_current_wal_insert_lsn 
    ---------------------------
    4/108BC550
    (1 row)

Запустил нагрузочное тестирование

    acpavel@postgres-edu08:~$ sudo -u postgres pgbench -c8 -P 60 -T 3600 -U postgres postgres

После 10 минут снова получил lsn

    postgres=# SELECT pg_current_wal_insert_lsn();
    pg_current_wal_insert_lsn 
    ---------------------------
    4/513DAD00
    (1 row)

3.
Вычислил размер данных в журнале

    postgres=# SELECT pg_size_pretty('4/513DAD00'::pg_lsn - '4/108BC550'::pg_lsn);
    pg_size_pretty 
    ----------------
    1035 MB
    (1 row)

В среднем 1035 / 20 = **52 МБ одна контрольная точка**

4.
Первые 10 минут контрольные точки создавались строго каждые 30 секунд, потому что было запущено нагрузочное тестирование.
Затем тестирование выключено и контрольные точки стали создаваться только после изменения данных.

    acpavel@postgres-edu08:~$ sudo cat /var/log/postgresql/postgresql-13-main.log | grep "checkpoint starting: time"
    2022-04-02 12:26:50.118 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:27:20.075 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:27:50.119 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:28:20.071 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:28:50.115 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:29:20.043 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:29:50.131 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:30:20.074 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:30:50.063 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:31:20.111 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:31:50.063 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:32:20.146 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:32:50.111 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:33:20.075 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:33:50.147 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:34:20.067 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:34:50.151 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:35:20.083 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:35:50.159 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:36:20.099 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:36:50.079 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:37:50.065 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:43:50.367 UTC [1047] LOG:  checkpoint starting: time
    2022-04-02 12:45:20.135 UTC [1047] LOG:  checkpoint starting: time

5.
Замер транзакций в секунду с синхронном режиме

    synchronous_commit = on

    acpavel@postgres-edu08:~$ sudo -u postgres pgbench -c8 -P 60 -T 3600 -U postgres postgres
    starting vacuum...end.
    progress: 60.0 s, 372.7 tps, lat 21.423 ms stddev 16.369
    progress: 120.0 s, 400.6 tps, lat 19.935 ms stddev 15.327
    progress: 180.0 s, 418.0 tps, lat 19.109 ms stddev 13.942
    progress: 240.0 s, 392.0 tps, lat 20.381 ms stddev 14.617
    progress: 300.0 s, 391.1 tps, lat 20.416 ms stddev 14.928
    progress: 360.0 s, 369.7 tps, lat 21.610 ms stddev 16.579
    progress: 420.0 s, 357.8 tps, lat 22.334 ms stddev 16.474
    progress: 480.0 s, 386.4 tps, lat 20.669 ms stddev 15.135
    progress: 540.0 s, 437.7 tps, lat 18.247 ms stddev 13.115
    progress: 600.0 s, 427.3 tps, lat 18.689 ms stddev 13.775

И в асинхронном...

    synchronous_commit = off

    acpavel@postgres-edu08:~$ sudo -u postgres pgbench -c8 -P 60 -T 3600 -U postgres postgres
    starting vacuum...end.
    progress: 60.0 s, 2616.4 tps, lat 3.020 ms stddev 1.291
    progress: 120.0 s, 2598.4 tps, lat 3.042 ms stddev 1.319
    progress: 180.0 s, 2709.5 tps, lat 2.918 ms stddev 1.240
    progress: 240.0 s, 2788.9 tps, lat 2.834 ms stddev 1.182
    progress: 300.0 s, 2766.2 tps, lat 2.857 ms stddev 1.211
    progress: 360.0 s, 2710.9 tps, lat 2.916 ms stddev 1.255
    progress: 420.0 s, 2816.3 tps, lat 2.806 ms stddev 1.201
    progress: 480.0 s, 2658.6 tps, lat 2.973 ms stddev 1.256
    progress: 540.0 s, 2701.6 tps, lat 2.926 ms stddev 1.229
    progress: 600.0 s, 2541.1 tps, lat 3.111 ms stddev 1.333

6.
Создаю кластер с контрольной суммой страниц

    sudo pg_createcluster 13 test_cl -- --data-checksums

Создаю таблицу и вставляю в нее 100 записей

    postgres=# create table test_table(id integer);
    postgres=# INSERT INTO test_table SELECT s.id FROM generate_series(1,100) AS s(id);

Получаю физическое расположение таблицы в ФС

    postgres=# SELECT pg_relation_filepath('test_table');
    pg_relation_filepath 
    ----------------------
    base/13448/16384
    (1 row)

После остановки кластера меняю несколько байт в таблице

    acpavel@postgres-edu08:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/13/test_cl/base/13448/16384 oflag=dsync conv=notrunc bs=1 count=8
    8+0 records in
    8+0 records out
    8 bytes copied, 0.0128229 s, 0.6 kB/s

Пробую выборку

    postgres=# select * from test_table ;

Ошибка

    WARNING:  page verification failed, calculated checksum 54617 but expected 19869
    ERROR:  invalid page in block 0 of relation base/13448/16384

Игнорирую контрольные суммы

    postgres=# alter system set ignore_checksum_failure = on;
    postgres=# select pg_reload_conf();

Выборка пролучается

    postgres=# select * from test_table limit 10;
    id 
    ----
    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    (10 rows)

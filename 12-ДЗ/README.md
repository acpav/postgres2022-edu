# Домашнее задание 12
## Нагрузочное тестирование и тюнинг PostgreSQL

Установил PostgreSQL из пакетов собираемых postgres.org на Ubuntu 20.04  

Установил sysbench из репозитария Ubuntu  

Скачал sysbench-tpcc с git  

Созадал БД sbtest, пользователя sbtest с паролем 123.  

Запутил создание таблиц с данными для теста --tables=5 --scale=5

    sudo -u postgres ./tpcc.lua prepare --pgsql-user=sbtest --pgsql-password=123 --pgsql-db=sbtest --time=120 --threads=2 --report-interval=1 --tables=5 --scale=5 --use_fk=0 --trx_level=RC --db-driver=pgsql

Запуск нагрузочного тестирования с праметрами postgres по умолчанию в 20 потоков

    sudo -u postgres ./tpcc.lua --pgsql-user=sbtest --pgsql-password=123 --pgsql-db=sbtest --time=1200 --threads=20 --report-interval=60 --tables=5 --scale=5 --use_fk=0 --trx_level=RC --db-driver=pgsql run

Результат 32.64 транзакций в секунду в среднем за 20 минут  

    SQL statistics:
        queries performed:
            read:                            519765
            write:                           538449
            other:                           86866
            total:                           1145080
        transactions:                        39187  (32.64 per sec.)
        queries:                             1145080 (953.67 per sec.)
        ignored errors:                      4401   (3.67 per sec.)
        reconnects:                          0      (0.00 per sec.)

    General statistics:
        total time:                          1200.7013s
        total number of events:              39187

    Latency (ms):
            min:                                    0.70
            avg:                                  612.60
            max:                                 5996.88
            95th percentile:                     2082.91
            sum:                             24005827.36

    Threads fairness:
        events (avg/stddev):           1959.3500/73.89
        execution time (avg/stddev):   1200.2914/0.18

Применяю параметры из pgtune для DB Type "Data warehouse", 4 Гб ОЗУ, 2 ЦПУ, ССД диска.  

Увеличились параметры памяти, используемой PG, изменились настройки паралельности для 2 ЦПУ.  

    max_connections = 100
    shared_buffers = 1GB
    effective_cache_size = 3GB
    maintenance_work_mem = 512MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 500
    random_page_cost = 1.1
    effective_io_concurrency = 200
    work_mem = 26214kB
    min_wal_size = 4GB
    max_wal_size = 16GB
    max_worker_processes = 2
    max_parallel_workers_per_gather = 1
    max_parallel_workers = 2
    max_parallel_maintenance_workers = 1

Результат 78.93 (+142% от дефолта) транзакций в секунду в среднем за 20 минут  

    SQL statistics:
        queries performed:
            read:                            1291291
            write:                           1340435
            other:                           214734
            total:                           2846460
        transactions:                        94790  (78.93 per sec.)
        queries:                             2846460 (2370.24 per sec.)
        ignored errors:                      12975  (10.80 per sec.)
        reconnects:                          0      (0.00 per sec.)

    General statistics:
        total time:                          1200.9132s
        total number of events:              94790

    Latency (ms):
            min:                                    0.74
            avg:                                  253.22
            max:                                 3222.07
            95th percentile:                      539.71
            sum:                             24002528.23

    Threads fairness:
        events (avg/stddev):           4739.5000/36.87
        execution time (avg/stddev):   1200.1264/0.08

Отключаю fsync и synchronous_commit.

    fsync = off
    synchronous_commit = off

Снижаем защиту от сбоя ОС или оборудования, получаем больше производительность.  
134.65 транзакций в секунду (+71% от настроек pgtune)

    SQL statistics:
        queries performed:
            read:                            2145206
            write:                           2218137
            other:                           356182
            total:                           4719525
        transactions:                        161715 (134.65 per sec.)
        queries:                             4719525 (3929.71 per sec.)
        ignored errors:                      17111  (14.25 per sec.)
        reconnects:                          0      (0.00 per sec.)

    General statistics:
        total time:                          1200.9834s
        total number of events:              161715

    Latency (ms):
            min:                                    0.66
            avg:                                  148.40
            max:                                10729.90
            95th percentile:                      458.96
            sum:                             23999198.63

    Threads fairness:
        events (avg/stddev):           8085.7500/96.08
        execution time (avg/stddev):   1199.9599/0.08
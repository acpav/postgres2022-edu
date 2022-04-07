# Домашнее задание 10
## Механизм блокировок

1.
Включил параметр **log_lock_waits = on**, установил **deadlock_timeout = 200ms**. В журнал стали записываться сообщения о длительных блокировках.

    2022-04-06 16:19:31.767 UTC [12207] postgres@testdb LOG:  process 12207 still waiting for ShareLock on transaction 500 after 200.186 ms
    2022-04-06 16:19:31.767 UTC [12207] postgres@testdb DETAIL:  Process holding the lock: 12206. Wait queue: 12207.
    2022-04-06 16:19:31.767 UTC [12207] postgres@testdb CONTEXT:  while updating tuple (0,16) in relation "testtab"
    2022-04-06 16:19:31.767 UTC [12207] postgres@testdb STATEMENT:  update testtab set id = 5 where id = 4;

2.
В трех соединениях выполняю UPDATE одной и той же строки таблицы testtab.
Сначала в соединении 12206, затем в 12254 и 12207.

    testdb=# SELECT pid, locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
    FROM pg_locks where database = 16384;
    pid  | locktype | relation | virtxid | xid |       mode       | granted 
    -------+----------+----------+---------+-----+------------------+---------
    12291 | relation | pg_locks |         |     | AccessShareLock  | t
    12254 | relation | testtab  |         |     | RowExclusiveLock | t
    12207 | relation | testtab  |         |     | RowExclusiveLock | t
    12206 | relation | testtab  |         |     | RowExclusiveLock | t
    12254 | tuple    | testtab  |         |     | ExclusiveLock    | t
    12207 | tuple    | testtab  |         |     | ExclusiveLock    | f
    (6 rows)

Все три соединения заблокировали таблицу (locktype = relation) в режиме RowExclusive. Блокировка получена (granted = t), потому что режим блокировки RowExclusive не конфликтует с RowExclusive.
Соединение 12254 заблокировало версию строки (locktype = tuple) в режиме Exclusive и ожидает блокировку номера транзакции.
Соединение 12207 пыталось заблокировать версию строки, но не смогло(granted = f), потому что уже наложена блокировка в режиме Exclusive. 

3.

Записи в журнале при взаимоблокировках трех транзакций

    2022-04-07 11:37:38.160 UTC [19285] postgres@testdb STATEMENT:  update testtab set id = id + 4 where id = 1;
    2022-04-07 11:37:44.473 UTC [19286] postgres@testdb LOG:  process 19286 detected deadlock while waiting for ShareLock on transaction 541 after 200.133 ms
    2022-04-07 11:37:44.473 UTC [19286] postgres@testdb DETAIL:  Process holding the lock: 19285. Wait queue: .
    2022-04-07 11:37:44.473 UTC [19286] postgres@testdb CONTEXT:  while updating tuple (0,2) in relation "testtab"
    2022-04-07 11:37:44.473 UTC [19286] postgres@testdb STATEMENT:  update testtab set id = id + 4 where id = 2;
    2022-04-07 11:37:44.473 UTC [19286] postgres@testdb ERROR:  deadlock detected
    2022-04-07 11:37:44.473 UTC [19286] postgres@testdb DETAIL:  Process 19286 waits for ShareLock on transaction 541; blocked by process 19285.
            Process 19285 waits for ShareLock on transaction 540; blocked by process 19283.
            Process 19283 waits for ShareLock on transaction 542; blocked by process 19286.
            Process 19286: update testtab set id = id + 4 where id = 2;
            Process 19285: update testtab set id = id + 4 where id = 1;
            Process 19283: update testtab set id = id + 4 where id = 3;
    2022-04-07 11:37:44.473 UTC [19286] postgres@testdb HINT:  See server log for query details.
    2022-04-07 11:37:44.473 UTC [19286] postgres@testdb CONTEXT:  while updating tuple (0,2) in relation "testtab"
    2022-04-07 11:37:44.473 UTC [19286] postgres@testdb STATEMENT:  update testtab set id = id + 4 where id = 2;
    2022-04-07 11:37:44.473 UTC [19283] postgres@testdb LOG:  process 19283 acquired ShareLock on transaction 542 after 15986.111 ms
    2022-04-07 11:37:44.473 UTC [19283] postgres@testdb CONTEXT:  while updating tuple (0,3) in relation "test

Мы видим что блокировка длится больше больше 200 ms
    2022-04-07 11:37:44.473 UTC [19286] postgres@testdb LOG:  process 19286 detected deadlock while waiting for ShareLock on transaction 541 after 200.133 ms

Видим ошибку deadlock detected
    2022-04-07 11:37:44.473 UTC [19286] postgres@testdb ERROR:  deadlock detected

Видим что сеанс 19286 ждет 19285, 19285 ждет 19283, а 19283 ждет первый сеанс 19286.

Видим текст запросов для каждого сеанса из взаимоблокировки.
    Process 19286: update testtab set id = id + 4 where id = 2;

4.

Воспроизвожу ситуацию когда два UPDATE без where блокируют друг друга.

Добавляю в таблицу миллион записей.

    testdb=# insert into testtab (id)  (SELECT generate_series(1,1000000));

Создаю индекс по полю id

    testdb=# create index on testtab (id);

Первый сеанс отключаем последовательное сканирование, что бы использовалось сканирование индекса.

    SET enable_seqscan = off;

В первом сеансе выполняю запрос

    testdb=# update testtab set id = id + 1;

Второй сеанс добавляем к ключевому полю 1000000, что бы первые записи оказались в хвосте индекса.

    testdb=# update testtab set id = id + 1000000;

В первом сеансе появляется ошибка, потому что первый сеанс дошел до заблокированных записей второго сеанса.

    ERROR:  deadlock detected
    DETAIL:  Process 12254 waits for ShareLock on transaction 532; blocked by process 12206.
    Process 12206 waits for ShareLock on transaction 531; blocked by process 12254.
    HINT:  See server log for query details.
    CONTEXT:  while rechecking updated tuple (22123,120) in relation "testtab"

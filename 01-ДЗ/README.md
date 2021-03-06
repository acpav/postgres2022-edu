# Домашнее задание 01
## Работа с уровнями изоляции транзакции в PostgreSQL

**auto commit** выключен  
Уровень изоляции в обоих сеансах **Read Committed**  

В первой сессии делаю **insert into persons(first_name, second_name) values('sergey', 'sergeev');**  
Во второй сессии (**select * from persons**) не видно новой записи из первой сессии, потому что в первой сессии не зафиксированна транзакция, а postgres не допускает "грязного чтения" ни на одном уровне изоляции.  

В первой сессии фиксирую транзакцию **commit**  
Во второй сессии (**select * from persons**) появляется новая запись из первой сессии, потому что на уровне изоляции Read Committed видны зафиксированные транзакции из других сессий.  
Два последовательных оператора SELECT могут видеть разные данные даже в рамках одной транзакции, если какие-то другие транзакции зафиксируют изменения после запуска первого SELECT, но до запуска второго.  

Завершаю транзакцию во второй сессии. Меняю уровень изоляции на Repeatable Read (**set transaction isolation level repeatable read;**)  
В первой сессии делаю **insert into persons(first_name, second_name) values('sveta', 'svetova');**  
Во второй сессии (**select * from persons**) не видно новой записи из первой сессии, потому что в первой сессии не зафиксированна транзакция, а postgres не допускает "грязного чтения" ни на одном уровне изоляции.  

В первой сессии фиксирую транзакцию **commit**  
Во второй сессии (**select * from persons**) не видна новая запись из первой сессии, потому что на уровне изоляции Repeatable Read видны только те данные, которые были зафиксированы до начала транзакции, но не видны незафиксированные данные и изменения, произведённые другими транзакциями в процессе выполнения данной транзакции.  
Последовательные команды SELECT в одной транзакции видят одни и те же данные; они не видят изменений, внесённых и зафиксированных другими транзакциями после начала их текущей транзакции.

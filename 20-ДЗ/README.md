# Домашнее задание 20
## Секционирование таблицы

Используется БД demo https://edu.postgrespro.ru/demo-medium.zip

Создаем секционированную таблицу **bookings_RANGE**, метод разбиения **RANGE**, ключ **book_date**  

    create TABLE bookings.bookings_RANGE (
        book_ref bpchar(6) NOT NULL,
        book_date timestamptz NOT NULL,
        total_amount numeric(10, 2) NOT NULL
    ) PARTITION by RANGE (book_date);

Источником данных будет существующая таблица **bookings.bookings**.  
Получаем все book_date с группировкой по году и месяцу

    SELECT 
    EXTRACT(year FROM book_date)::char(4) || '-' || EXTRACT(month FROM book_date)::char(2)
    FROM bookings.bookings
    group by EXTRACT(year FROM book_date)::char(4) || '-' || EXTRACT(month FROM book_date)::char(2)
    ;

    2017-4
    2017-5
    2017-6
    2017-7
    2017-8

Создаем секции  

    CREATE TABLE bookings_RANGE_201704 PARTITION OF bookings_RANGE
        FOR VALUES FROM ('2017-04-01') TO ('2017-05-01');
        
    CREATE TABLE bookings_RANGE_201705 PARTITION OF bookings_RANGE
        FOR VALUES FROM ('2017-05-01') TO ('2017-06-01');
        
    CREATE TABLE bookings_RANGE_201706 PARTITION OF bookings_RANGE
        FOR VALUES FROM ('2017-06-01') TO ('2017-07-01');
        
    CREATE TABLE bookings_RANGE_201707 PARTITION OF bookings_RANGE
        FOR VALUES FROM ('2017-07-01') TO ('2017-08-01');
        
    CREATE TABLE bookings_RANGE_201708 PARTITION OF bookings_RANGE
        FOR VALUES FROM ('2017-08-01') TO ('2017-09-01');
    

Создаем секцию по умолчанию  

    CREATE TABLE bookings_RANGE_DEFAULT PARTITION OF bookings_RANGE
        DEFAULT;

Заливаем данные из таблицы **bookings** в **bookings_range**

    insert into bookings_range select * from bookings;  

Добавляем первичный ключ

    ALTER TABLE bookings.bookings_range ADD CONSTRAINT bookings_range_pkey PRIMARY KEY (book_date, book_ref);

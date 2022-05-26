# Домашнее задание 19
## Работа с индексами, join'ами, статистикой

Используется БД demo https://edu.postgrespro.ru/demo-medium.zip

#### Запрос  

    select
        f.flight_no,
        SUM(tf.amount)
    from
        ticket_flights tf
    inner join
    flights f 
    on
        tf.flight_id = f.flight_id
    left join
    boarding_passes bp 
    on
        bp.ticket_no = tf.ticket_no
    where
        bp.ticket_no isnull
        and tf.fare_conditions = 'Business'
    group by
        f.flight_no
    order by
        SUM(tf.amount) desc
    limit 10
    ;

Выбираем стоимость билетов на места бизнес класса с группировкой по номеру самолета, среди билетов по которым небыло посадки в самолет.

План выполнения запроса  

    QUERY PLAN                                                                                                                               
    -----------------------------------------------------------------------------------------------------------------------------------------
    Limit  (cost=77612.50..77612.53 rows=10 width=39)                                                                                        
    ->  Sort  (cost=77612.50..77614.28 rows=710 width=39)                                                                                  
            Sort Key: (sum(tf.amount)) DESC                                                                                                  
            ->  Finalize GroupAggregate  (cost=77411.96..77597.16 rows=710 width=39)                                                         
                Group Key: f.flight_no                                                                                                     
                ->  Gather Merge  (cost=77411.96..77577.64 rows=1420 width=39)                                                             
                        Workers Planned: 2                                                                                                   
                        ->  Sort  (cost=76411.93..76413.71 rows=710 width=39)                                                                
                            Sort Key: f.flight_no                                                                                          
                            ->  Partial HashAggregate  (cost=76369.43..76378.31 rows=710 width=39)                                         
                                    Group Key: f.flight_no                                                                                   
                                    ->  Hash Join  (cost=38132.46..76270.65 rows=19757 width=13)                                             
                                        Hash Cond: (tf.flight_id = f.flight_id)                                                            
                                        ->  Parallel Hash Anti Join  (cost=35542.02..73113.35 rows=19757 width=10)                         
                                                Hash Cond: (tf.ticket_no = bp.ticket_no)                                                     
                                                ->  Parallel Seq Scan on ticket_flights tf  (cost=0.00..31963.41 rows=102019 width=24)       
                                                    Filter: ((fare_conditions)::text = 'Business'::text)                                   
                                                ->  Parallel Hash  (cost=21821.90..21821.90 rows=789290 width=14)                            
                                                    ->  Parallel Seq Scan on boarding_passes bp  (cost=0.00..21821.90 rows=789290 width=14)
                                        ->  Hash  (cost=1448.64..1448.64 rows=65664 width=11)                                              
                                                ->  Seq Scan on flights f  (cost=0.00..1448.64 rows=65664 width=11)                          

#### Создаем индекс на класс места в самолете  

    create index ticket_flights_fare_conditions_idx on ticket_flights (fare_conditions);

#### План запроса после создания индекса  

    QUERY PLAN                                                                                                                                                                   
    -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Limit  (cost=68995.05..68995.08 rows=10 width=39)                                                                                                                            
    ->  Sort  (cost=68995.05..68996.83 rows=710 width=39)                                                                                                                      
            Sort Key: (sum(tf.amount)) DESC                                                                                                                                      
            ->  Finalize GroupAggregate  (cost=68794.51..68979.71 rows=710 width=39)                                                                                             
                Group Key: f.flight_no                                                                                                                                         
                ->  Gather Merge  (cost=68794.51..68960.19 rows=1420 width=39)                                                                                                 
                        Workers Planned: 2                                                                                                                                       
                        ->  Sort  (cost=67794.48..67796.26 rows=710 width=39)                                                                                                    
                            Sort Key: f.flight_no                                                                                                                              
                            ->  Partial HashAggregate  (cost=67751.98..67760.86 rows=710 width=39)                                                                             
                                    Group Key: f.flight_no                                                                                                                       
                                    ->  Hash Join  (cost=38132.89..67647.74 rows=20848 width=13)                                                                                 
                                        Hash Cond: (tf.flight_id = f.flight_id)                                                                                                
                                        ->  Parallel Hash Anti Join  (cost=35542.45..64477.58 rows=20848 width=10)                                                             
                                                Hash Cond: (tf.ticket_no = bp.ticket_no)                                                                                         
                                                ->  Parallel Index Scan using ticket_flights_fare_conditions_idx on ticket_flights tf  (cost=0.43..23336.76 rows=100740 width=24)
                                                    Index Cond: ((fare_conditions)::text = 'Business'::text)                                                                   
                                                ->  Parallel Hash  (cost=21821.90..21821.90 rows=789290 width=14)                                                                
                                                    ->  Parallel Seq Scan on boarding_passes bp  (cost=0.00..21821.90 rows=789290 width=14)                                    
                                        ->  Hash  (cost=1448.64..1448.64 rows=65664 width=11)                                                                                  
                                                ->  Seq Scan on flights f  (cost=0.00..1448.64 rows=65664 width=11)                                                              

Видно, что используется новый индекс  

    ->  Parallel Index Scan using ticket_flights_fare_conditions_idx on ticket_flights tf  (cost=0.43..23336.76 rows=100740 width=24)
        Index Cond: ((fare_conditions)::text = 'Business'::text)                                                                   

#### Создаем индекс для полнотекстового поиска  

    create index passenger_name_gin on tickets using gin (to_tsvector('english', passenger_name));

Индексируем колонку с именем пассажира таблицы билетов.  

Запрос использующий индекс  

    select
    t.ticket_no ,
    t.passenger_name 
    from tickets t 
    where t.passenger_name @@ to_tsquery('english', 'VLADIMIR')
    limit 10;

#### Создаем частичный индекс  

    create index ticket_flights_amount_idx on ticket_flights (amount) where fare_conditions <> 'Economy'; 

Сейчас для условия
    
    where tf.amount  = 54555 and tf.fare_conditions = 'Economy';

используется **Parallel Seq Scan on ticket_flights**

а для условий

    where tf.amount  = 54555 and tf.fare_conditions = 'Buisness';  

**Index Scan using ticket_flights_amount_idx on ticket_flights**  

#### Создаем составной индекс  

    create index aircrafts_data_aircraft_code_range_idx on aircrafts_data (aircraft_code, range);


Запрос с кросс соединением, каждая строка таблицы ticket_flights соединяется со строкой таблицы boarding_passes

    select
    tf.ticket_no,
    bp.ticket_no 
    from 
    ticket_flights tf 
    cross join
    boarding_passes bp;

Запрос с полным соединением таблиц ticket_flights и boarding_passes с соединением по номеру билета, включая пустые значения в певрой и второй таблице  

    select
    tf.ticket_no,
    bp.ticket_no 
    from 
    ticket_flights tf
    full join
    boarding_passes bp 
    on tf.ticket_no  = bp.ticket_no;

Предыдущий запрос можно заменить на  

    select
    tf.ticket_no,
    bp.ticket_no 
    from 
    ticket_flights tf
    left join
    boarding_passes bp 
    on tf.ticket_no  = bp.ticket_no
    
    union

    select
    tf.ticket_no,
    bp.ticket_no 
    from 
    ticket_flights tf
    right join
    boarding_passes bp 
    on tf.ticket_no  = bp.ticket_no;
    ;


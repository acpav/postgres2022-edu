# Домашнее задание 23
## Триггеры, поддержка заполнения витрин
```sql
--Создаю таблицы скриптом hw_triggers.sql  

--Добавляю первичный ключ на поле good_name в таблице good_sum_mart

    ALTER TABLE pract_functions.good_sum_mart ADD CONSTRAINT good_sum_mart_pk PRIMARY KEY (good_name);

--Создаю процедуру добавления или обновления таблицы good_sum_mart

    create or replace procedure good_sum_mart_update(l_goods_id int4, l_sales_qty int4) as
    $$
    begin
        
        insert
        into pract_functions.good_sum_mart
        (good_name, sum_sale)
        select g.good_name, coalesce(gsm.sum_sale, 0) + (l_sales_qty * g.good_price)
        from pract_functions.goods as g
        left join
        pract_functions.good_sum_mart as gsm
        on g.good_name = gsm.good_name
        where g.goods_id = l_goods_id
        ON CONFLICT (good_name) DO UPDATE SET sum_sale = EXCLUDED.sum_sale
        ;

    end
    $$
    language plpgsql
    ;

--Создаю триггерную функцию  

    create or replace function good_sum_mart_trig() returns trigger as
    $$
    begin
        
        if (TG_OP = 'INSERT') then
            call good_sum_mart_update(new.good_id, new.sales_qty);
        elseif (TG_OP = 'DELETE') then
            call good_sum_mart_update(old.good_id, -old.sales_qty);
        elseif (TG_OP = 'UPDATE') then
            if (new.good_id = old.good_id) then
                call good_sum_mart_update(new.good_id, new.sales_qty-old.sales_qty);
            else
                call good_sum_mart_update(old.good_id, -old.sales_qty);
                call good_sum_mart_update(new.good_id, new.sales_qty);
            end if;
        elseif (TG_OP = 'TRUNCATE') then
            truncate good_sum_mart;
        end if;
        
        return null;

    end
    $$
    language plpgsql
    ;

--drop trigger if exists sales_after_change on pract_functions.sales;

--Сам триггер на добавление, удаление и обновление записей в таблице продаж  

    create trigger sales_after_change
    after insert or delete or update 
    on pract_functions.sales
    FOR EACH ROW EXECUTE function good_sum_mart_trig();

--Триггер на очистку таблицы продаж  

    create trigger sales_after_truncate
    after truncate  
    on pract_functions.sales
    EXECUTE function good_sum_mart_trig();

--Добавляю данные  

    INSERT INTO sales (good_id, sales_qty) VALUES (1, 13);

    INSERT INTO sales (good_id, sales_qty) VALUES (1, 20);

    INSERT INTO sales (good_id, sales_qty) VALUES (2, 1);

--Добавляю новый товар и продаю его  

    INSERT INTO goods (goods_id, good_name, good_price)
    VALUES 	(3, 'Печенье', 20);

    INSERT INTO sales (good_id, sales_qty) VALUES (3, 5);

--Удаляю продажу  

    delete from sales where sales_id = 15;

--Меняю количество проданных товаров  

    update sales set sales_qty = 2 where sales_id = 12;

--Меняю номеклатуру проданного товара  

    update sales set good_id = 3 where sales_id = 10;

--Очищаю таблицу  

    truncate sales ;

--Добавляю перваначальные данные  

    INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

--Для корректного обновления "витрины" при изменении цены товара, нужно хранить в таблице продаж цену, по которой товар был продан.  
Например спички стоили 0.5 рубля, поменяли цену в таблице товаров на 1 рубль. Теперь при отмене продажи совершенной по старой цене, в отчете спишется не верная сумма. Если хранить цену продажи, такого не случилось бы.  
Или можно хранить в таблице good_sum_mart общее количество проданного товара, тогда можно списывать сумму продажи "по среднему".  
```
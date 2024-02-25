# Сравнение хранения данных в двух видах таблиц - реляционная и таблица с большим jsonb полем

1. Создадим 2 таблицы
   
   Реляционная:
   ```
     create table order_item
    (
        id                bigserial primary key,
        order_id          varchar(20) not null,
        sku               varchar(20) not null,
        bar_code          varchar(13),
        bar_codes         jsonb,
        name              varchar(20) not null,
        brand             varchar(20) not null,
        category          int,
        count             int,
        count_pick        numeric(8, 3),
        count_courier     numeric(8, 3),
        count_to_remove   numeric(8, 3),
        count_actual      numeric(8, 3),
        count_wait_return numeric(8, 3),
        count_returned    numeric(8, 3),
        position          numeric(8, 3),
        price_regular     int,
        price_base        int,
        price             int,
        price_weight      int,
        total_price       int,
        unit              varchar(20),
        image             varchar(50),
        image_high_array  jsonb,
        marking           jsonb,
        weight_barcodes   jsonb,
        stock_data        jsonb
    );
   CREATE INDEX sku_index ON order_item (sku);
   CREATE INDEX order_id_index ON order_item (order_id);
   ```

   Таблица с jsonb полем:
   ```
    CREATE TABLE table_jsonb
   (
       id   SERIAL PRIMARY KEY,
       order_id varchar(20) not null,
       items jsonb
   );

   CREATE INDEX order_id_table_json_index ON table_jsonb (order_id);
   ```

3. Посмотрим стратегию TOAST

   order_item:
   
   ```
     select attname,
       atttypid::regtype,
       case pg_attribute.attstorage
           when 'p' THEN 'plain'
           when 'e' THEN 'external'
           when 'm' THEN 'main'
           when 'x' THEN 'extended'
           end as storage
    from pg_attribute
    where attrelid = 'order_item'::regclass
      and attnum > 0;
   ```
    attname | atttypeid | storage
    --- | --- | --- 
    id | bigint | plain
    order_id | character varying | extended
    sku | character varying | extended
    bar_code | character varying | extended
    bar_codes | jsonb | extended
    name | character varying | extended
    brand | character varying | extended
    category | integer | plain
    count | integer | plain
    count_pick | numeric | main
    count_courier | numeric | main
    count_to_remove | numeric | main
    count_actual | numeric | main
    count_wait_return | numeric | main
    count_returned | numeric | main
    position | numeric | main
    price_regular | integer | plain
    price_base | integer | plain
    price | integer | plain
    price_weight | integer | plain
    total_price | integer | plain
    unit | character varying | extended
    image | character varying | extended
    image_high_array | jsonb | extended
    marking | jsonb | extended
    weight_barcodes | jsonb | extended
    stock_data | jsonb | extended



   table_jsonb:

   ```
      select attname,
       atttypid::regtype,
       case pg_attribute.attstorage
           when 'p' THEN 'plain'
           when 'e' THEN 'external'
           when 'm' THEN 'main'
           when 'x' THEN 'extended'
           end as storage
      from pg_attribute
      where attrelid = 'table_jsonb'::regclass
        and attnum > 0;
   ```

     attname | atttypeid | storage
    --- | --- | --- 
    id | integer | plain
    order_id | character varying | extended
    items| jsonb | extended
   
    // todo: описание полей 


5. Заполняем таблицы
   
   Реляционную таблицу заполняем 300000 записей, а таблицу с jsonb заполняем 100000 в каждой json будет по 3 товара
6. Смотрим TOAST таблички
   ```
      SELECT c1.oid, c1.reltoastrelid, c2.relname
      FROM pg_class AS c1
               LEFT JOIN pg_class AS c2
                         ON c1.reltoastrelid = c2.oid
      WHERE c1.relname = 'order_item';
   ```
    relnamespace | relname
     --- | --- 
    pg_toast | pg_toast_2877391

    имя нашей toast таблицы pg_toast_2877391

    ```
       select * from pg_toast.pg_toast_2877391; // записей нет
    ```

    смотрим тоже самое для table_jsonb таблицы
    ```
       select pg_class.relnamespace::regnamespace, relname
      from pg_class
      where oid = (select reltoastrelid
                   from pg_class
                   where relname = 'table_jsonb');
    ```

    relnamespace | relname
     --- | --- 
    pg_toast | pg_toast_2877402

    ```
       select * from pg_toast.pg_toast_2877402; // 200000 записей
    ```
7. Смотрим занимаемое место вместе с индексами и toast таблицей

   ```
      SELECT relname AS table_name,
       pg_size_pretty(pg_total_relation_size(C.oid)) AS total_size
      FROM pg_class C
               LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
      WHERE nspname NOT IN ('pg_catalog', 'information_schema')
        AND C.relkind <> 'i'
        AND relname in ('table_jsonb', 'order_item')
      ORDER BY pg_total_relation_size(C.oid) DESC;
   ```

   table_name | total_size
     --- | --- 
    order_item | 284 MB
    table_jsonb | 272 MB

   Примерно одинаковые размеры, попробем теперь запустить скрипт, чтобы обновить все записи в обеих таблицах и посмотрим как изменится их размер
   
   Повторяем запрос
   ```
      SELECT relname AS table_name,
       pg_size_pretty(pg_total_relation_size(C.oid)) AS total_size
      FROM pg_class C
               LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
      WHERE nspname NOT IN ('pg_catalog', 'information_schema')
        AND C.relkind <> 'i'
        AND relname in ('table_jsonb', 'order_item')
      ORDER BY pg_total_relation_size(C.oid) DESC;
   ```

    table_name | total_size
     --- | --- 
    order_item | 354 MB
    table_jsonb | 543 MB

   таблица с jsonb увеличилась почти в 2 раза, реляционная таблица увеличилась меньше
   
   Если запустить обновление еще раз, то размер уже не так быстро растет, но таблица table_jsonb все равно увеличивается сильнее
   
   table_name | total_size
     --- | --- 
    order_item | 358 MB 
    table_jsonb | 609 MB

   Очистим таблицы от мертвых записей:
   ```
      VACUUM FULL table_jsonb, order_item;

      SELECT relname AS table_name,
       pg_size_pretty(pg_total_relation_size(C.oid)) AS total_size
      FROM pg_class C
               LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
      WHERE nspname NOT IN ('pg_catalog', 'information_schema')
        AND C.relkind <> 'i'
        AND relname in ('table_jsonb', 'order_item')
      ORDER BY pg_total_relation_size(C.oid) DESC;
   ```

   table_name | total_size
     --- | --- 
    order_item | 280 MB 
    table_jsonb | 274 MB
   
9. Сравниваем операции вставки/обновления

   Обновление:
   
      ```
          EXPLAIN ANALYZE UPDATE table_jsonb set body = 'большой json из 10 товаров' where order_id = 'JlbNjZ;
   
         Update on table_jsonb  (cost=0.42..8.44 rows=0 width=0) (actual time=2.856..2.857 rows=0 loops=1)
           ->  Index Scan using order_id_table_json_index on table_jsonb  (cost=0.42..8.44 rows=1 width=38) (actual time=0.042..0.043 rows=1 loops=1)
                 Index Cond: ((order_id)::text = 'JlbNjZ'::text)
         Planning Time: 0.229 ms
         Execution Time: 2.874 ms

   
      ```

     ```
         EXPLAIN ANALYZE UPDATE order_item SET count_pick = 120 WHERE order_id='tehTbk'
   
         Update on order_item  (cost=0.42..20.49 rows=0 width=0) (actual time=0.338..0.338 rows=0 loops=1)
           ->  Index Scan using order_id_index on order_item  (cost=0.42..20.49 rows=4 width=20) (actual time=0.068..0.081 rows=3 loops=1)
                 Index Cond: ((order_id)::text = 'tehTbk'::text)
         Planning Time: 0.060 ms
         Execution Time: 0.351 ms
   
     ```

     Пробую обновить order_item по id 10 штук
     ```
        EXPLAIN ANALYZE UPDATE order_item SET count_pick = 22 WHERE id BETWEEN 1030 and 1039;

        Update on order_item  (cost=0.42..34.47 rows=0 width=0) (actual time=0.700..0.701 rows=0 loops=1)
           ->  Index Scan using order_item_pkey on order_item  (cost=0.42..34.47 rows=10 width=20) (actual time=0.130..0.194 rows=10 loops=1)
                 Index Cond: ((id >= 1030) AND (id <= 1039))
         Planning Time: 0.465 ms
         Execution Time: 0.718 ms

     ``` 
      обновление order_item происходит быстрее
   
     Вставка:
   
      ```
          EXPLAIN ANALYZE INSERT INTO table_jsonb (order_id, body) VALUES ('123, 'большой json из 10 товаров')
   
         Insert on table_jsonb  (cost=0.00..0.01 rows=0 width=0) (actual time=3.360..3.360 rows=0 loops=1)
           ->  Result  (cost=0.00..0.01 rows=1 width=94) (actual time=0.006..0.006 rows=1 loops=1)
         Planning Time: 0.035 ms
         Execution Time: 3.375 ms

   
      ```
   
      ```
         EXPLAIN ANALYSE INSERT INTO order_item (order_id, sku, bar_code, bar_codes, name, brand, category, count, count_pick, count_courier,
                        count_to_remove, count_actual, count_wait_return, count_returned, position, price_regular,
                        price_base, price, price_weight, total_price, unit, image, image_high_array, marking,
                        weight_barcodes, stock_data)
         VALUES ... 10 строк с одинаковым order_id

      
         Insert on order_item  (cost=0.00..0.15 rows=0 width=0) (actual time=0.447..0.448 rows=0 loops=1)
           ->  Values Scan on ""*VALUES*""  (cost=0.00..0.15 rows=10 width=746) (actual time=0.036..0.073 rows=10 loops=1)
         Planning Time: 0.328 ms
         Execution Time: 0.474 ms

         Вставка в реляционную таблицу значительно быстрее
      ```


С данными типами полей и искуственными данными реляционная таблица имеет преимущество в скорости обновления и скорости вставки. При обновлении таблиц, таблица с jsonb увеличивается сильнее. Возможно с другими данными, будет другой результат. Если реляционная таблица будет занимать больше места и поля типа extended после сжатия попадут в toast, то время обновления/вставки увеличится.

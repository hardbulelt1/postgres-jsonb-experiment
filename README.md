# postgres-jsonb-experiment
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
        body jsonb
    );
   ```

3. Посмотри стратегию TOAST
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

    // todo: описание полей 


4. Заполняем таблицы
   
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
    order_item | 538 MB 
    table_jsonb | 344 MB

   таблица с jsonb увеличилась почти в 2 раза, реляционная таблица увеличилась меньше

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

   

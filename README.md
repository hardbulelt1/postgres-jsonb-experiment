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

2. Посмотри стратегию TOAST
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

   

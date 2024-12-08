# Задание 2: Специальные случаи использования индексов

# Партиционирование и специальные случаи использования индексов

1. Удалите прошлый инстанс PostgreSQL - `docker-compose down` в папке `src` и запустите новый: `docker-compose up -d`.

2. Создайте партиционированную таблицу и заполните её данными:

    ```sql
    -- Создание партиционированной таблицы
    CREATE TABLE t_books_part (
        book_id     INTEGER      NOT NULL,
        title       VARCHAR(100) NOT NULL,
        category    VARCHAR(30),
        author      VARCHAR(100) NOT NULL,
        is_active   BOOLEAN      NOT NULL
    ) PARTITION BY RANGE (book_id);

    -- Создание партиций
    CREATE TABLE t_books_part_1 PARTITION OF t_books_part
        FOR VALUES FROM (MINVALUE) TO (50000);

    CREATE TABLE t_books_part_2 PARTITION OF t_books_part
        FOR VALUES FROM (50000) TO (100000);

    CREATE TABLE t_books_part_3 PARTITION OF t_books_part
        FOR VALUES FROM (100000) TO (MAXVALUE);

    -- Копирование данных из t_books
    INSERT INTO t_books_part 
    SELECT * FROM t_books;
    ```

3. Обновите статистику таблиц:
   ```sql
   ANALYZE t_books;
   ANALYZE t_books_part;
   ```
   
   *Результат:*
   Query returned successfully in 1 secs 682 msec.

4. Выполните запрос для поиска книги с id = 18:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part WHERE book_id = 18;
   ```
   
   *План выполнения:*
```
"Seq Scan on t_books_part_1 t_books_part  (cost=0.00..1032.99 rows=1 width=32) (actual time=0.013..4.970 rows=1 loops=1)"
"  Filter: (book_id = 18)"
"  Rows Removed by Filter: 49998"
"Planning Time: 0.315 ms"
"Execution Time: 4.988 ms"
```
   
   *Объясните результат:*
   последовательно проходится по всем строкам и ищет что нужно, но в этот раз тут нет встроенного индекса по book_id, т.к. в таблице t_books_part оно не являетс primary key


5. Выполните поиск по названию книги:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
```
   "Append  (cost=0.00..3100.01 rows=3 width=33) (actual time=7.654..24.214 rows=1 loops=1)"
"  ->  Seq Scan on t_books_part_1  (cost=0.00..1032.99 rows=1 width=32) (actual time=7.652..7.654 rows=1 loops=1)"
"        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
"        Rows Removed by Filter: 49998"
"  ->  Seq Scan on t_books_part_2  (cost=0.00..1033.00 rows=1 width=33) (actual time=9.580..9.581 rows=0 loops=1)"
"        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
"        Rows Removed by Filter: 50000"
"  ->  Seq Scan on t_books_part_3  (cost=0.00..1034.01 rows=1 width=34) (actual time=6.968..6.968 rows=0 loops=1)"
"        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
"        Rows Removed by Filter: 50001"
"Planning Time: 0.542 ms"
```
   
   *Объясните результат:*
   т.к. наша таблица разделена на три, то поиск выполняется последовательно, но по трем таблицам поменьше одновременно, что ускоряет наш запрос

6. Создайте партиционированный индекс:
   ```sql
   CREATE INDEX ON t_books_part(title);
   ```
   
   *Результат:*
   CREATE INDEX Query returned successfully in 721 msec.

7. Повторите запрос из шага 5:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
```
"Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.054..0.102 rows=1 loops=1)"
"  ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.053..0.054 rows=1 loops=1)"
"        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
"  ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.027..0.027 rows=0 loops=1)"
"        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
"  ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.019..0.019 rows=0 loops=1)"
"        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
"Planning Time: 0.680 ms"
"Execution Time: 0.138 ms"
```
   
   *Объясните результат:*
   теперь в каждой из трех таблиц дополнительно поиск ускоряется индексом, поэтому общий запрос ускорился

8. Удалите созданный индекс:
   ```sql
   DROP INDEX t_books_part_title_idx;
   ```
   
   *Результат:*
   DROP INDEX
   Query returned successfully in 298 msec.

9. Создайте индекс для каждой партиции:
   ```sql
   CREATE INDEX ON t_books_part_1(title);
   CREATE INDEX ON t_books_part_2(title);
   CREATE INDEX ON t_books_part_3(title);
   ```
   
   *Результат:*
   CREATE INDEX
   Query returned successfully in 1 secs 240 msec.

10. Повторите запрос из шага 5:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part 
    WHERE title = 'Expert PostgreSQL Architecture';
    ```
    
    *План выполнения:*
```
"Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.024..0.053 rows=1 loops=1)"
"  ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.023..0.024 rows=1 loops=1)"
"        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
"  ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.014..0.014 rows=0 loops=1)"
"        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
"  ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.013..0.013 rows=0 loops=1)"
"        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
"Planning Time: 0.798 ms"
"Execution Time: 0.084 ms"
```
    
    *Объясните результат:*
    Индекс для каждой отдельной таблички ускорил запрос сильнее, чем один индекс для мастер-таблицы. Т.к. индекс для мастер-таблицы в каждой отдельной табличке проводил бы поиск по лишним значениям, а здесь каждый индекс оптимизирован под значения каждой конкретной таблицы, что позволило существенно ускорить выполнение запроса.

1.  Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_part_1_title_idx;
    DROP INDEX t_books_part_2_title_idx;
    DROP INDEX t_books_part_3_title_idx;
    ```
    
    *Результат:*
    DROP INDEX
    Query returned successfully in 282 msec.

2.  Создайте обычный индекс по book_id:
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id);
    ```
    
    *Результат:*
    CREATE INDEX
Query returned successfully in 279 msec.

3.  Выполните поиск по book_id:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part WHERE book_id = 11011;
    ```
    
    *План выполнения:*
```
"Index Scan using t_books_part_1_book_id_idx on t_books_part_1 t_books_part  (cost=0.29..8.31 rows=1 width=32) (actual time=1.040..1.041 rows=1 loops=1)"
"  Index Cond: (book_id = 11011)"
"Planning Time: 0.487 ms"
"Execution Time: 1.070 ms"
```
    
    *Объясните результат:*
    т.к. изначально партиции проводились именно по book_id, планировщик смог понять, в какой именно табличке будет находиться нужное нам значение. так же общий индекс разделился на три индекса (t_books_part_1_book_id_idx и еще два), опять же потому что изначальные партиции проводились по book_id

1.  Создайте индекс по полю is_active:
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active);
    ```
    
    *Результат:*
    CREATE INDEX

    Query returned successfully in 308 msec.

2.  Выполните поиск активных книг с отключенным последовательным сканированием:
    ```sql
    SET enable_seqscan = off;
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE is_active = true;
    SET enable_seqscan = on;
    ```
    
    *План выполнения:*
    пустота
    
    *Объясните результат:*
    Индекс по is_active не позволяет однозначно найти все нужные нам строки без последовательного сканирования, т.к. is_active низкоселективный параметр - внутри категории is_active = true нам все еще нужно провести последовательно сканирование чтобы найти все строки, удовлетворяющие этому условию, так что планировщик не справился с задачей это сделать с отключенной функцией.

3.  Создайте составной индекс:
    ```sql
    CREATE INDEX t_books_author_title_index ON t_books(author, title);
    ```
    
    *Результат:*
    CREATE INDEX
    Query returned successfully in 837 msec.

4.  Найдите максимальное название для каждого автора:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, MAX(title) 
    FROM t_books 
    GROUP BY author;
    ```
    
    *План выполнения:*
```
"HashAggregate  (cost=3475.00..3485.01 rows=1001 width=42) (actual time=106.894..107.046 rows=1003 loops=1)"
"  Group Key: author"
"  Batches: 1  Memory Usage: 193kB"
"  ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=21) (actual time=0.007..15.993 rows=150000 loops=1)"
"Planning Time: 0.325 ms"
"Execution Time: 107.143 ms"
```
    
    *Объясните результат:*
    для каждого автора ему нужно просканировать соответствующие ему книги и вывести максимальное, так что сократить время выполнения запроса при помощи индекса не вышло.

5.  Выберите первых 10 авторов:
    ```sql
    EXPLAIN ANALYZE
    SELECT DISTINCT author 
    FROM t_books 
    ORDER BY author 
    LIMIT 10;
    ```
    
    *План выполнения:*
```
"Limit  (cost=0.42..56.61 rows=10 width=10) (actual time=0.141..0.471 rows=10 loops=1)"
"  ->  Result  (cost=0.42..5625.42 rows=1001 width=10) (actual time=0.140..0.468 rows=10 loops=1)"
"        ->  Unique  (cost=0.42..5625.42 rows=1001 width=10) (actual time=0.139..0.465 rows=10 loops=1)"
"              ->  Index Only Scan using t_books_author_title_index on t_books  (cost=0.42..5250.42 rows=150000 width=10) (actual time=0.136..0.352 rows=1318 loops=1)"
"                    Heap Fetches: 3"
"Planning Time: 0.127 ms"
"Execution Time: 0.498 ms"

```
    *Объясните результат:*
    Индекс тут хорошо сработал, во время выполнения запроса не потребовалось даже обращаться напрямую к таблице, хватило данных чисто из индекса (Index Only), там лежат уже отсортированные уникальные значения по author, что позволяет легко и быстро достать первые 10

1.  Выполните поиск и сортировку:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE author LIKE 'T%'
    ORDER BY author, title;
    ```
    
    *План выполнения:*

```
"Sort  (cost=3100.29..3100.33 rows=15 width=21) (actual time=23.095..23.097 rows=1 loops=1)"
"  Sort Key: author, title"
"  Sort Method: quicksort  Memory: 25kB"
"  ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=21) (actual time=23.058..23.061 rows=1 loops=1)"
"        Filter: ((author)::text ~~ 'T%'::text)"
"        Rows Removed by Filter: 149999"
"Planning Time: 0.172 ms"
"Execution Time: 23.153 ms"
```
    
    *Объясните результат:*
    Тут индекс не позволил ускорить работу запроса, т.к. 1) сортировка проводится по author, title, а данных по title у индекса нет. 

2.  Добавьте новую книгу:
    ```sql
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true);
    COMMIT;
    ```
    
    *Результат:*
WARNING:  there is no transaction in progress
COMMIT

Query returned successfully in 305 msec.
3.  Создайте индекс по категории:
    ```sql
    CREATE INDEX t_books_cat_idx ON t_books(category);
    ```
    
    *Результат:*
    Query returned successfully in 420 msec.

4.  Найдите книги без категории:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
```
"Index Scan using t_books_cat_idx on t_books  (cost=0.29..8.16 rows=1 width=21) (actual time=0.030..0.032 rows=1 loops=1)"
"  Index Cond: (category IS NULL)"
"Planning Time: 1.471 ms"
"Execution Time: 0.049 ms"
```    
    *Объясните результат:*
    индекс по категории позволил быстро найти строки, где category is NULL и поэтому индекс все круто ускорил

5.  Создайте частичные индексы:
    ```sql
    DROP INDEX t_books_cat_idx;
    CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL;
    ```
    
    *Результат:*



6.  Повторите запрос из шага 22:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
```
"Index Scan using t_books_cat_null_idx on t_books  (cost=0.12..7.99 rows=1 width=21) (actual time=1.762..1.768 rows=1 loops=1)"
"Planning Time: 0.427 ms"
"Execution Time: 1.795 ms"
```
    
    *Объясните результат:*
    Индекс работает медленнее, чем предыдущий (правда я прогнала несколько раз и он варьируется сильно от 0,030 мс до 1,8 мс), возможно это связано с тем что таблица т_букс забита рандомными значениями и в зависимости от количества NULL категорий в табличке но это догадка

1.  Создайте частичный уникальный индекс:
    ```sql
    CREATE UNIQUE INDEX t_books_selective_unique_idx 
    ON t_books(title) 
    WHERE category = 'Science';
    
    -- Протестируйте его
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150002, 'Unique Science Book', 'Author 1', 'Science', true);
    
    -- Попробуйте вставить дубликат
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150003, 'Unique Science Book', 'Author 2', 'Science', true);
    
    -- Но можно вставить такое же название для другой категории
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150004, 'Unique Science Book', 'Author 3', 'History', true);
    ```
    
    *Результат:*
```
CREATE INDEX Query returned successfully in 548 msec.
INSERT 0 1 Query returned successfully in 282 msec.
```

```
ERROR:  Key (title)=(Unique Science Book) already exists.duplicate key value violates unique constraint t_books_selective_unique_idx
ERROR:  duplicate key value violates unique constraint t_books_selective_unique_idx
SQL state: 23505
Detail: Key (title)=(Unique Science Book) already exists.
```

```
INSERT 0 1 Query returned successfully in 309 msec.
```
    
    *Объясните результат:*
    уникальный индекс это как констрейт unique, но с допольнительным условием, тут он следит, чтобы внутри категории science все названия были уникальны, но помимо этого уникальность тайтлов его не смущает.
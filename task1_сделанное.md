# Задание 1. B-tree индексы в PostgreSQL

1. Запустите БД через docker compose в ./src/docker-compose.yml:

2. Выполните запрос для поиска книги с названием 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ```sql
   "Seq Scan on t_books  (cost=0.00..3155.00 rows=1 width=33) (actual time=25.585..27.349 rows=1 loops=1)"
    "Filter: ((title)::text = 'Oracle Core'::text)"
    "Rows Removed by Filter: 149999"
    "Planning Time: 0.211 ms"
    "Execution Time: 27.375 ms"
    ```
   
   *Объясните результат:*
   ```
   Последовательное прохождение по всем строкам таблицы заняло 27 мс, а время на планирование и анализ SQL-запроса: 0,211 мс
   ```

3. Создайте B-tree индексы:
   ```sql
   CREATE INDEX t_books_title_idx ON t_books(title);
   CREATE INDEX t_books_active_idx ON t_books(is_active);
   ```
   
   *Результат:*
   CREATE INDEX
    Query returned successfully in 786 msec.

4. Проверьте информацию о созданных индексах:
   ```sql
   SELECT schemaname, tablename, indexname, indexdef
   FROM pg_catalog.pg_indexes
   WHERE tablename = 't_books';
   ```
   
   *Результат:*
    "tikhonova"	"t_books"	"t_books_title_idx"	"CREATE INDEX t_books_title_idx ON tikhonova.t_books USING btree (title)"
    "tikhonova"	"t_books"	"t_books_active_idx"	"CREATE INDEX t_books_active_idx ON tikhonova.t_books USING btree (is_active)"
   
   *Объясните результат:*
   В моей схеме (тихонова) в таблице t_books были созданы два индекса б-деревьев на столбцы title и is_active

5. Обновите статистику таблицы:
   ```sql
   ANALYZE t_books;
   ```
   
   *Результат:*
   Query returned successfully in 396 msec.

6. Выполните запрос для поиска книги 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   "Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.032..0.033 rows=1 loops=1)"
    "Index Cond: ((title)::text = 'Oracle Core'::text)"
    "Planning Time: 0.303 ms"
    "Execution Time: 0.053 ms"
   
   *Объясните результат:*
   индекс ускорил поиск по названию более чем в 500 раз засчет использования индекса, б-деревья эффективны в поиске конкретного значеняи среди сравниваемых строк

7. Выполните запрос для поиска книги по book_id и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```
    "Index Scan using t_books_id_pk on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.032..0.034 rows=1 loops=1)"
    "  Index Cond: (book_id = 18)"
    "Planning Time: 0.134 ms"
    "Execution Time: 0.067 ms"
    ```
   
   *Объясните результат:*
   ```
   несмотря на то что вручную по book_id мы индекс не создавали, он автоматически создался при создании ограничения (констрейта) primary key, в создании таблицы (из условия), так что запрос выполнился быстро 
   ```

8. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   "Seq Scan on t_books  (cost=0.00..2780.00 rows=75075 width=33) (actual time=0.013..20.114 rows=75405 loops=1)"
    "  Filter: is_active"
    "  Rows Removed by Filter: 74595"
    "Planning Time: 0.155 ms"
    "Execution Time: 23.527 ms"
   
   *Объясните результат:*
   Несмотря на то, что на столбец is_active создан индекс, его использование не позволяет ускорить поиск по этому столбцу, т.к. значения в столбце бинарные и не позволяют их отсортировать так, чтобы использование б-дерева было эффективным. Получается, что по запросу все равно нужно пройти все значения is_active = True, т.к. большей определенности использование б-деревьев тут не позволяет добиться.

9. Посчитайте количество строк и уникальных значений:
   ```sql
   SELECT 
       COUNT(*) as total_rows,
       COUNT(DISTINCT title) as unique_titles,
       COUNT(DISTINCT category) as unique_categories,
       COUNT(DISTINCT author) as unique_authors
   FROM t_books;
   ```
   
   *Результат:*
   total rows: 150000	
   unique titles: 150000	
   unique_categories: 6	
   unique_authors:1003

10. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:*
    DROP INDEX
    Query returned successfully in 207 msec.

11. Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:
    a. `WHERE title = $1 AND category = $2`
    b. `WHERE title = $1`
    c. `WHERE category = $1 AND author = $2`
    d. `WHERE author = $1 AND book_id = $2`
    
    *Созданные индексы:*
    ```sql
    CREATE INDEX t_books_title_сat_idx ON t_books(title); 
    CREATE INDEX idx_author_category ON t_books(author, category);
    ```
    
    *Объясните ваше решение:*
    а-б создаем индекс по тайтл, т.к. он высокоселективный. создавать составной индекс по title и category для запроса (a) избыточно, т.к. индекс по title достаточно сузит радиус поиска.
    c создаем составной индекс по author и category, причем сначала ставим author, т.к. он более высокоселективен
    d Ничего не создаем, запрос достаточно оптимизирован уже существующим индексом по book_id

12. Протестируйте созданные индексы.
    
    *Результаты тестов:*
    ```
    a) "Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=1.321..1.324 rows=1 loops=1)"
    "  Index Cond: ((title)::text = 'Book_11'::text)"
    "  Filter: ((category)::text = 'History'::text)"
    "Planning Time: 0.620 ms"
    "Execution Time: 1.351 ms"

    b) "Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.054..0.056 rows=1 loops=1)"
    "  Index Cond: ((title)::text = 'Book_11'::text)"
    "Planning Time: 0.139 ms"
    "Execution Time: 0.094 ms"
    c) "Bitmap Heap Scan on t_books  (cost=4.60..110.97 rows=30 width=33) (actual time=0.060..0.164 rows=36 loops=1)"
    "  Recheck Cond: (((author)::text = 'Author_11'::text) AND ((category)::text = 'History'::text))"
    "  Heap Blocks: exact=35"
    "  ->  Bitmap Index Scan on idx_author_category  (cost=0.00..4.59 rows=30 width=0) (actual time=0.049..0.050 rows=36 loops=1)"
    "        Index Cond: (((author)::text = 'Author_11'::text) AND ((category)::text = 'History'::text))"
    "Planning Time: 0.141 ms"
    "Execution Time: 0.202 ms"
    d) "Index Scan using t_books_id_pk on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.040..0.040 rows=0 loops=1)"
    "  Index Cond: (book_id = 1295)"
    "  Filter: ((author)::text = 'Author_11'::text)"
    "  Rows Removed by Filter: 1"
    "Planning Time: 0.213 ms"
    "Execution Time: 0.071 ms"
    ```
    
    *Объясните результаты:*
    индексы ускорили запросы :), т.к. б-деревья тут эффективно работают засчет того что они ищут конкретные значения среди сравниваемых значений.

13. Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ```
    "Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=127.448..127.450 rows=0 loops=1)"
    "  Filter: ((title)::text ~~* 'Relational%'::text)"
    "  Rows Removed by Filter: 150000"
    "Planning Time: 0.596 ms"
    "Execution Time: 127.495 ms"
    ```
    
    *Объясните результат:*
    работает долго, т.к. проходит по всем строкам и смотрит на их начало

14. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    CREATE INDEX
    Query returned successfully in 649 msec.

15. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    ```
    "Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=33)
    (actual time=77.763..77.763 rows=0 loops=1)"
    "  Filter: (upper((title)::text) ~~ 'RELATIONAL%'::text)"
    "  Rows Removed by Filter: 150000"
    "Planning Time: 0.379 ms"
    "Execution Time: 77.794 ms"
    ```
    
    *Объясните результат:*
    Можно заметить, что наш индекс тут вообще не используется. Сравнение выполняется быстрее засчет того, что теперь все строки написаны заглавными буквами и используется метод LIKE а не ILIKE. плюс, нужно каждую строку привести к верхнему регистру, что занимает время, поэтому б-три индекс не может оптимально проводить поиск, т.к. присутствует функциональная часть

16. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    ```
    "Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=126.644..126.648 rows=1 loops=1)"
    "  Filter: ((title)::text ~~* '%Core%'::text)"
    "  Rows Removed by Filter: 149999"
    "Planning Time: 0.303 ms"
    "Execution Time: 126.687 ms"
    ```
    
    *Объясните результат:*
    Долго выполняется, т.к. ходит по всем строкам и используется метод ILIKE, б-три индекс не используется, т.к. не может ускорить поиск подстрок

17. Попробуйте удалить все индексы:
    ```sql
    DO $$ 
    DECLARE
        r RECORD;
    BEGIN
        FOR r IN (SELECT indexname FROM pg_indexes 
                  WHERE tablename = 't_books' 
                  AND indexname != 'books_pkey')
        LOOP
            EXECUTE 'DROP INDEX ' || r.indexname;
        END LOOP;
    END $$;
    ```
    
    *Результат:*
    ```
    DO

    Query returned successfully in 228 msec.
    ```
    
    *Объясните результат:*
    Все индексы удалены

18. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*\
    1 вариант: 
    Начало строки: 
```
    "Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=20.125..20.126 rows=0 loops=1)"
    "  Filter: ((title)::text ~~ 'Relational%'::text)"
    "  Rows Removed by Filter: 150000"
    "Planning Time: 0.138 ms"
    "Execution Time: 20.153 ms"
```
Середина строки:
```
"Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=122.204..122.208 rows=1 loops=1)"
"  Filter: ((title)::text ~~* '%Core%'::text)"
"  Rows Removed by Filter: 149999"
"Planning Time: 0.170 ms"
"Execution Time: 122.230 ms"
```
2 вариант: 
Начало строки: 
```"Bitmap Heap Scan on t_books  (cost=95.15..150.36 rows=15 width=33) (actual time=0.122..0.123 rows=0 loops=1)"
"  Recheck Cond: ((title)::text ~~ 'Relational%'::text)"
"  Rows Removed by Index Recheck: 1"
"  Heap Blocks: exact=1"
"  ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..95.15 rows=15 width=0) (actual time=0.108..0.108 rows=1 loops=1)"
"        Index Cond: ((title)::text ~~ 'Relational%'::text)"
"Planning Time: 2.432 ms"
"Execution Time: 0.247 ms"

```
Середина строки:

```
"Bitmap Heap Scan on t_books  (cost=21.57..76.78 rows=15 width=33) (actual time=0.057..0.059 rows=1 loops=1)"
"  Recheck Cond: ((title)::text ~~* '%Core%'::text)"
"  Heap Blocks: exact=1"
"  ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..21.56 rows=15 width=0) (actual time=0.028..0.028 rows=1 loops=1)"
"        Index Cond: ((title)::text ~~* '%Core%'::text)"
"Planning Time: 0.276 ms"
"Execution Time: 0.103 ms"

```

    
    *Объясните результаты:*
    Индекс с триграммой сработал сильно лучше и ускорил запрос, т.к. принцип его работы (разбиение текста на последовательности из 3 букв) позволяет быстро исключать из поиска строки, где нет требуемых послеовательностей. реверс в этом плане вообще не практичен, т.к. переворачивание строки не ускоряет сравнения всех подстрок, а в остальном этот индекс просто индекс б-дерева.

1.  Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
```"Bitmap Heap Scan on t_books  (cost=116.57..120.58 rows=1 width=33) (actual time=0.071..0.072 rows=1 loops=1)"
"  Recheck Cond: ((title)::text = 'Oracle Core'::text)"
"  Heap Blocks: exact=1"
"  ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..116.57 rows=1 width=0) (actual time=0.057..0.057 rows=1 loops=1)"
"        Index Cond: ((title)::text = 'Oracle Core'::text)"
"Planning Time: 0.288 ms"
"Execution Time: 0.114 ms"
```
    
    *Объясните результат:*
    Точное совпадение - как будто бы частный случай поиска строк с частичным совпаднием. Здесь индекс проверяет наличие в строке всех триграмм заданной строки, что совпадает с методикой поиской строк с частичным совпадением, что позволяет оптимизировать запрос


2.  Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*

```"Bitmap Heap Scan on t_books  (cost=95.15..150.36 rows=15 width=33) (actual time=0.040..0.041 rows=0 loops=1)"
"  Recheck Cond: ((title)::text ~~ 'RELATIONAL%'::text)"
"  Rows Removed by Index Recheck: 1"
"  Heap Blocks: exact=1"
"  ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..95.15 rows=15 width=0) (actual time=0.028..0.029 rows=1 loops=1)"
"        Index Cond: ((title)::text ~~ 'RELATIONAL%'::text)"
"Planning Time: 0.134 ms"
"Execution Time: 0.068 ms"
```

    
    *Объясните результат:*
    Быстро и хорошо ищет подстроку, сравнивая триграммы. все гуд
    
3.  Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
``` sql
	    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
```
    
    *План выполнения:*
```"Index Scan using t_books_desc_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.046..0.047 rows=1 loops=1)"
"  Index Cond: ((title)::text = 'Oracle Core'::text)"
"Planning Time: 0.349 ms"
"Execution Time: 0.071 ms"
```
    
    *Объясните результат:*
    такой индекс ускорил поиск в сравнении с индексом reverse, т.к. позволяет релевантно сравнивать между собой строки. Здесь он работает просто как обычный б-дерево индекс, что позволило ускорить запрос.
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
    ```
    INFO:  analyzing "public.t_books"
    INFO:  "t_books": scanned 1225 of 1225 pages, containing 150000 live rows and 3 dead rows; 30000 rows in sample, 150000 estimated total rows
    INFO:  analyzing "public.t_books_part" inheritance tree
    INFO:  "t_books_part_1": scanned 408 of 408 pages, containing 49999 live rows and 0 dead rows; 9984 rows in sample, 49999 estimated total rows
    INFO:  "t_books_part_2": scanned 409 of 409 pages, containing 50000 live rows and 0 dead rows; 10008 rows in sample, 50000 estimated total rows
    INFO:  "t_books_part_3": scanned 409 of 409 pages, containing 50001 live rows and 0 dead rows; 10008 rows in sample, 50001 estimated total rows
    INFO:  analyzing "public.t_books_part_1"
    INFO:  "t_books_part_1": scanned 408 of 408 pages, containing 49999 live rows and 0 dead rows; 30000 rows in sample, 49999 estimated total rows
    INFO:  analyzing "public.t_books_part_2"
    INFO:  "t_books_part_2": scanned 409 of 409 pages, containing 50000 live rows and 0 dead rows; 30000 rows in sample, 50000 estimated total rows
    INFO:  analyzing "public.t_books_part_3"
    INFO:  "t_books_part_3": scanned 409 of 409 pages, containing 50001 live rows and 0 dead rows; 30000 rows in sample, 50001 estimated total rows
    ANALYZE

    Query returned successfully in 507 msec.
    ```
4. Выполните запрос для поиска книги с id = 18:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```
   "Seq Scan on t_books_part_1 t_books_part  (cost=0.00..1032.99 rows=1 width=32) (actual time=0.021..8.187 rows=1 loops=1)"
   "  Filter: (book_id = 18)"
   "  Rows Removed by Filter: 49998"
   "Planning Time: 1.226 ms"
   "Execution Time: 8.308 ms"
   ```
   
   *Объясните результат:*
   Последовательное сканирование осуществлялось с помощью последовательного сканирования, но не во всей таблице, а в ее первой партиции, что позволило оптимизировать запрос.

5. Выполните поиск по названию книги:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   ```
   "Append  (cost=0.00..3101.01 rows=3 width=33) (actual time=6.504..17.035 rows=1 loops=1)"
   "->  Seq Scan on t_books_part_1  (cost=0.00..1032.99 rows=1 width=32) (actual time=6.503..6.503 rows=1 loops=1)"
   "Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "Rows Removed by Filter: 49998"
   "->  Seq Scan on t_books_part_2  (cost=0.00..1034.00 rows=1 width=33) (actual time=4.703..4.703 rows=0 loops=1)"
   "Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "Rows Removed by Filter: 50000"
   "->  Seq Scan on t_books_part_3  (cost=0.00..1034.01 rows=1 width=34) (actual time=5.819..5.819 rows=0 loops=1)"
   "Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "Rows Removed by Filter: 50001"
   "Planning Time: 0.705 ms"
   "Execution Time: 17.117 ms"
   ```
   
   *Объясните результат:*
   Видно, что оптимизатор в данном запросе соединил и прошелся по всем трем таблицам-партициям и в них искал последовательным поиском строки, подходящие по критерию. Видно, что он исключил все строки, кроме одной в первой партиции. 

6. Создайте партиционированный индекс:
   ```sql
   CREATE INDEX ON t_books_part(title);
   ```
   
   *Результат:*
   ```
   CREATE INDEX
   Query returned successfully in 225 msec.
   ``` 

7. Повторите запрос из шага 5:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   ```
   "Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.094..0.245 rows=1 loops=1)"
   "->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.094..0.095 rows=1 loops=1)"
   "Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.061..0.061 rows=0 loops=1)"
   "Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.080..0.080 rows=0 loops=1)"
   "Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "Planning Time: 1.781 ms"
   "Execution Time: 0.337 ms"
   ```
   
   *Объясните результат:*
   Оптимизатор также проходит а не по самой таблице, а по ее партициям, при этом он использует не последовательный поиск, а индекс, который был создан на основную таблицу, который впоследствии применился и для партиций, следовательно, скорость запроса после создания индекса увеличилась приблизительно в 50 раз.

8. Удалите созданный индекс:
   ```sql
   DROP INDEX t_books_part_title_idx;
   ```
   
   *Результат:*
   ```
   DROP INDEX
   
   Query returned successfully in 95 msec.
   ```

9. Создайте индекс для каждой партиции:
   ```sql
   CREATE INDEX ON t_books_part_1(title);
   CREATE INDEX ON t_books_part_2(title);
   CREATE INDEX ON t_books_part_3(title);
   ```
   
   *Результат:*
   ```
   CREATE INDEX

   Query returned successfully in 244 msec.
   ```

10. Повторите запрос из шага 5:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part 
    WHERE title = 'Expert PostgreSQL Architecture';
    ```
    
    *План выполнения:*
    ```
    "Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.162..0.378 rows=1 loops=1)"
    "->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.162..0.163 rows=1 loops=1)"
    "Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.087..0.087 rows=0 loops=1)"
    "Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.118..0.118 rows=0 loops=1)"
    "Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "Planning Time: 2.228 ms"
    "Execution Time: 0.491 ms"
    ```

    *Объясните результат:*
    Аналогично тому, что было в пункте 7.

11. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_part_1_title_idx;
    DROP INDEX t_books_part_2_title_idx;
    DROP INDEX t_books_part_3_title_idx;
    ```
    
    *Результат:*
    ```
    DROP INDEX

    Query returned successfully in 116 msec.
    ```

12. Создайте обычный индекс по book_id:
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id);
    ```
    
    *Результат:*
    ```
    CREATE INDEX

    Query returned successfully in 81 msec.
    ```

13. Выполните поиск по book_id:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part WHERE book_id = 11011;
    ```
    
    *План выполнения:*
    ```
    "Index Scan using t_books_part_1_book_id_idx on t_books_part_1 t_books_part  (cost=0.29..8.31 rows=1 width=32) (actual time=0.178..0.187 rows=1 loops=1)"
    "Index Cond: (book_id = 11011)"
    "Planning Time: 1.985 ms"
    "Execution Time: 0.281 ms"
    ```
    
    *Объясните результат:*
    Для начала оптимизатор замечает, что id находится в таблице-партиции под номером 1, следовательно, можно отсечь 2 остальные таблички, и далее фильтровать по 1 партиции. Таким образом, внутри данной таблицы оптимизатор проведет поиск по индексу.

14. Создайте индекс по полю is_active:
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active);
    ```
    
    *Результат:*
    ```
    CREATE INDEX

    Query returned successfully in 111 msec.
    ```
15. Выполните поиск активных книг с отключенным последовательным сканированием:
    ```sql
    SET enable_seqscan = off;
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE is_active = true;
    SET enable_seqscan = on;
    ```
    
    *План выполнения:*
    ```
    "Bitmap Heap Scan on t_books  (cost=840.69..2814.59 rows=74890 width=33) (actual time=2.859..11.833 rows=74839 loops=1)"
    "Recheck Cond: is_active"
    "Heap Blocks: exact=1225"
    "->  Bitmap Index Scan on t_books_active_idx  (cost=0.00..821.97 rows=74890 width=0) (actual time=2.662..2.662 rows=74839 loops=1)"
    "Index Cond: (is_active = true)"
    "Planning Time: 0.154 ms"
    "Execution Time: 14.468 ms"
    ```
    
    *Объясните результат:*
    Сначала оптимизиатор создает карты битов, на основе которой легче осуществлять сканирование таблицы. Далее, по этим битам с помощью индекса происходит сканирование.

16. Создайте составной индекс:
    ```sql
    CREATE INDEX t_books_author_title_index ON t_books(author, title);
    ```
    
    *Результат:*
    ```
    CREATE INDEX

    Query returned successfully in 303 msec.
    ```

17. Найдите максимальное название для каждого автора:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, MAX(title) 
    FROM t_books 
    GROUP BY author;
    ```
    
    *План выполнения:*
    ```
    "HashAggregate  (cost=3475.00..3485.01 rows=1001 width=42) (actual time=57.869..57.953 rows=1003 loops=1)"
    "Group Key: author"
    "Batches: 1  Memory Usage: 193kB"
    "->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=21) (actual time=0.019..10.242 rows=150000 loops=1)"
    "Planning Time: 0.123 ms"
    "Execution Time: 58.018 ms"
    ```
    
    *Объясните результат:*
    В данном случае оптимизатор решил, что не использовать индекс будет оптимальнее. Сортировка происходит по столбцу title, поэтому индекс, использующий и title, и author оказался неэффективен. Поэтому оптимизатор использовал здесь HashAggregate - метод агрегации, который использует для этого хэш-таблицы, а далее он выполнил последовательный поиск.

18. Выберите первых 10 авторов:
    ```sql
    EXPLAIN ANALYZE
    SELECT DISTINCT author 
    FROM t_books 
    ORDER BY author 
    LIMIT 10;
    ```
    
    *План выполнения:*
    ```
    "Limit  (cost=0.42..56.61 rows=10 width=10) (actual time=0.029..0.385 rows=10 loops=1)"
    "->  Result  (cost=0.42..5625.42 rows=1001 width=10) (actual time=0.028..0.383 rows=10 loops=1)"
    "->  Unique  (cost=0.42..5625.42 rows=1001 width=10) (actual time=0.027..0.381 rows=10 loops=1)"
    "->  Index Only Scan using t_books_author_title_index on t_books  (cost=0.42..5250.42 rows=150000 width=10) (actual time=0.026..0.232 rows=1383 loops=1)"
    "Heap Fetches: 5"
    "Planning Time: 0.106 ms"
    "Execution Time: 0.404 ms"
    ```
    
    *Объясните результат:*
    Вначале задается ограничение на количество строк. Далее, в результате задется ограничение уникальности и проводится поиск с помощью ранее созданного индекса. В ходе фильтрации пришлось найти 5 значений не с помощью индекса. Скорость запроса - средняя.

19. Выполните поиск и сортировку:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE author LIKE 'T%'
    ORDER BY author, title;
    ```
    
    *План выполнения:*
    ```
    "Sort  (cost=3100.29..3100.33 rows=15 width=21) (actual time=35.754..35.762 rows=1 loops=1)"
    "Sort Key: author, title"
    "Sort Method: quicksort  Memory: 25kB"
    "->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=21) (actual time=35.686..35.694 rows=1 loops=1)"
    "Filter: ((author)::text ~~ 'T%'::text)"
    "Rows Removed by Filter: 149999"
    "Planning Time: 1.870 ms"
    "Execution Time: 35.861 ms"
    ```
    
    *Объясните результат:*
    Вначале оптимизатор прибегает к сортировке с помощью алгоритма "Quicksort", который заключается в определении опорного элемента (середины), от которого отталкивается дальнейшее решение. К сортированным данным применяется проверка по фильтру, в результате которой убираются ненужные строки.

20. Добавьте новую книгу:
    ```sql
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true);
    COMMIT;
    ```
    
    *Результат:*
    ```
    WARNING:  there is no transaction in progress
    COMMIT

    Query returned successfully in 92 msec.
    ```

21. Создайте индекс по категории:
    ```sql
    CREATE INDEX t_books_cat_idx ON t_books(category);
    ```
    
    *Результат:*
    ```
    CREATE INDEX

    Query returned successfully in 168 msec.
    ```

22. Найдите книги без категории:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```
    "Index Scan using t_books_cat_idx on t_books  (cost=0.29..8.17 rows=1 width=21) (actual time=0.062..0.070 rows=1 loops=1)"
    "Index Cond: (category IS NULL)"
    "Planning Time: 0.684 ms"
    "Execution Time: 0.129 ms"
    ```
    
    *Объясните результат:*
    Созданный индекс используется в сортировке оптимизатором при запросе, который включает в себя фильтрацию по категории. Таким образом, запрос становится значительно быстрее.

23. Создайте частичные индексы:
    ```sql
    DROP INDEX t_books_cat_idx;
    CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL;
    ```
    
    *Результат:*
    ```
    CREATE INDEX

    Query returned successfully in 102 msec.
    ```

24. Повторите запрос из шага 22:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```
    "Index Scan using t_books_cat_null_idx on t_books  (cost=0.12..8.00 rows=1 width=21) (actual time=0.038..0.040 rows=1 loops=1)"
    "Planning Time: 0.636 ms"
    "Execution Time: 0.087 ms"
    ```
    
    *Объясните результат:*
    Данный запрос использует частичный индекс, и он значительно ускоряет запрос, так как в запросе нам требуется вывести те строки, в которых категория не задана, а данный индекс помогает повысить эффективность сортировки по конкретному значению категории (NULL).

25. Создайте частичный уникальный индекс:
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
    INSERT 0 1

    Query returned successfully in 86 msec.
    --------

    ERROR:  Key (title)=(Unique Science Book) already exists.duplicate key value violates unique constraint "t_books_selective_unique_idx" 

    ERROR:  duplicate key value violates unique constraint "t_books_selective_unique_idx"
    SQL state: 23505
    Detail: Key (title)=(Unique Science Book) already exists.
    --------

    INSERT 0 1

    Query returned successfully in 73 msec.
    ```


    
    *Объясните результат:*
    Так как создание индекса вводит новое ограничение на уникальность, то вводится запрет на создание строк с категориями, которые уже существуют.
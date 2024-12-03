# Задание 1. B-tree индексы в PostgreSQL

1. Запустите БД через docker compose в ./src/docker-compose.yml:

2. Выполните запрос для поиска книги с названием 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ```
   "Seq Scan on t_books  (cost=0.00..3099.00 rows=1 width=33) (actual time=32.224..32.225 rows=1 loops=1)"
   "Filter: ((title)::text = 'Oracle Core'::text)"
   "Rows Removed by Filter: 149999"
   "Planning Time: 0.601 ms"
   "Execution Time: 32.294 ms"
   ```

   *Объясните результат:*
   В данном запросе используется последовательный поиск, так как индексы не были созданы. Можно заметить, что время исполнения очень долгое по сравнению с использованием индекса, следовательно, это неэффективно.

3. Создайте B-tree индексы:
   ```sql
   CREATE INDEX t_books_title_idx ON t_books(title);
   CREATE INDEX t_books_active_idx ON t_books(is_active);
   ```
   
   *Результат:*
   ```
   CREATE INDEX

   Query returned successfully in 382 msec.
   ```
4. Проверьте информацию о созданных индексах:
   ```sql
   SELECT schemaname, tablename, indexname, indexdef
   FROM pg_catalog.pg_indexes
   WHERE tablename = 't_books';
   ```
   
   *Результат:*
   ```
   "public"	"t_books"	"t_books_id_pk"	"CREATE UNIQUE INDEX t_books_id_pk ON public.t_books USING btree (book_id)"
   "public"	"t_books"	"t_books_title_idx"	"CREATE INDEX t_books_title_idx ON public.t_books USING btree (title)"
   "public"	"t_books"	"t_books_active_idx"	"CREATE INDEX t_books_active_idx ON public.t_books USING btree (is_active)"
   ```
   *Объясните результат:*
   Можно заметить, что был создан индекс b-tree.
   В запросе не было указано, что нужно создать конкретный индекс, но оптимизатор решил, что создание данного яндекса является оптимальным. Его создание значительно ускоряет скорость запросов, в которых используются столбцы, включенные в индекс.

5. Обновите статистику таблицы:
   ```sql
   ANALYZE t_books;
   ```
   
   *Результат:*
   ```
   INFO:  analyzing "public.t_books"
   INFO:  "t_books": scanned 1224 of 1224 pages, containing 150000 live rows and 3 dead rows;    30000 rows in sample, 150000 estimated total rows
   ANALYZE

   Query returned successfully in 212 msec.
   ```
   

6. Выполните запрос для поиска книги 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ```
   "Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.179..0.181 rows=1 loops=1)"
   "Index Cond: ((title)::text = 'Oracle Core'::text)"
   "Planning Time: 1.378 ms"
   "Execution Time: 0.308 ms"
   ```
   
   *Объясните результат:*
   Можно заметить, что при использовании индекса b-tree вместо sequantial scan скорость запроса увеличилась примерно в 104.8 раза. Таким образом, использование яндекса при запросе существенно уменьшает время запроса.

7. Выполните запрос для поиска книги по book_id и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```
   "Index Scan using t_books_id_pk on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.261..0.263 rows=1 loops=1)"
   "Index Cond: (book_id = 18)"
   "Planning Time: 0.795 ms"
   "Execution Time: 0.440 ms"
   ```
   
   *Объясните результат:*
   Так как book_id является PK, для него автоматически создается индекс b-tree, благодаря которому скорость запроса значительно увеличивается. Таким образом, использование сортировки по первичному ключу даже без ручного создания индекса является довольно быстрой операцией.

8. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   ```
   "Seq Scan on t_books  (cost=0.00..2724.00 rows=75270 width=33) (actual time=0.049..24.351 rows=74808 loops=1)"
   "Filter: is_active"
   "Rows Removed by Filter: 75192"
   "Planning Time: 0.308 ms"
   "Execution Time: 27.960 ms"
   ```

   
   *Объясните результат:*
   Для данного запроса не использовался созданный индекс, так как, если атрибут имеет низкую селективность, то есть на выбор оптимизатору предстает только 2 значения (true & false), то он для данного запроса выберет seqscan вместо использования индекса, так как он посчитал это более эффективным. Однако стоит обратить внимание на то, что планируемое время запроса было существенно меньше, чем итоговое.

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
   ```
   150000	150000	6	1003
   ```

10. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:*
    ```
    DROP INDEX

    Query returned successfully in 100 msec.
    ```

11. Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:
    a. `WHERE title = $1 AND category = $2`
    b. `WHERE title = $1`
    c. `WHERE category = $1 AND author = $2`
    d. `WHERE author = $1 AND book_id = $2`
    
    *Созданные индексы:*
    ### a:
    ```sql 
    CREATE INDEX title_category_idx ON t_books(title, category);
    ```
    ### b:
    ```sql
    CREATE INDEX hash_title_idx ON t_books USING HASH(title);
    ```
    ### c:
    ```sql
    CREATE INDEX author_category_idx ON t_books(author, category);
    ```
    ### d:
    ```sql
    CREATE INDEX id_author_idx ON t_books(book_id, author);
    ```
    
    *Объясните ваше решение:*
    В пунктах "a", "c" и "d" я принял решение создать составные индексы, так как, по результатам проведенных мной тестов, они являются одними из самых быстрых в плане времени выполнения запросов. На первое место в данных индексах я ставил атрибут с наибольшей селективностью (то есть, значения более уникальны, чем у другого атрибута), так как, благодаря этому, сортировка происходит эффективнее. В пункте "b" я создал hash-индекс, так как он является одним из самых эффективных индексов для одномерных данных в случае поиска по равенству. Данный индекс создает отдельно таблицу с хэшами с помощью хэш-функций, к которой впоследствии обращаются запросы с использованием индекса.

12. Протестируйте созданные индексы.
    
    *Результаты тестов:*
    ### a:
    ```
    "Index Scan using title_category_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.040..0.042 rows=1 loops=1)"
    "Index Cond: (((title)::text = 'Book_9'::text) AND ((category)::text = 'History'::text))"
    "Planning Time: 0.120 ms"
    "Execution Time: 0.063 ms"
    ```
    ### b:
    ```
    "Index Scan using hash_title_idx on t_books  (cost=0.00..8.02 rows=1 width=33) (actual time=0.023..0.024 rows=1 loops=1)"
    "Index Cond: ((title)::text = 'Book_9'::text)"
    "Planning Time: 0.110 ms"
    "Execution Time: 0.045 ms"
    ```
    ### c:
    ```
    "Bitmap Heap Scan on t_books  (cost=4.60..110.96 rows=30 width=33) (actual time=0.040..0.094 rows=33 loops=1)"
    "Recheck Cond: (((author)::text = 'Author_493'::text) AND ((category)::text = 'History'::text))"
    "Heap Blocks: exact=32"
    "->  Bitmap Index Scan on author_category_idx  (cost=0.00..4.59 rows=30 width=0) (actual time=0.029..0.029 rows=33 loops=1)"
    "Index Cond: (((author)::text = 'Author_493'::text) AND ((category)::text = 'History'::text))"
    "Planning Time: 0.117 ms"
    "Execution Time: 0.119 ms"
    ```
    ### d:
    ```
    "Index Scan using id_author_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.031..0.033 rows=1 loops=1)"
    "Index Cond: ((book_id = 23) AND ((author)::text = 'Author_946'::text))"
    "Planning Time: 0.138 ms"
    "Execution Time: 0.055 ms"
    ```
    
    *Объясните результаты:*
    Как я заметил на основе запросов по поиску данных без индекса и с индексом, его использование ускоряет скорость запроса в 50-100 раз, а иногда и больше. Таким образом, в пункте *a* использовался b-tree составной индекс по поиске значений по категории и названию. Первым в индексе использовалось название, так как оно более селективное. 
    В пункте *b* мною был создан хэш-индекс, который ускоряет время запроса в связи с тем, что он позволяет быстро обрабатывать запросы с определением равенства. В запросах сравнения он был бы менее эффективным.
    В пункте *c* также использовался b-tree индекс, который использовал вспомогательные битмап сканирование и битмап индексное сканирование. Данные в данном случае представляются в виде битмапа, где каждый бит сооответствует определенному блоку ячеек в таблице. Такое сканирование наиболее эффективно, когда запрос фильтруется сразу по нескольким условиям
    В пункте *d*, аналогично пункту *a*, используется b-tree составной индекс, который значительно ускоряет время выполнения запроса.

13. Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ```
    "Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=33) (actual time=66.838..66.839 rows=0 loops=1)"
    "Filter: ((title)::text ~~* 'Relational%'::text)"
    "Rows Removed by Filter: 150000"
    "Planning Time: 1.919 ms"
    "Execution Time: 66.980 ms"
    ```

    
    *Объясните результат:*
    В данном запросе используется поиск по названию, при этом используется не равенство в фильтрации, а оператор *ILIKE*, который нечувствителен к регистру. Таким образом, по нему можно найти названия, которые содержат в начале буквы "Rational", "rational", "RATIONAL" и так далее. Так как в данном запросе используется Seq Scan, то он является довольно долгим (67 мс).

14. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    ```
    CREATE INDEX

    Query returned successfully in 320 msec.
    ```

15. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    ```
    "Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=33) (actual time=48.518..48.519 rows=0 loops=1)"
    "Filter: (upper((title)::text) ~~ 'RELATIONAL%'::text)"
    "Rows Removed by Filter: 150000"
    "Planning Time: 0.108 ms"
    "Execution Time: 48.540 ms"
    ```
    
    *Объясните результат:*
    Функциональный индекс не является эффективным, так как для каждого значения title приходится выполнять операцию перехода в верхний регистр, помимо этого, выполняется Seq Scan, из-за которого использование данного индекса не является целесообразным в данном случае, так как время выполнение запроса с индексом и без почти одинаковое от случая к случаю.

16. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    ```
    "Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=33) (actual time=61.740..61.743 rows=1 loops=1)"
    "Filter: ((title)::text ~~* '%Core%'::text)"
    "Rows Removed by Filter: 149999"
    "Planning Time: 0.956 ms"
    "Execution Time: 61.895 ms"
    ```
    
    *Объясните результат:*
    Запрос представляет собой поиск строки атрибута title, в которой *Core* является внутренней частью строки. Можно заметить, что без создания индекса, по умолчанию, оптимизатор использует последовательный перебор всех значений, таким образом, скорость запроса неоптимизирована и составляет почти 62 мс, что является довольно большим временем для запроса.

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

    Query returned successfully in 69 msec.
    ```
    
    *Объясните результат:*
    C помощью данного запроса я удалил все индексы кроме индекса первичного ключа при помощи вспомогательного оператора *DO*.

18. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*
    Вариант 1:
    ```
    CREATE INDEX

    Query returned successfully in 250 msec.
    ```
    Вариант 2:
    ```
    CREATE INDEX

    Query returned successfully in 416 msec.
    ```

    *Объясните результаты:*
    В первом варианте я создал индекс который оптимизирует поиск по названию книги, записанному наоборот. Сделано это для того, чтобы было эффективнее фильтровать данные по концу названия. Во втором же случае проверяется, существует ли расширение с триграммами, и если оно не существует, то расширение создается. Далее создается индекс gin, который работает с триплетами, осуществляя поиск по ним. Индекс gin нужен для работы с массивами, так как одно слово (название) будет разбиваться на триграммы (слова, состоящие из 3 символов), которые должны, в теории, помочь в оптимизации процесса фильтрации. То есть, далее поиск будет осуществляться не по словам, а по их триграммам, которые прикреплены к соответствующей строке.
    

19. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
    ```
    "Bitmap Heap Scan on t_books  (cost=116.57..120.58 rows=1 width=33) (actual time=0.037..0.038 rows=1 loops=1)"
    "Recheck Cond: ((title)::text = 'Oracle Core'::text)"
    "Heap Blocks: exact=1"
    "->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..116.57 rows=1 width=0) (actual time=0.028..0.028 rows=1 loops=1)"
    "Index Cond: ((title)::text = 'Oracle Core'::text)"
    "Planning Time: 0.099 ms"
    "Execution Time: 0.058 ms"
    ```
    
    *Объясните результат:*
    Оптимизатор посчитал, что использование индекса напрямую будет менее эффективным, чем использование битмап сканирования. Можно заметить, что скорость запроса очень быстрая, что свидетельствует об эффективности использования индекса в данном ключе. Битмап осуществляет сканирование только тех строк, которые являются потенциально подходящими по критерию, поэтому его использование достаточно эффективно. Recheck Cond говоит о том, что после чтения блоков будет повторная проверка *text = 'Oracle Core'*.
    

20. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ```
    "Bitmap Heap Scan on t_books  (cost=95.15..150.36 rows=15 width=33) (actual time=0.178..0.179 rows=0 loops=1)"
    "Recheck Cond: ((title)::text ~~* 'Relational%'::text)"
    "Rows Removed by Index Recheck: 1"
    "Heap Blocks: exact=1"
    "->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..95.15 rows=15 width=0) (actual time=0.120..0.120 rows=1 loops=1)"
    "Index Cond: ((title)::text ~~* 'Relational%'::text)"
    "Planning Time: 0.984 ms"
    "Execution Time: 0.370 ms"
    ```
    
    *Объясните результат:*
    При использовании данного индекса оптимизатор также посчитал, что здесь нужно использовать битмап сканирование, в результате которого данные представляются в виде битмапа, в котором каждый бит соответствует определенному блоку данных в таблице. Можно заметить, что в результате перепроверки была убрана одна строка.
    

21. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books
    ORDER BY title DESC;
    ```
    
    *План выполнения:*

    Cоздание индекса:
    ```
    CREATE INDEX

    Query returned successfully in 337 msec.
    ```
    Выполнение запроса:
    ```
    "Index Scan using t_books_desc_idx on t_books  (cost=0.42..9065.55 rows=150000 width=33) (actual time=0.049..26.369 rows=150000 loops=1)"
    "Planning Time: 0.418 ms"
    "Execution Time: 31.320 ms"
    ```

    
    *Объясните результат:*
    Таким образом, здесь будет использоваться индекс b-tree, который значительно ускоряет время запроса (примерно в 10 раз), позволяя сэкономить время на сортировке по убыванию с помощью b-дерева.
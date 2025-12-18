## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

     [2025-12-18 17:13:39] workshop.public> ANALYZE t_books
[2025-12-18 17:13:39] completed in 98 ms

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

     [2025-12-18 17:14:16] workshop.public> CREATE INDEX t_books_fts_idx ON t_books
                                           USING GIN (to_tsvector('english', title))
[2025-12-18 17:14:17] completed in 377 ms

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.054..0.055 rows=1 loops=1)
"  Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
  Heap Blocks: exact=1
  ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.043..0.043 rows=1 loops=1)
"        Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
Planning Time: 0.567 ms
Execution Time: 0.095 ms

    
    *Объясните результат:*
    Используется GIN-индекс t_books_fts_idx по выражению to_tsvector('english', title), поэтому поиск по полнотекстовому условию выполняется быстро.

     Сначала идёт Bitmap Index Scan: индекс возвращает кандидатов.

     Затем Bitmap Heap Scan читает только нужные блоки таблицы, а Recheck Cond выполняет перепроверку условия на строках.

     Итоговое время очень маленькое, потому что таблица не сканируется целиком.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

     [2025-12-18 17:26:10] workshop.public> DROP INDEX t_books_fts_idx
[2025-12-18 17:26:10] completed in 8 ms

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

     [2025-12-18 17:28:23] workshop.public> ALTER TABLE t_lookup
                                           ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key)
[2025-12-18 17:28:23] completed in 7 ms

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

     [2025-12-18 17:42:05] workshop.public> INSERT INTO t_lookup
                                       SELECT
                                           LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
                                           'Value_' || generate_series(1, 150000)
[2025-12-18 17:42:05] 150,000 rows affected in 270 ms

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

     [2025-12-18 17:43:28] workshop.public> CREATE TABLE t_lookup_clustered (
                                                                           item_key VARCHAR(10) PRIMARY KEY,
                                                                           item_value VARCHAR(100)
                                       )
[2025-12-18 17:43:28] completed in 12 ms

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

     [2025-12-18 17:43:49] workshop.public> INSERT INTO t_lookup_clustered
                                       SELECT * FROM t_lookup
[2025-12-18 17:43:49] 150,000 rows affected in 267 ms
[2025-12-18 17:43:49] workshop.public> CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey
[2025-12-18 17:43:49] completed in 88 ms

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

     [2025-12-18 17:44:08] workshop.public> ANALYZE t_lookup
[2025-12-18 17:44:08] completed in 52 ms
[2025-12-18 17:44:08] workshop.public> ANALYZE t_lookup_clustered
[2025-12-18 17:44:08] completed in 46 ms

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.055..0.056 rows=1 loops=1)
  Index Cond: ((item_key)::text = '0000000455'::text)
Planning Time: 0.356 ms
Execution Time: 0.088 ms

     
     *Объясните результат:*
     Используется Index Scan по первичному ключу t_lookup_pk, так как условие это точное равенство по уникальному ключу.

     Поиск селективный, поэтому индекс даёт быстрый доступ без сканирования таблицы.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.136..0.137 rows=1 loops=1)
  Index Cond: ((item_key)::text = '0000000455'::text)
Planning Time: 1.048 ms
Execution Time: 0.231 ms

     
     *Объясните результат:*
     Так же, как и в обычной таблице, используется Index Scan по первичному ключу, потому что поиск идёт по уникальному значению.

     Кластеризация физически упорядочивает строки по индексу, но для точечного поиска одной строки выигрыш обычно небольшой.

     Поэтому отличие во времени здесь не принципиально и может меняться от запуска к запуску.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     [Вставьте план выполнения]
     
     *Объясните результат:*
     [Ваше объяснение]

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     [Вставьте план выполнения]
     
     *Объясните результат:*
     [Ваше объяснение]

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     [Ваше сравнение]
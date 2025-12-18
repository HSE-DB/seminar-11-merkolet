## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

    [2025-12-18 18:43:18] workshop.public> CREATE TABLE test_cluster AS
                                       SELECT
                                           generate_series(1,1000000) as id,
                                           CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
                                           md5(random()::text) as data
[2025-12-18 18:43:19] 1,000,000 rows affected in 1 s 274 ms

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

    [2025-12-18 18:43:36] workshop.public> CREATE INDEX test_cluster_cat_idx ON test_cluster(category)
[2025-12-18 18:43:36] completed in 302 ms

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    Bitmap Heap Scan on test_cluster  (cost=5550.89..20158.23 rows=497867 width=39) (actual time=15.796..92.061 rows=500852 loops=1)
  Recheck Cond: (category = 'A'::text)
  Heap Blocks: exact=8334
  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5426.43 rows=497867 width=0) (actual time=14.595..14.595 rows=500852 loops=1)
        Index Cond: (category = 'A'::text)
Planning Time: 0.805 ms
Execution Time: 105.788 ms

    
    *Объясните результат:*
    Условие category = A выбирает примерно половину таблицы, поэтому обычный Index Scan был бы слишком дорогим — Postgres использует связку Bitmap Index Scan -> Bitmap Heap Scan.

    Bitmap Index Scan получает множество совпадений из индекса test_cluster_cat_idx.

    Bitmap Heap Scan затем читает нужные блоки таблицы пачками. До кластеризации строки категории A разбросаны по таблице, из-за чего приходится читать много разных страниц, это увеличивает время.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    [2025-12-18 18:46:59] workshop.public> CLUSTER test_cluster USING test_cluster_cat_idx
[2025-12-18 18:46:59] completed in 517 ms

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    Bitmap Heap Scan on test_cluster  (cost=5550.89..20108.23 rows=497867 width=39) (actual time=11.189..61.241 rows=500852 loops=1)
  Recheck Cond: (category = 'A'::text)
  Heap Blocks: exact=4174
  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5426.43 rows=497867 width=0) (actual time=10.687..10.688 rows=500852 loops=1)
        Index Cond: (category = 'A'::text)
Planning Time: 0.573 ms
Execution Time: 73.673 ms

    
    *Объясните результат:*
    План остался тем же, потому что выборка всё ещё очень большая, и bitmap остаётся оптимальной стратегией.

    Но после CLUSTER строки категории A стали лежать более компактно, поэтому таблице нужно прочитать меньше страниц:

    было Heap Blocks: exact=8334

    стало Heap Blocks: exact=4174

    За счёт уменьшения числа читаемых блоков снизилось время выполнения.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    До cluster: Execution Time = 105.788 ms, Heap Blocks = 8334

    После cluster: Execution Time = 73.673 ms, Heap Blocks = 4174

    Вывод: кластеризация по category улучшила локальность данных и уменьшила число читаемых блоков примерно в 2 раза, что дало ускорение запроса с ~106 ms до ~74 ms.
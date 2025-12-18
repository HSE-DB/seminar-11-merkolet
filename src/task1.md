# Задание 1: BRIN индексы и bitmap-сканирование

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

   [2025-12-18 16:40:37] workshop> ANALYZE t_books
[2025-12-18 16:40:37] completed in 88 ms

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

   [2025-12-18 16:41:14] workshop.public> CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category)
[2025-12-18 16:41:14] completed in 21 ms

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.015..0.016 rows=0 loops=1)
  Recheck Cond: (category IS NULL)
  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.011..0.012 rows=0 loops=1)
        Index Cond: (category IS NULL)
Planning Time: 0.501 ms
Execution Time: 0.038 ms

   
   *Объясните результат:*
   Postgres использует BRIN индекс t_books_brin_cat_idx через Bitmap Index Scan: BRIN быстро определяет ranges, где может встречаться category IS NULL.

   Затем выполняется Bitmap Heap Scan: по битовой карте читаются только кандидаты-страницы из таблицы.

   Recheck Cond означает, что условие проверяется повторно на строках таблицы.

   rows=0 означает, что строк с NULL в category сейчас нет/

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

   [2025-12-18 16:45:15] workshop.public> CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author)
[2025-12-18 16:45:15] completed in 27 ms

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=11.472..11.472 rows=0 loops=1)
  Recheck Cond: ((category)::text = 'INDEX'::text)
  Rows Removed by Index Recheck: 150000
  Filter: ((author)::text = 'SYSTEM'::text)
  Heap Blocks: lossy=1224
  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.150..0.150 rows=12240 loops=1)
        Index Cond: ((category)::text = 'INDEX'::text)
Planning Time: 0.481 ms
Execution Time: 11.507 ms

   
   *Объясните результат (обратите внимание на bitmap scan):*
   Для условия по category = 'INDEX' используется BRIN индекс (t_books_brin_cat_idx) через Bitmap Index Scan. BRIN не хранит точные позиции строк, он хранит “сводку” по диапазонам страниц, поэтому возвращает набор подходящих page ranges.

   Далее Postgres выполняет Bitmap Heap Scan: читает подходящие страницы таблицы и делает перепроверку. Это важно, потому что BRIN даёт кандидатные страницы, а не гарантированные строки.

   Heap Blocks: lossy=1224 означает lossy bitmap: в битовой карте отмечались целые страницы, а не точные строки. Поэтому перепроверка отбрасывает огромное число строк.

   Условие по author = 'SYSTEM' применилось как обычный Filter, потому что план не использовал BRIN-индекс по author.

   rows=0 означает, что строк с одновременно category='INDEX' и author='SYSTEM' нет.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   Sort  (cost=3099.11..3099.12 rows=5 width=7) (actual time=28.739..28.741 rows=6 loops=1)
  Sort Key: category
  Sort Method: quicksort  Memory: 25kB
  ->  HashAggregate  (cost=3099.00..3099.05 rows=5 width=7) (actual time=28.706..28.707 rows=6 loops=1)
        Group Key: category
        Batches: 1  Memory Usage: 24kB
        ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=7) (actual time=0.019..9.617 rows=150000 loops=1)
Planning Time: 0.425 ms
Execution Time: 28.912 ms

   
   *Объясните результат:*
   Чтобы получить DISTINCT category, Postgreы использует HashAggregate (хеш-группировка по category), но для этого всё равно нужно прочитать всю таблицу → поэтому Seq Scan на 150000 строк.

   Затем выполняется Sort. Уникальных категорий мало, поэтому сортировка быстрая и занимает мало памяти.

   BRIN индекс по category здесь не помогает, так как задача — перечислить все уникальные значения по всей таблице, и дешевле просто один раз последовательно прочитать таблицу.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   Aggregate  (cost=3099.04..3099.05 rows=1 width=8) (actual time=11.230..11.231 rows=1 loops=1)
  ->  Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=0) (actual time=11.226..11.226 rows=0 loops=1)
        Filter: ((author)::text ~~ 'S%'::text)
        Rows Removed by Filter: 150000
Planning Time: 0.484 ms
Execution Time: 11.277 ms

   
   *Объясните результат:*
   PostgreSQL выполняет Seq Scan, потому что условие author LIKE 'S%' не является селективным по данным.

   BRIN-индекс по author здесь не используется, так как для шаблонов LIKE и при отсутствии подходящих значений дешевле один последовательный проход по таблице.

   Узел Aggregate просто подсчитывает количество строк, прошедших фильтр.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

   [2025-12-18 16:53:33] workshop.public> CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title))
[2025-12-18 16:53:33] completed in 144 ms

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   Aggregate  (cost=3475.88..3475.89 rows=1 width=8) (actual time=29.869..29.870 rows=1 loops=1)
  ->  Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=0) (actual time=29.862..29.864 rows=1 loops=1)
        Filter: (lower((title)::text) ~~ 'o%'::text)
        Rows Removed by Filter: 149999
Planning Time: 0.327 ms
Execution Time: 29.899 ms

   
   *Объясните результат:*
   Несмотря на наличие функционального индекса LOWER(title), планировщик выбрал Seq Scan.

   Причина — запрос возвращает всего 1 строку, но COUNT(*) требует пройти по всем строкам, если оптимизатор не использует индексный путь как более дешёвый.

   Узел Aggregate просто суммирует количество строк, прошедших фильтр.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

   [2025-12-18 16:58:44] workshop.public> DROP INDEX t_books_brin_cat_idx
[2025-12-18 16:58:44] completed in 10 ms
[2025-12-18 16:58:44] workshop.public> DROP INDEX t_books_brin_author_idx
[2025-12-18 16:58:44] completed in 2 ms
[2025-12-18 16:58:44] workshop.public> DROP INDEX t_books_lower_title_idx
[2025-12-18 16:58:44] completed in 5 ms

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

   [2025-12-18 16:59:08] workshop.public> CREATE INDEX t_books_brin_cat_auth_idx ON t_books
                                           USING brin(category, author)
[2025-12-18 16:59:08] completed in 36 ms

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=1.706..1.707 rows=0 loops=1)
  Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
  Rows Removed by Index Recheck: 8814
  Heap Blocks: lossy=72
  ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.033..0.033 rows=720 loops=1)
        Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
Planning Time: 0.370 ms
Execution Time: 1.757 ms
   
   *Объясните результат:*
   Используется составной BRIN индекс t_books_brin_cat_auth_idx (category, author) через Bitmap Index Scan, причём теперь Index Cond включает сразу оба условия.

   В отличие от варианта с одним BRIN по category, индекс сразу сильнее сужает набор кандидатов.

   Bitmap Heap Scan остаётся, потому что BRIN работает на уровне диапазонов страниц и требует Recheck Cond на реальных строках.

   Итоговое время стало лучше, потому что составной BRIN точнее отсекает нерелевантные диапазоны, и таблица читается значительно меньше.
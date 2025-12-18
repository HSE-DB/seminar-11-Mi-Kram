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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.018..0.019 rows=0 loops=1)
     Recheck Cond: (category IS NULL)
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.010..0.010 rows=0 loops=1)
           Index Cond: (category IS NULL)
   Planning Time: 0.306 ms
   Execution Time: 0.056 ms
   ```
   
   *Объясните результат:*
   ```
   План использует Bitmap Index Scan по BRIN-индексу, а затем Bitmap Heap Scan.
   В итоге строк с NULL не нашлось (rows=0), поэтому чтение было минимальным.
   ```

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.16..2366.21 rows=1 width=33) (actual time=13.335..13.336 rows=0 loops=1)
     Recheck Cond: ((category)::text = 'INDEX'::text)
     Rows Removed by Index Recheck: 150000
     Filter: ((author)::text = 'SYSTEM'::text)
     Heap Blocks: lossy=1225
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=75270 width=0) (actual time=0.087..0.088 rows=12250 loops=1)
           Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.193 ms
   Execution Time: 13.377 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   ```
   Используется только BRIN по category, поэтому Bitmap Index Scan отмечает много блоков как "возможные".
   
   Bitmap Heap Scan затем проверяет строки повторно.
   Отсюда огромный Rows Removed by Index Recheck: 150000
   
   Поэтому запрос получился дорогим и медленным.
   ```

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   Sort  (cost=3100.11..3100.12 rows=5 width=7) (actual time=26.694..26.696 rows=6 loops=1)
     Sort Key: category
     Sort Method: quicksort  Memory: 25kB
     ->  HashAggregate  (cost=3100.00..3100.05 rows=5 width=7) (actual time=26.660..26.661 rows=6 loops=1)
           Group Key: category"
           Batches: 1  Memory Usage: 24kB
           ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.007..6.006 rows=150000 loops=1)
   Planning Time: 0.064 ms
   ```
   
   *Объясните результат:*
   ```
   Индекс BRIN здесь не помогает для DISTINCT по всей таблице, поэтому:
   - postgres выбирает Seq Scan
   - затем HashAggregate чтобы получить уникальные category
   - и в конце Sort для ORDER BY.
   
   Основная стоимость - именно полное чтение таблицы.
   ```

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=9.477..9.479 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=9.473..9.473 rows=0 loops=1)
           Filter: ((author)::text ~~ 'S%'::text)
           Rows Removed by Filter: 150000
   Planning Time: 0.270 ms
   Execution Time: 9.501 ms
   ```
   
   *Объясните результат:*
   ```
   Индекс BRIN по author не даёт точной выборки для префиксного поиска.
   В результате читаются все строки (Sec Scan).
   ```

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=31.244..31.245 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=31.236..31.238 rows=1 loops=1)
           Filter: (lower((title)::text) ~~ 'o%'::text)
           Rows Removed by Filter: 149999
   Planning Time: 0.332 ms
   Execution Time: 31.269 ms
   ```
   
   *Объясните результат:*
   ```
   Postgres использует индекс только при подходящем операторном классе/условиях (нужен text_pattern_ops).
   
   Также postgres мог решить, что выгоднее просканировать таблицу (Sec Scan), чем использовать индекс.
   
   Итог - полное чтение и почти все строки отфильтрованы
   ```

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.16..2366.21 rows=1 width=33) (actual time=0.929..0.930 rows=0 loops=1)
     Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
     Rows Removed by Index Recheck: 8850
     Heap Blocks: lossy=73
     ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.16 rows=75270 width=0) (actual time=0.025..0.025 rows=730 loops=1)
           Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.271 ms
   Execution Time: 0.963 ms
   ```
   
   *Объясните результат:*
   ```
   Составной BRIN (category, author) позволяет сразу отфильтровать диапазоны по обоим условиям.
   
   Поэтому Bitmap Index Scan возвращает намного меньше "возможных" строк.
   Bitmap Heap Scan делает меньше повторных проверок.
   
   Результат - Выполнение стало в разы быстрее.
   ```
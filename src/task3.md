## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=12.241..91.593 rows=499446 loops=1)
      Recheck Cond: (category = 'A'::text)
      Heap Blocks: exact=8334
      ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=11.276..11.276 rows=499446 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.246 ms
    Execution Time: 104.348 ms
    ```
    
    *Объясните результат:*
    ```
    План выбирает Bitmap Index Scan (собирает bitmap из TID всех подходящих строк).
    Затем Bitmap Heap Scan читает множество страниц таблицы, чтобы вытащить сами строки.
    Из-за рассеянного расположения строк приходится читать много разных heap-страниц (Heap Blocks: exact=8334)
    Recheck Cond присутствует, потому что при bitmap heap scan условие перепроверяется на считанных строках/страницах.
    ```

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    ```
    CLUSTER

    Query returned successfully in 744 msec.
    ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on test_cluster  (cost=59.17..7668.56 rows=5000 width=68) (actual time=9.320..46.133 rows=499446 loops=1)
      Recheck Cond: (category = 'A'::text)
      Heap Blocks: exact=4163
      ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=8.804..8.805 rows=499446 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.188 ms
    Execution Time: 58.884 ms
    ```
    
    *Объясните результат:*
    ```
    После CLUSTER таблица переписана в порядке индекса test_cluster_cat_idx.
    Строки категории A оказываются физически сгруппированы.
    План остаётся тем же, но нужно прочитать меньше страниц (Heap Blocks: exact=4163).
    Поэтому фактическое время заметно падает.
    ```

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    ```
    - Время выполнения: ускорение почти в 2 раза (104.348 ms, 58.884 ms).
    - Heap Blocks (exact): уменьшение примерно в 2 раза (8334, 4163).

    Кластеризация улучшила локальность данных для условия category='A',
    поэтому выборка той же половины таблицы требует меньше чтений страниц.
    ```

# Домашнее задание к занятию "`12.5. Индексы`" - `Белов Максим`


### Задание 1
```sql
SELECT SUM(index_length)/(SUM(index_length)+SUM(data_length))*100
FROM INFORMATION_SCHEMA.TABLES
WHERE table_schema = "sakila";  
```
![alt text](https://github.com/Maxterx10/12-05-index/blob/main/12-05-1.png)

---

### Задание 2
Запуск в изначальном виде:
```
-> Limit: 200 row(s)  (cost=0..0 rows=0) (actual time=4552..4552 rows=200 loops=1)
    -> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=4552..4552 rows=200 loops=1)
        -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=4552..4552 rows=391 loops=1)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=2079..4413 rows=642000 loops=1)
                -> Sort: c.customer_id, f.title  (actual time=2079..2125 rows=642000 loops=1)
                    -> Stream results  (cost=23.2e+6 rows=16.7e+6) (actual time=70.1..1513 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=23.2e+6 rows=16.7e+6) (actual time=70.1..1304 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=21.5e+6 rows=16.7e+6) (actual time=67.3..1158 rows=642000 loops=1)
                                -> Nested loop inner join  (cost=19.8e+6 rows=16.7e+6) (actual time=60.9..1006 rows=642000 loops=1)
                                    -> Inner hash join (no condition)  (cost=1.65e+6 rows=16.5e+6) (actual time=54.2..132 rows=634000 loops=1)
                                        -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.94 rows=16500) (actual time=28.9..45.9 rows=634 loops=1)
                                            -> Table scan on p  (cost=1.94 rows=16500) (actual time=28.9..44.8 rows=16044 loops=1)
                                        -> Hash
                                            -> Covering index scan on f using idx_title  (cost=112 rows=1000) (actual time=25.1..25.2 rows=1000 loops=1)
                                    -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=1 rows=1.01) (actual time=902e-6..0.00127 rows=1.01 loops=634000)
                                -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.001 rows=1) (actual time=128e-6..142e-6 rows=1 loops=642000)
                            -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.001 rows=1) (actual time=104e-6..119e-6 rows=1 loops=642000)
```
Хочется отметить, что логика в этом запросе отрабатывает не так, как задумано, в части `partition by c.customer_id, f.title`.  
Главная трата ресурсов в запросе уходит на соединение вложенных циклов. Это можно оптимизировать соединением таблиц (+ исправлена логика):
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p
JOIN rental r ON r.rental_id = p.rental_id
JOIN customer c ON c.customer_id = r.customer_id
JOIN inventory i ON i.inventory_id = r.inventory_id
JOIN film f ON f.film_id = i.film_id
where date(p.payment_date) = '2005-07-30'
```
EXPLAIN ANALYZE выглядит в этом случае следующим образом:
```
-> Limit: 200 row(s)  (cost=0..0 rows=0) (actual time=6.5..6.52 rows=200 loops=1)
    -> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=6.5..6.52 rows=200 loops=1)
        -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=6.5..6.5 rows=599 loops=1)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=5.64..6.39 rows=634 loops=1)
                -> Sort: c.customer_id, f.title  (actual time=5.63..5.66 rows=634 loops=1)
                    -> Stream results  (cost=36359 rows=16500) (actual time=0.0467..5.5 rows=634 loops=1)
                        -> Nested loop inner join  (cost=36359 rows=16500) (actual time=0.0432..5.32 rows=634 loops=1)
                            -> Nested loop inner join  (cost=30584 rows=16500) (actual time=0.0412..4.85 rows=634 loops=1)
                                -> Nested loop inner join  (cost=24809 rows=16500) (actual time=0.0393..4.39 rows=634 loops=1)
                                    -> Nested loop inner join  (cost=19034 rows=16500) (actual time=0.0364..4.03 rows=634 loops=1)
                                        -> Filter: ((cast(p.payment_date as date) = '2005-07-30') and (p.rental_id is not null))  (cost=1674 rows=16500) (actual time=0.0301..3.52 rows=634 loops=1)
                                            -> Table scan on p  (cost=1674 rows=16500) (actual time=0.0226..2.59 rows=16044 loops=1)
                                        -> Single-row index lookup on r using PRIMARY (rental_id=p.rental_id)  (cost=0.952 rows=1) (actual time=704e-6..718e-6 rows=1 loops=634)
                                    -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=452e-6..467e-6 rows=1 loops=634)
                                -> Single-row index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.25 rows=1) (actual time=602e-6..616e-6 rows=1 loops=634)
                            -> Single-row index lookup on f using PRIMARY (film_id=i.film_id)  (cost=0.25 rows=1) (actual time=639e-6..653e-6 rows=1 loops=634)
```
Время выполнения запроса сократилось на 3 порядка даже без введения новых индексов. Добавим индекс по payment_date и перепишем условие where:
```
-> Limit: 200 row(s)  (cost=0..0 rows=0) (actual time=4.75..4.77 rows=200 loops=1)
    -> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=4.75..4.77 rows=200 loops=1)
        -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=4.75..4.75 rows=599 loops=1)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=3.89..4.64 rows=634 loops=1)
                -> Sort: c.customer_id, f.title  (actual time=3.88..3.92 rows=634 loops=1)
                    -> Stream results  (cost=1618 rows=634) (actual time=0.0328..3.73 rows=634 loops=1)
                        -> Nested loop inner join  (cost=1618 rows=634) (actual time=0.0294..3.54 rows=634 loops=1)
                            -> Nested loop inner join  (cost=1396 rows=634) (actual time=0.0269..3.05 rows=634 loops=1)
                                -> Nested loop inner join  (cost=1175 rows=634) (actual time=0.0251..2.6 rows=634 loops=1)
                                    -> Nested loop inner join  (cost=953 rows=634) (actual time=0.0225..2.06 rows=634 loops=1)
                                        -> Filter: (p.rental_id is not null)  (cost=286 rows=634) (actual time=0.0169..1.52 rows=634 loops=1)
                                            -> Index range scan on p using idx_payment_date over ('2005-07-30 00:00:00' <= payment_date < '2005-07-31 00:00:00'), with index condition: ((p.payment_date >= TIMESTAMP'2005-07-30 00:00:00') and (p.payment_date < <cache>(('2005-07-30' + interval 1 day))))  (cost=286 rows=634) (actual time=0.0163..1.49 rows=634 loops=1)
                                        -> Single-row index lookup on r using PRIMARY (rental_id=p.rental_id)  (cost=0.952 rows=1) (actual time=739e-6..754e-6 rows=1 loops=634)
                                    -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=750e-6..764e-6 rows=1 loops=634)
                                -> Single-row index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.25 rows=1) (actual time=602e-6..616e-6 rows=1 loops=634)
                            -> Single-row index lookup on f using PRIMARY (film_id=i.film_id)  (cost=0.25 rows=1) (actual time=666e-6..680e-6 rows=1 loops=634)
```
Видно, что добавление индекса сократило время выполнения запроса еще на 25-30%.

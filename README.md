# sql_netology_6_DZ
--=============== МОДУЛЬ 6. POSTGRESQL =======================================
--= ПОМНИТЕ, ЧТО НЕОБХОДИМО УСТАНОВИТЬ ВЕРНОЕ СОЕДИНЕНИЕ И ВЫБРАТЬ СХЕМУ PUBLIC===========
SET search_path TO public;

--======== ОСНОВНАЯ ЧАСТЬ ==============

--ЗАДАНИЕ №1
--Напишите SQL-запрос, который выводит всю информацию о фильмах 
--со специальным атрибутом "Behind the Scenes".

--explain analyze cost 68.8 /  time 0.8
--explain analyze
SELECT film_id, title, array_to_string(special_features, ', ') AS special_features
FROM film
WHERE 'Behind the Scenes' = ANY(special_features)


--ЗАДАНИЕ №2
--Напишите еще 2 варианта поиска фильмов с атрибутом "Behind the Scenes",
--используя другие функции или операторы языка SQL для поиска значения в массиве.

--explain analyze cost 68.8 /  time 1.1
--explain analyze
select film_id, title, array_to_string(special_features, ', ') AS special_features
from film 
where special_features && array['Behind the Scenes']

--explain analyze cost 72.5 /  time 1.56
--explain analyze
select film_id, title, array_to_string(special_features, ', ') AS special_features
from film 
where special_features::text like '%Behind the Scenes%' 


--ЗАДАНИЕ №3
--Для каждого покупателя посчитайте сколько он брал в аренду фильмов 
--со специальным атрибутом "Behind the Scenes.

--Обязательное условие для выполнения задания: используйте запрос из задания 1, 
--помещенный в CTE. CTE необходимо использовать для решения задания.

--explain analyze cost 673 /  time 16.6
--explain analyze
WITH behind_films AS (
    SELECT film_id, title, array_to_string(special_features, ', ') AS special_features
    FROM film
    WHERE 'Behind the Scenes' = ANY(special_features)
)
SELECT customer_id, COUNT(*) AS film_count
FROM rental r
JOIN inventory i ON r.inventory_id = i.inventory_id
JOIN behind_films bf ON i.film_id = bf.film_id
GROUP BY customer_id
ORDER BY customer_id


--ЗАДАНИЕ №4
--Для каждого покупателя посчитайте сколько он брал в аренду фильмов
-- со специальным атрибутом "Behind the Scenes".

--Обязательное условие для выполнения задания: используйте запрос из задания 1,
--помещенный в подзапрос, который необходимо использовать для решения задания.
--explain analyze cost 673 /  time 15.6
--explain analyze
SELECT customer_id, COUNT(*) AS film_count
FROM rental r
JOIN inventory i ON r.inventory_id = i.inventory_id
JOIN (
    SELECT film_id, title, array_to_string(special_features, ', ') AS special_features
    FROM film
    WHERE 'Behind the Scenes' = ANY(special_features)
) f ON i.film_id = f.film_id
GROUP BY customer_id
ORDER BY customer_id

--ЗАДАНИЕ №5
--Создайте материализованное представление с запросом из предыдущего задания
--и напишите запрос для обновления материализованного представления
--explain analyze cost 675 /  time 11.77
--explain analyze
CREATE MATERIALIZED VIEW behind_the_scenes_rentals AS
SELECT customer_id, COUNT(*) AS film_count
FROM rental r
JOIN inventory i ON r.inventory_id = i.inventory_id
JOIN (
    SELECT film_id, title, array_to_string(special_features, ', ') AS special_features
    FROM film
    WHERE 'Behind the Scenes' = ANY(special_features)
) f ON i.film_id = f.film_id
GROUP BY customer_id
ORDER BY customer_id

--explain analyze cost 9.9 /  time 0.058
--explain analyze
select *
from behind_the_scenes_rentals

REFRESH MATERIALIZED VIEW behind_the_scenes_rentals


drop MATERIALIZED VIEW behind_the_scenes_rentals

--ЗАДАНИЕ №6
--С помощью explain analyze проведите анализ стоимости выполнения запросов из предыдущих заданий и ответьте на вопросы:
--1. с каким оператором или функцией языка SQL, используемыми при выполнении домашнего задания: 
--поиск значения в массиве затрачивает меньше ресурсов системы;

поиск значения в массиве затрачивает меньше ресурсов системы при испольовании материализованного представления cost 9.9 /  time 0.058, 
однако его создание требует cost 675 /  time 11.77 итого cost 684.9 и 11,828 секунд.

select (9.9 + 675), (0.058 + 11.77) 

--ЗАДАНИЕ №1
--explain analyze cost 68.8 /  time 0.8    VVVV
--ЗАДАНИЕ №2
--explain analyze cost 68.8 /  time 1.1
--explain analyze cost 72.5 /  time 1.56

Если не учитывать работу с материализованным представлением, то в 1 и 2 задании затраты ресурсов близки, хотя оператор any (ЗАДАНИЕ №1) 
затрачивает меньше времени системы для поиска значения в массиве.


--2. какой вариант вычислений затрачивает меньше ресурсов системы: 
--с использованием CTE или с использованием подзапроса.

--explain analyze cost 640 /  time 16.6 CTE
--explain analyze cost 640 /  time 15.6 Подзапрос

CTE и подзапросы имеют схожую производительность

--======== ДОПОЛНИТЕЛЬНАЯ ЧАСТЬ ==============

--ЗАДАНИЕ №1
--Выполняйте это задание в форме ответа на сайте Нетологии

--Сделайте explain analyze этого запроса.
--explain analyze 374 / 97.23

--explain analyze
select distinct cu.first_name  || ' ' || cu.last_name as name, 
	count(ren.iid) over (partition by cu.customer_id)
from customer cu
full outer join 
	(select *, r.inventory_id as iid, inv.sf_string as sfs, r.customer_id as cid
	from rental r 
	full outer join 
		(select *, unnest(f.special_features) as sf_string
		from inventory i
		full outer join film f on f.film_id = i.film_id) as inv 
		on r.inventory_id = inv.inventory_id) as ren 
	on ren.cid = cu.customer_id 
where ren.sfs like '%Behind the Scenes%'
order by count desc

--Основываясь на описании запроса, найдите узкие места и опишите их.

Потенциальные узкие места:
Фильтрация строк по спецпризнакам фильма:  
Самое медленное место - полный перебор каждой строки фильтром по строке:  WHERE inv.sf_string LIKE '%Behind the Scenes%'

Последовательное сканирование таблицы Inventory:  
Последовательность просмотра (seq scan) большого числа строк оказывает влияние на производительность.

Операции Unnest:  
Разбор значений массива добавляет дополнительную нагрузку.

Оконная функция:  
Окончательная агрегация с окном увеличивает время выполнения.


--Сравните с вашим запросом из основной части (если ваш запрос изначально укладывается в 15мс — отлично!).
--Сделайте построчное описание explain analyze на русском языке оптимизированного запроса.


ИЗ примера  explain analyze cost 374 / time 97.23
МОЙ ЗАПРОС explain analyze cost 673 /  time 16.6, в состоянии ненагруженной системы и в безветренную погоду до время падает до 15 мс.



--ЗАДАНИЕ №2
--Используя оконную функцию выведите для каждого сотрудника
--сведения о самой первой продаже этого сотрудника.


    SELECT st.staff_id, f.film_id, f.title, py.payment_date, py.amount, c.last_name, c.first_name 
FROM staff st
JOIN (
    SELECT p.*, 
           ROW_NUMBER() OVER(PARTITION BY p.staff_id ORDER BY p.payment_date ASC) AS row_num
    FROM payment p
) py ON st.staff_id = py.staff_id
JOIN rental r ON py.rental_id = r.rental_id
JOIN inventory i ON r.inventory_id = i.inventory_id
join customer c on r.customer_id = c.customer_id 
JOIN film f ON i.film_id = f.film_id
WHERE py.row_num = 1

        

--ЗАДАНИЕ №3
--Для каждого магазина определите и выведите одним SQL-запросом следующие аналитические показатели:
-- 1. день, в который арендовали больше всего фильмов (день в формате год-месяц-день)
-- 2. количество фильмов взятых в аренду в этот день
-- 3. день, в который продали фильмов на наименьшую сумму (день в формате год-месяц-день)
-- 4. сумму продажи в этот день

WITH rental_counts AS (
    SELECT
        s.store_id,
        r.rental_date::date AS rental_day,
        COUNT(*) AS rental_count,
        ROW_NUMBER() OVER (PARTITION BY s.store_id ORDER BY COUNT(*) DESC) AS rn_rental
    FROM
        rental r
    JOIN
        inventory i ON r.inventory_id = i.inventory_id
    JOIN
        store s ON i.store_id = s.store_id
    GROUP BY
        s.store_id, rental_day
),
max_rental_day AS (
    SELECT
        store_id,
        rental_day,
        rental_count
    FROM
        rental_counts
    WHERE
        rn_rental = 1
),
payment_sums AS (
    SELECT
        s.store_id,
        p.payment_date::date AS payment_day,
        SUM(p.amount) AS payment_sum,
        ROW_NUMBER() OVER (PARTITION BY s.store_id ORDER BY SUM(p.amount) ASC) AS rn_payment
    FROM
        payment p
    JOIN
        rental r ON p.rental_id = r.rental_id
    JOIN
        inventory i ON r.inventory_id = i.inventory_id
    JOIN
        store s ON i.store_id = s.store_id
    GROUP BY
        s.store_id, payment_day
),
min_payment_day AS (
    SELECT
        store_id,
        payment_day,
        payment_sum
    FROM
        payment_sums
    WHERE
        rn_payment = 1
)
SELECT
    mrd.store_id,
    mrd.rental_day AS max_rental_day,
    mrd.rental_count,
    mpd.payment_day AS min_payment_day,
    mpd.payment_sum
FROM
    max_rental_day mrd
JOIN
    min_payment_day mpd ON mrd.store_id = mpd.store_id;



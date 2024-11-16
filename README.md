SET search_path to bookings

-- Выведите название самолетов, которые имеют менее 50 посадочных мест

SELECT  a.model, count(*) as "кол-во мест"
 FROM  aircrafts a
 join seats s on s.aircraft_code = a.aircraft_code 
 GROUP BY a.model
 having count(*) < 50

--Выведите для каждого покупателя его адрес проживания, город и страну проживания.

select concat_ws(' ', last_name, first_name) as customer_name, address, city,country
from customer cus
join address a on a.address_id = cus.address_id 
join city c on c.city_id = a.city_id
join country con on con.country_id  = c.country_id

--Посчитайте для каждого покупателя 4 аналитических показателя:
--  1. количество фильмов, которые он взял в аренду
--  2. общую стоимость платежей за аренду всех фильмов (значение округлите до целого числа)
--  3. минимальное значение платежа за аренду фильма
--  4. максимальное значение платежа за аренду фильма

select concat_ws(' ', c.last_name, c.first_name) as "Фамилия и имя покупателя", 
count(r.rental_id) as "Кол-во фильмов" , 
round(sum(p.amount)) as "общая стоимость платежей", 
min(p.amount) as "минимальное значение платежа" ,
max(p.amount)  as "максимальное значение платежа"
from customer c 
join rental r on r.customer_id = c.customer_id 
join payment p on p.customer_id = c.customer_id and p.rental_id = r.rental_id 
group by c.customer_id 

--Используя данные из таблицы rental о дате выдачи фильма в аренду (поле rental_date) и 
--дате возврата (поле return_date), вычислите для каждого покупателя среднее количество 
--дней, за которые он возвращает фильмы. В результате должны быть дробные значения, а не интервал.

select r.customer_id as "ID покупателя", round(avg(date_part('day', return_date  - rental_date::date))::numeric,2) as "среднее кол-во дней возврата" 
from rental r 
group by r.customer_id 
order by r.customer_id
 
-- Выведите процентное изменение ежемесячной суммы бронирования билетов, округленной до сотых.

select *,
round (((a.sum - lag(a.sum, 1, 0) over (order by "date_trunc"))/
		 lag(a.sum, 1) over (order by "date_trunc"))*100,2) as "изм-е, %"
from (
select  (date_trunc('month', book_date))::date, sum(total_amount)
from bookings b 
group by date_trunc('month', book_date)
order by date_trunc('month', book_date)) a
 
-- Выведите названия самолетов не имеющих бизнес - класс. Решение должно быть через функцию array_agg.
 
 select a.model 
 from
 (select  a.model, array_agg(fare_conditions)
 from seats s 
 join aircrafts a on a.aircraft_code = s.aircraft_code
 group by a.model
 having array_position(array_agg(fare_conditions), 'Business') is null) a
 
 

-- Найдите процентное соотношение перелетов по маршрутам от общего количества перелетов.
 --Выведите в результат названия аэропортов и процентное отношение.
 --Решение должно быть через оконную функцию.

select departure_airport_name, arrival_airport_name, ((count(*))*100/ sum(count(*))  over()) as "% отношение"
from flights_v fv 
group by departure_airport_name, arrival_airport_name


-- Выведите количество пассажиров по каждому коду сотового оператора, если учесть, что код оператора - это три символа после +7

select count (passenger_name) as "кол-во пассажиров", 
substring(contact_data ->> 'phone' from 3 for 3 ) as "код сотового оператора"
from tickets t
group by 2
order by 2


	
	
		


	

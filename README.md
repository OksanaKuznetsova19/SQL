SET search_path to bookings

--1. Выведите название самолетов, которые имеют менее 50 посадочных мест

SELECT  a.model, count(*) as "кол-во мест"
 FROM  aircrafts a
 join seats s on s.aircraft_code = a.aircraft_code 
 GROUP BY a.model
 having count(*) < 50
 
--2. Выведите процентное изменение ежемесячной суммы бронирования билетов, округленной до сотых.

select *,
round (((a.sum - lag(a.sum, 1, 0) over (order by "date_trunc"))/
		 lag(a.sum, 1) over (order by "date_trunc"))*100,2) as "изм-е, %"
from (
select  (date_trunc('month', book_date))::date, sum(total_amount)
from bookings b 
group by date_trunc('month', book_date)
order by date_trunc('month', book_date)) a
 
--3. Выведите названия самолетов не имеющих бизнес - класс. Решение должно быть через функцию array_agg.
 
 select a.model 
 from
 (select  a.model, array_agg(fare_conditions)
 from seats s 
 join aircrafts a on a.aircraft_code = s.aircraft_code
 group by a.model
 having array_position(array_agg(fare_conditions), 'Business') is null) a
 
 
 --4. Вывести накопительный итог количества мест в самолетах по каждому аэропорту на каждый день, учитывая только те самолеты, которые летали пустыми 
--и только те дни, где из одного аэропорта таких самолетов вылетало более одного.
--В результате должны быть код аэропорта, дата, количество пустых мест в самолете и накопительный итог.
	
		
	select t."аэропорт вылета", t."фактическое время вылета", t.count, t.sum
from 
(select t."фактическое время вылета", t."аэропорт вылета", t.aircraft_code, s.count,
count(t.aircraft_code) over (partition by t."фактическое время вылета", t."аэропорт вылета") as "кол-во мест",
sum(s.count) over(partition by t."фактическое время вылета", t."аэропорт вылета" ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
from
(select date_trunc('day', f.actual_departure) as "фактическое время вылета",
f.departure_airport as "аэропорт вылета",
f.aircraft_code
from flights f 
join boarding_passes bp on bp.flight_id =f.flight_id 
where bp.boarding_no = 0
and f.actual_departure is not null 
and f.status like 'D%' or f.status like 'A%'
group by "фактическое время вылета", "аэропорт вылета", f.aircraft_code) t
left join 
(select aircraft_code, count (*) 
from seats s 
group by 1) s on s.aircraft_code = t.aircraft_code
order by 1,2) t
where "кол-во мест" > 1

	

--5. Найдите процентное соотношение перелетов по маршрутам от общего количества перелетов.
 --Выведите в результат названия аэропортов и процентное отношение.
 --Решение должно быть через оконную функцию.

select departure_airport_name, arrival_airport_name, ((count(*))*100/ sum(count(*))  over()) as "% отношение"
from flights_v fv 
group by departure_airport_name, arrival_airport_name


--6. Выведите количество пассажиров по каждому коду сотового оператора, если учесть, что код оператора - это три символа после +7

select count (passenger_name) as "кол-во пассажиров", 
substring(contact_data ->> 'phone' from 3 for 3 ) as "код сотового оператора"
from tickets t
group by 2
order by 2

--7. Классифицируйте финансовые обороты (сумма стоимости перелетов) по маршрутам:
-- До 50 млн - low
-- От 50 млн включительно до 150 млн - middle
-- От 150 млн включительно - high
	
	select a.ca, count(*)
from(
select  f.departure_airport, f.arrival_airport, sum(tf.amount),
	case
		when sum(tf.amount) < 50000000 then 'low'
		when sum(tf.amount) >= 50000000  and sum(tf.amount) < 150000000 then 'middle'
		when sum(tf.amount) >= 150000000 then 'high'
		end  ca
	from flights f 
	join ticket_flights tf on tf.flight_id = f.flight_id 
	group by f.departure_airport, f.arrival_airport) a
	group by 1
	
	
--8. Вычислите медиану стоимости перелетов, 
--медиану размера бронирования 
--и отношение медианы бронирования к медиане стоимости перелетов, округленной до сотых
		
	select  "медиана бронирования", "медиана стоимости перелетов", round(("медиана бронирования"/"медиана стоимости перелетов")::numeric, 2) as "медиана брон-ия/медиана стоимости"
 from
 (with cte as(
		select percentile_cont(0.5) within group (order by total_amount)
		from bookings
		union
		select percentile_cont(0.5) within group (order by amount)
		from ticket_flights tf )
	select 
			max(case when percentile_cont='13400' then percentile_cont  end) "медиана стоимости перелетов",
max(case when percentile_cont='55900' then percentile_cont  end)  "медиана бронирования" from cte)
		
--9. Найдите значение минимальной стоимости полета 1 км для пассажиров. То есть нужно найти расстояние между аэропортами и с учетом стоимости перелетов получить искомый результат
  --Для поиска расстояния между двумя точками на поверхности Земли используется модуль earthdistance.
  --Для работы модуля earthdistance необходимо предварительно установить модуль cube.
  --Установка модулей происходит через команду: create extension название_модуля
		
create extension cube
		
create extension earthdistance
		
		
with cte as(
	select f.flight_id, 
		    departure_airport, a1.longitude, a1.latitude, 
			arrival_airport, a2.longitude, a2.latitude,
			round((earth_distance (ll_to_earth ( a1.latitude, a1.longitude), ll_to_earth (a2.latitude, a2.longitude))::numeric)/1000 ,2) as "расстояние"
		from flights f
		join airports a1 on f.departure_airport = a1.airport_code
		join airports a2 on f.arrival_airport = a2.airport_code
		group by 1, 3, 4, 6, 7)
		
select cte.flight_id, cte.departure_airport, cte.arrival_airport, "расстояние", tf.amount as "стоимость перелета", round (tf.amount/"расстояние", 2) as "стоимость 1 км"
from ticket_flights tf 
join cte  on cte.flight_id = tf.flight_id 
group by cte.flight_id, "расстояние", tf.amount, cte.departure_airport, cte.arrival_airport
order by "стоимость 1 км"
fetch first 1 rows only

	

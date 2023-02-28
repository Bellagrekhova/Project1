# Анализ эффективности работы компании
# Задачи:

1.Оценить динамику продаж и распределение выручки по товарам

2.Составить портрет клиента и проанализировать, какие клиенты приносят больше всего выручки

3.Проанализировать логистику компании: определить, все ли заказы доставляются в срок, и в каком штате лучше всего открывать офлайн-магазин

# Анализ эффективности продаж

Определим сумму выручки по месяцам:

SELECT date_trunc('MONTH', sd.order_date)::date AS date, round(sum(sp.price*sc.quantity*(1-sc.discount))) AS revenue
/*
*в первом столбце указана дата,когда заказ поступил в работу,сокращенная до месяца 
*во втором столбце указана выручка по заказам за каждый месяц
*/

FROM sql.store_products AS sp

JOIN sql.store_carts AS sc

	ON sp.product_id=sc.product_id
	
JOIN sql.store_delivery AS sd

	ON sc.order_id=sd.order_id
	
GROUP BY  1

ORDER BY  1

![2023-02-09 (1)](https://user-images.githubusercontent.com/125760683/221899514-3640f75e-da36-498d-811d-00a86ce65081.png)

На графике видим,что в целом выручка растет. Однако можно заметить значительное проседание выручки в январе и феврале каждого года. При этом наибольшие показатели по выручке, как правило, достигаются в ноябре-декабре.

Определение суммы выручки товаров по различным категориям и подкатегориям:

SELECT category,
		 subcategory,
		 round(sum(sp.price*sc.quantity*(1-sc.discount))) AS revenue--столбцы с категориями, подкатегориями и выручкой, округленной до целых чисел
FROM sql.store_products AS sp
JOIN sql.store_carts AS sc
	ON sp.product_id=sc.product_id
JOIN sql.store_delivery AS sd
	ON sc.order_id=sd.order_id
GROUP BY  1, 2--группируем по категориям и подкатегориям
ORDER BY  3 DESC --сортировка по выручке в порядке убывания

![2023-02-14 (5)](https://user-images.githubusercontent.com/125760683/221901029-5f3570df-a763-45c6-bd86-2b3faa4b1d80.png)

Распределение выручки по категориям, наибольший объем выручки приходится на категорию ‘Technology’.

![2023-02-14 (6)](https://user-images.githubusercontent.com/125760683/221901310-1c392f8a-0d8e-439b-baaf-4b90f50e8567.png)

Распределение выручки по подкатегориям, наибольший объем выручки приходится на подкатегорию ‘Chairs’.

Выделяем топ-25 товаров по объему выручки:

with total AS 
	(SELECT sum(sp.price*sc.quantity*(1-sc.discount)) AS total_revenue
	FROM sql.store_products AS sp
	JOIN sql.store_carts AS sc
		ON sp.product_id=sc.product_id ),/*создаем cte, чтобы определить общую выручку по всем товарам*/
	table1 as
	(SELECT DISTINCT product_nm,
		 round(sum(sp.price*sc.quantity*(1-sc.discount)), 2) AS revenue,
		 sum(sc.quantity) AS quantity
	FROM sql.store_products AS sp
	JOIN sql.store_carts AS sc
		ON sp.product_id=sc.product_id
	JOIN sql.store_delivery AS sd
		ON sc.order_id=sd.order_id
	GROUP BY  1 )/*создаем cte, в котором указаны наименования товаров,              выручка по разным товарам и количество проданных товаров*/
SELECT table1.product_nm,
		 revenue,
		 quantity,
		 round(revenue/total_revenue*100, 2) AS percent_from_total/*
		 *выводим наименование товаров, выручку,количество проданных товаров, а также какую долю от общей выручки составила выручка от конкретного товара 
		 */
FROM table1
JOIN total 
	ON true
ORDER BY  revenue DESC limit 25--сортируем по выручке в порядке убывания и выводим только первые 25 товаров

![2023-02-15 (1)](https://user-images.githubusercontent.com/125760683/221901696-26550a89-c8aa-47c4-9be6-0ae430b9492e.png)


По доле от общей выручки товар ‘Canon imageCLASS 2200 Advanced Copier’ значительно превосходит остальные товары. 

# Составление портрета клиента


Определяем выручку от B2B и B2C клиентов:

SELECT scu.category,
		 count(distinct scu.cust_id) AS cust_cnt,
		 round(sum(sca.quantity*sp.price*(1-sca.discount))) AS revenue/*в первом столбце определяем категорию клиента, 
		 *во втором считаем количество клиентов в каждой категории, для этого считаем только уникальные id клиента,
		 *в третьем столбце считаем выручку от клиентов по каждой категории
		 */
FROM sql.store_customers scu
JOIN sql.store_delivery sd
	ON scu.cust_id=sd.cust_id
JOIN sql.store_carts sca
	ON sd.order_id=sca.order_id
JOIN sql.store_products sp
	ON sp.product_id=sca.product_id
GROUP BY  1--группируем по категории
ORDER BY  3 DESC--сортируем по выручке в порядке убывания

![2023-02-14 (2)](https://user-images.githubusercontent.com/125760683/221901952-3b530456-aac4-448e-927c-82f8717b8ed8.png)

На графике видим, что выручка от B2B-клиентов в разы больше, чем от B2C-клиентов. Далее проанализируем B2B-клиентов более подробно. 

Динамика новых B2B-клиентов по месяцам:


with cohorts AS 
	(SELECT sc.cust_id,
		 min(order_date) AS date
	FROM sql.store_delivery sd
	JOIN sql.store_customers sc
		ON sd.cust_id=sc.cust_id
	GROUP BY  1, sc.category
	HAVING sc.category='Corporate')--создаем cte, в котором определяем самую первую дату совершения покупок по каждому корпортивному клиенту
SELECT date_trunc('MONTH',date)::date AS date, count(distinct cust_id) new_custs--выводим месяц из cte и считаем количество корпоративных клиентов, совершивших первые покупки в каждом месяце
FROM cohorts
GROUP BY  1--группируем по дате
ORDER BY  1 --сортируем по количеству клиентов в порядке возрастания

![2023-02-14 (1)](https://user-images.githubusercontent.com/125760683/221902162-2fcf54a1-ced7-418d-a501-2df9f73a1622.png)

Анализируя график, можно заметить, что новые корпоративные клиенты практически не привлекаются, начиная с 2018 года. Так как эти клиенты приносят большую часть выручки, необходимо возобновить привлечение новых корпоративных клиентов,которое уже проводилось в 2017 году. 

Определяем среднее количество различных товаров в чеке у корпоративных клиентов:

with orders AS 
	(SELECT sd.order_id,
		 count(distinct sp.product_id) orders
	FROM sql.store_customers scu
	JOIN sql.store_delivery sd
		ON sd.cust_id=scu.cust_id
	JOIN sql.store_carts sca
		ON sd.order_id=sca.order_id
	JOIN sql.store_products sp
		ON sca.product_id=sp.product_id
	GROUP BY  1, scu.category
	HAVING scu.category='Corporate')--создаем cte, в котором определяем выручку по корпоративным клиентам
SELECT round(avg(orders), 1)--считаем среднее значение выручки, округляем до 1 знака после запятой
FROM orders 


Получаем значение 2.


Определяем среднюю сумму заказов у корпоративных клиентов:

with cte AS 
	(SELECT sd.order_id,
		 sum(sp.price*sca.quantity*(1-sca.discount)) amount
	FROM sql.store_customers scu
	JOIN sql.store_delivery sd
		ON sd.cust_id=scu.cust_id
	JOIN sql.store_carts sca
		ON sd.order_id=sca.order_id
	JOIN sql.store_products sp
		ON sca.product_id=sp.product_id
	GROUP BY  1, scu.category
	HAVING scu.category='Corporate' )
SELECT round(avg(amount), 1)
FROM cte 

Средняя сумма составляет 285,9.



Определяем среднее количество различных офисов у корпоративных клиентов:


with cte AS 
	(SELECT scu.cust_id,
		 count(distinct sd.zip_code) offices
	FROM sql.store_customers scu
	JOIN sql.store_delivery sd
		ON sd.cust_id=scu.cust_id
	GROUP BY  1, scu.category
	HAVING scu.category='Corporate')--создаем cte, где считаем количество различных офисов у корпоративных клиентов
SELECT round(avg(offices), 1)--считаем среднее количество, округляем до 1 знака после запятой
FROM cte 


Получаем значение 6,2.

# Анализ логистики компании


Определим, какая доля заказов выполняется в срок по каждой категории доставки:


with table1 as
	(SELECT order_id,
		 ship_mode,
		 ship_date-order_date AS order_days
	FROM sql.store_delivery ),--создаем cte, в котором определяем, сколько дней шел заказ
	orders_type AS 
	(SELECT order_id,
		
		CASE
		WHEN order_days>6
			AND ship_mode='Standard Class' THEN
		'not_success'
		WHEN order_days>4
			AND ship_mode='Second Class' THEN
		'not_success'
		WHEN order_days>3
			AND ship_mode='First Class' THEN
		'not_success'
		WHEN order_days>0
			AND ship_mode='Same Day' THEN
		'not_success'
		ELSE 'success'
		END AS status, ship_mode
	FROM table1 ),--в этом cte присваиваем заказам статус 'успешно', если заказ доставили вовремя и 'не успешно', если с опозданием
	late_orders AS 
	(SELECT ship_mode,
		 count(order_id) orders_late
	FROM orders_type
	GROUP BY  1, orders_type.status
	HAVING status='not_success'), --считаем, сколько заказов были доставлены не вовремя по каждому типу доставки
	cte AS 
	(SELECT sd.ship_mode,
		 count(order_id) AS orders_cnt,
		 orders_late,
		 count(order_id)-orders_late AS success
	FROM sql.store_delivery sd
	LEFT JOIN late_orders lo
		ON lo.ship_mode=sd.ship_mode
	GROUP BY  1,3)--считаем количество заказов, в зависимости от типа доставки, а также определяем, сколько заказов были доставлены вовремя по каждому типу
SELECT ship_mode,
		 orders_cnt,
		 orders_late,
		 round(success/orders_cnt::numeric*100,  2) success--определяем долю вовремя доставленных заказов
FROM cte
ORDER BY  4 --сортируем по доле вовремя доставленных заказов в порядке возрастания

![2023-02-14 (3)](https://user-images.githubusercontent.com/125760683/221902420-14562dde-42ed-4a41-a97a-74073e436374.png)

На графике можно заметить, что больше всего посылок с опозданием доставляются типом доставки ‘Standart Class’. Однако наименьший процент вовремя доставленных посылок приходится на тип доставки ‘Second Class’, поэтому проанализируем его более подробно. 

Определяем самый популярный штат по количеству доставок:

SELECT state,
		 count(order_id) AS orders_amount
FROM sql.store_delivery
GROUP BY  1
ORDER BY  2 DESC limit 1 

Самый популярный штат по количеству доставок - California.


Определяем самый популярный город по количеству доставок: 


SELECT city,
		 count(order_id) AS orders_amount
FROM sql.store_delivery
GROUP BY  1
ORDER BY  2 DESC limit 1 

Самый популярный город по количеству доставок - New York City.


Определяем количество доставок по штатам:

SELECT state,
		 count(order_id) AS orders_amount
FROM sql.store_delivery
GROUP BY  1
ORDER BY  2 DESC 

![2023-02-14](https://user-images.githubusercontent.com/125760683/221902562-7884ea79-4288-4c2f-950e-1637b4db8001.png)

Самым перспективным для открытия офлайн-магазина является штат California, который опережает остальные штаты по количеству доставок. Количество доставок в этот штат составило 1021. Это почти вдвое больше,чем количество доставок в штат New York, который находится на 2-м месте в рейтинге по этому показателю.

Определяем долю заказов, отправленных вторым классом, которые были доставлены с опозданием, по кварталам:

WITH table1 as
	(SELECT order_id,
	
		 ship_date,
		 
		 ship_mode,
		 
		 ship_date-order_date AS order_days
		 
	FROM sql.store_delivery
	
	WHERE ship_mode='Second Class' ),--считаем, за сколько дней были доставлены заказы по типу доставки "второй класс"
	
	orders_type AS 
	
	(SELECT order_id,
		
		CASE
		
		WHEN order_days>4 THEN
		
		'not_success'
		
		ELSE 'success'
		
		END AS status, ship_date
		
	FROM table1 ),--присваиваем каждому заказу, который был доставлен по типу доставки "второй класс", статус "успешно", если доставлен вовремя и "не успешно", если с опозданием
	
	late_orders AS 
	
	(SELECT date_trunc('QUARTER', ship_date) AS ship_date, count(order_id) orders_late
	
	FROM orders_type
	
	GROUP BY  1, orders_type.status
	
	HAVING status='not_success'),--считаем количество доставок с опозданием по каждому кварталу 
	
	cte AS 
	
	(SELECT date_trunc('QUARTER', table1.ship_date) AS ship_date, count(table1.order_id) AS orders_cnt
	
	FROM table1
	
	LEFT JOIN late_orders lo
	
		ON lo.ship_date=table1.ship_date
		
	GROUP BY  1)--считаем общее количество доставок по каждому кварталу
	
SELECT date_trunc('QUARTER', cte.ship_date) AS ship_date, round(orders_late/orders_cnt::numeric*100, 2) late_orders

/*
*считаем процент доставок с опозданием ообщего числа доставок по каждому кварталу
*/

FROM cte

JOIN late_orders lo

	ON lo.ship_date=cte.ship_date
	
ORDER BY  1 --сортируем по дате в порядке возрастания

![2023-02-15](https://user-images.githubusercontent.com/125760683/221902686-87a7b937-52f8-4683-bf86-48d56ac969ac.png)

На графике видим, что наименьшая доля доставок с опозданием приходится на первый квартал 2019 года и составляет 7,5% от общего числа доставок за этот период. В среднем этот показатель варьируется от 13% до 30%. Показатели за первый квартал 2021 года можно не учитывать при анализе, т к данных за этот период недостаточно.


# Выводы:

Ежемесячная выручка компании за 2017-2021гг увеличилась с 9,300 до 48,300. Однако в начале каждого года в январе-феврале происходит значительное “проседание” выручки.

Наибольшую выручку компания получает от продажи товаров категории “Technology”. Первое место по доле от общей выручки занимает товар ‘Canon imageCLASS 2200 Advanced Copier’. Сумма от его продаж составляет более 2,5% от общей выручки, что значительно превышает показатели по другим товарам.

Большую часть выручки компании приносят корпоративные клиенты. Однако, начиная с 2019 года, новые корпоративные клиенты практически не привлекались.
Больше всего заказов с опозданием доставляются типом доставки “Second Class”. За период 2017-2020гг показатель доставок с опозданием в среднем составляет от 13% до 30%.

Наибольшее количество заказов доставляются в штат California. За 2017-2020гг в штат был доставлен 1021 заказ, поэтому целесообразно открыть в этом штате офлайн-магазин, что может сократить затраты на логистику.










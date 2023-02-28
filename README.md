# project2
Задачи:

Оценить динамику продаж и распределение выручки по товарам
Составить портрет клиента и проанализировать, какие клиенты приносят больше всего выручки
Проанализировать логистику компании: определить, все ли заказы доставляются в срок, и в каком штате лучше всего открывать офлайн-магазин

Анализ эффективности продаж

Определим сумму выручки по месяцам:

SELECT date_trunc('MONTH', sd.order_date)::date AS date, round(sum(sp.price*sc.quantity*(1-sc.discount))) AS revenue /*
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

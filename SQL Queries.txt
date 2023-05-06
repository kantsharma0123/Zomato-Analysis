--Q1 Metric 1 - What is the total amount each customer spent on zomato

select s.userid,sum(price) as total_money_spent from sales s 
join product p
on s.product_id = p.product_id
group by s.userid

--Q2 How many days each customer visited zomato

select userid, count(created_date) as no_of_times_vited from sales
group by userid


--Q3 What was the first product purchased by each of the customer

select x.userid,x.first_order,b.product_id from
(
select s.userid,min(s.created_date) as first_order from sales s
group by s.userid
)x
join sales b
on x.userid = b.userid
and x.first_order = b.created_date

--Q4 What is the most purchased item on the menu and how many times was it purchased by the customers

with cte as
(
select product_id,count(product_id) as most_purchased,RANK() over(order by count(product_id) desc) as cou  from sales
group by product_id
)
select userid,product_id,count(product_id) as times_purchased  from sales
where product_id = (select product_id from cte where cou = 1)
group by userid,product_id


--Q5 which item is the most popular for each customer

select x.userid,x.product_id as most_purchasedProduct from
(
select userid,product_id,count(product_id) as times_bought,RANK() over(partition by userid order by count(product_id) desc) as rank  from sales
group by userid,product_id
)x
where x.rank = 1

--Q6 Which item was purchased first by the customer after they became a member

select x.userid,x.created_date,x.product_id as first_product_after_gold from
(
select s.userid,created_date,product_id,RANK() over(partition by s.userid order by s.created_date) as ranki,g.gold_signup_date from sales s
join goldusers_signup g
on s.userid = g.userid
and s.created_date > g.gold_signup_date
)x
where x.ranki = 1


--Q7 Which item was purchased just before the customer became a member


select x.userid,x.created_date,x.product_id from
(
select s.userid,s.created_date,s.product_id,g.gold_signup_date,RANK() over(partition by s.userid order by s.created_date desc) as ranki from sales s
join goldusers_signup g
on s.userid = g.userid
and s.created_date < g.gold_signup_date
)x
where x.ranki = 1


--Q8 What is the total orders and amount spent for each member before they became a member

select userid,count(sprod) as total_items_bought,sum(price) as total_spent from
(
select s.userid,s.created_date,s.product_id as sprod from sales s
join goldusers_signup g
on s.userid = g.userid 
and s.created_date < g.gold_signup_date
)x
join product p
on x.sprod = p.product_id
group by userid


--Q9 If buying each product generates points for eg 5rs=2 zomato point 
--and each product has different purchasing points for eg for p1 5rs=1 zomato point,
--for p2 10rs= 5 zomato point and p3 5rs=1 zomato point  2rs =1zomato point, calculate points collected by each customer and for 
--which product most points have been given till now 
-- Also calcuate the cashback for values 60 zomato points = 5 rupees , calucate cashback money in the wallet

-- part 1 (calculating points and cashbacks)

select  userid, sum(noe)  as total_zomato_points,cast(sum(noe) * 0.12 as int)  as total_cashback_wallet from
(
select s.userid,s.product_id,sum(price) as total_spent, case when s.product_id = 1 then sum(p.price)/5 *1  when s.product_id = 2 then
sum(p.price)/10 *5 else sum(p.price)/5 *1 end as noe
from sales s
join product p
on s.product_id = p.product_id
group by s.userid,s.product_id
)x
group by userid

-- part 2 ( For which product most points have been given till now )


select top  1 * from
(
select product_id,sum(noe) as max_points_for_product from
(
select s.userid,s.product_id,sum(price) as total_spent, case when s.product_id = 1 then sum(p.price)/5 *1  when s.product_id = 2 then
sum(p.price)/10 *5 else sum(p.price)/5 *1 end as noe
from sales s
join product p
on s.product_id = p.product_id
group by s.userid,s.product_id
)x
group by product_id
)y
order by y.max_points_for_product desc

-- part 3 ( For which product most points have been given till now customer wise)

select userid,product_id,noe as total_points from
(
select userid,product_id,noe,RANK() over(partition by userid order by noe desc) as ranki from
(
select s.userid,s.product_id,sum(price) as total_spent, case when s.product_id = 1 then sum(p.price)/5 *1  when s.product_id = 2 then
sum(p.price)/10 *5 else sum(p.price)/5 *1 end as noe
from sales s
join product p
on s.product_id = p.product_id
group by s.userid,s.product_id
)x
)y
where y.ranki = 1

--Q10  In the first year after a customer joins the gold program (including the join date )
--irrespective of what customer has purchased earn 5 zomato points for every 10rs spent who earned 
-- more 1 or 3 what int earning in first yr ? 2zomato point = 1 rupees

select suserid,sum(price) as total_spent,(sum(price)/10)*5 as zomato_points,cast(((sum(price)/10)*5) * 0.5 as int) as cashback_earned from
(
select s.userid as suserid,s.created_date,s.product_id,g.userid as guserid,g.gold_signup_date,p.price,datediff(MONTH,g.gold_signup_date,s.created_date) as timeline from sales s
join goldusers_signup g
on s.userid = g.userid
join product p
on s.product_id = p.product_id
where s.created_date >= g.gold_signup_date
and datediff(MONTH,g.gold_signup_date,s.created_date) <=12
)x
group by suserid


--Q11   rnk all transaction of the customers, user_id wise

select *,rank() over(partition by userid order by created_date) as rank from sales

--Q12 rank all the transactions for each member whenever they are zomato gold member, for every non gold member transaction mark as NA

select s.userid,s.created_date,case when gold_signup_date > created_date then null when g.userid is null then null else rank() over(partition by s.userid order by s.created_date desc) end as ranked from sales s
left join goldusers_signup g
on s.userid = g.userid

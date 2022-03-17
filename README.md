# DVD_Rentals
SQL scripts

1. Find the top 10 most popular movies from rentals in H1 2005, by category.
```
-- The SQL query below is performing the following:
-- 1. Filtering rental data by the date partition, to only return H1 2005, 
--    and optimise the query so it's not scanning the whole database.
-- 2. Then it's grouping by inventory_id to reduce the number of rows before 
--    performing the joins in the next step.
-- 3. Then it's calculating the total demand per film.
-- 4. Then it introduces the ranking system, where the films with the 
--    highest demand appear in descending order with the respective rank.
-- 5. Lastly, we only return the category name and 10 films per each category
--    by specifying to return row numbers 1 to 10 only.

select 
      d.name as category
    , d.title as film
from 
	(select      
                  c.name
		, c.title
		, c.total_renting
		, row_number() over(partition by c.name order by c.total_renting desc) as rn 
	 from    
		(select  c.name
		       , f.title
		       , sum(b.renting_frequency) as total_renting        
		 from 
			(select   a.inventory_id
				 , count(*) as renting_frequency            
			from (select * 
			      from rental 
			      where rental_date between '2005-01-01 00:00:00'::timestamp and '2005-07-01 00:00:00'::timestamp
			     ) as a 
			group by a.inventory_id
			) as b  
      
		join inventory as i using (inventory_id) 
		join film as f using (film_id) 
		join film_category as fc using (film_id) 
		join category as c using (category_id)
    
		group by c.name, f.title
		) as c
	) as d
where rn < 11
;
```

2. Find the avg. customer value per store by month for rentals in 2005. Please exclude the top & bottom 10% of customers by value from the analysis.
```
With t1 as (
		select * 
		from rental 
		where to_char(rental_date, ‘YYYY’) = ‘2005’
		)
, t2 as(
	Select 
		 a.customer_id
		,a.amount
	From (
		select
		 	 p.customer_id
			,p.amount
			,percent_rank() over (order by p.amount) as pct_rank
		from (select customer_id, sum(amount) as amount from payment group by customer_id) as p
		) as a
	Where a.pct_rank between 0.1 and 0.9
	)

select 
	  s.store_id
	, extract(month from t1.rental_date) as month
	, avg(t2.amount) as avg_customer_value 
From t1
Join t2 
On t1.customer_id = t2.customer_id
join staff as s
On s.staff_id = t1.staff_id

Group by store_id, month
order by store_id, month
;
```

3. Create a table, film_recommendations, which provides 10 film recommendations per customer. Future recommendations could be based upon a customer's previous film choices, other customer's choices etc. Please only use SQL to complete this and include all the DDL needed to create the table.
```
Create table schema.film_recommendations 
(
customer_id		integer,
film_recommendations 	varchar(40) not null
)
;

-- What are the customer's top 3 favourite genres that they rented in the past
-- as well as films from those 3 categories?

Drop table if exists schema.customer_top_3_categories;
Create table schema.customer_top_3_categories as 

With t1 as (
		Select 
	  		  r.customer_id
			, fc.category_id
			, fc.film_id
			, count(*) as renting_frequency

		From rental r
		Join inventory i
		Using (inventory_id)
		Join film f 
		Using (film_id)
		Join film_category fc
		Using (film_id)

		Group by 1,2,3
                )
, t2 as (
		Select 
			  customer_id
			, category_id
			, sum(renting_frequency) as renting_by_genre
		From t1
		group by 1,2
       	)
, t3 as (
		select 
			 customer_id
			,category_id
			,renting_by_genre
			,row_number() over(partition by customer_id, category order by renting_by_genre desc) as rn 
		from t2
	)
, t4 as(
		select 
			 customer_id
			,category_id
		from t3
		where rn <= 3
	)

select 
 	 a.customer_id,
	,a.category_id
	,b.film_id
from t4 as a
join t1 as b using (customer_id, category_id)
;

-- Top 100 films per category from all customers choices

Drop table if exists schema.top_100_films_per_category;
Create table schema.top_100_films_per_category as 

select b.category_id, b.film_id, b.title as film, b.total_renting
from 
	(select     a.category_id
		  , a.film_id
		  , a.title
		  , a.total_renting
		  , row_number() over(partition by a.category_id order by a.total_renting desc) as rn 
	 from 
		(select    fc.category_id
		         , fc.film_id
		         , f.title
		         , sum(p.renting_frequency) as total_renting 
		 from 
			(select  inventory_id
				, count(*) as renting_frequency 
			from rental 
			group by inventory_id
			) as p 
      
                join inventory as i using (inventory_id) 
		join film as f using (film_id) 
		join film_category as fc using (film_id) 

		group by fc.category_id, fc.film_id, f.title
		) as a
	) as b 
where rn <= 100
;

-- In order to recommend new films to the customer, it might be a good idea
-- to exclude the films that the customer rented in the past.

Drop table if exists schema.films_customer_hasnt_rented;
Create table schema.films_customer_hasnt_rented as 

Select 
	 a.customer_id
	,b.film
	,b.total_renting
From schema.customer_top_3_categories as a

Outer Join schema.top_100_films_per_category as b
Using (category_id)

Where a.film_id is null
;

— 10 film recommendations per customer

Insert into schema.film_recommendations 

Select a.customer_id, a.film_recommendations
From (
	Select      customer_id
		  , film
		  , total_renting
		  , row_number() over(partition by customer_id order by total_renting desc) as rn
	From schema.films_customer_hasnt_rented
       	) as a

Where rn <= 10
;
```

4. Create a table, customer_lifecycle, with a primary key of customer_id. Please include all the required DDL. This table is designed to provide a holistic view of a customers activity and should include:
- The revenue generated in the first 30 days of the customer's life-cycle, with day 0 being their first rental date.
- A value tier based on the first 30 day revenue.
- The name of the first film they rented.
- The name of the last film they rented.
- Last rental date.
- Avg. time between rentals.
- Total revenue.
- The top 3 favorite actors per customer.
- Any other interesting dimensions or facts you might want to include.

```
Create table schema.customer_lifecycle
(
customer_id		integer PRIMARY KEY,
revenue_first30days 	integer
value_tier            varchar
first_film
last_film
last_rental_date
average_time_bw_rentals 
total_revenue
top1_actor
top2_actor
top3_actor
last_updated          date not null default current_date
)
;

Drop table if exists schema.revenue_first30days;

Create table schema.revenue_first30days as 
With rentals as (

select 
	  d.customer_id
	, d.rental_id
	, d.inventory_id
	, d.rental_date
	, d.date_part
	, d.sum 
from(Select 
		  c.customer_id
		, c.rental_id
		, c.inventory_id
		, c.rental_date
		, c.date_part
		, sum(c.date_part) over(partition by c.customer_id order by c.rental_date asc rows between unbounded preceding and current row) 
	from(select 
			 b.customer_id
			,b.rental_id
			,b.inventory_id
			,b.rental_date
			,b.next_rental
			,date_part('day',b.rental_date::timestamp - b.next_rental::timestamp) 
		from(select 
				 customer_id
				,rental_id
				,inventory_id
				,rental_date
				,lag(rental_date,1) over(partition by customer_id) as next_rental 
			from rental 
			) as b
		) as c
	) as d 
where  d.sum <= 31 
	or d.sum is null
				)

select 
		 r.customer_id
		,sum(p.amount) as spend
from rentals as r
join payment as p
using (customer_id, rental_id)
group by r.customer_id
;

Drop table if exists schema.value_tier;

Create table schema.value_tier as

select 
	  a.customer_id
	, min(a.spend) as low
	, avg(a.spend) as med
	, max(a.spend) as high

from(select customer_id, sum(amount) as spend from payment group by customer_id) as a
;

With rental_data as (
		select 
		          rental_date
			, inventory_id
			, customer_id
			, row_number() over(partition by customer_id order by rental_date) as rn
			, count(*) over(partition by customer_id) as total_count 
		from rental
		    )
, first_film as (
			Select rd.customer_id, f.title
			From rental_data as rd
			join inventory as i
			using (inventory_id)
			join film as f
			using (film_id)
			where rn = 1
				)
, last_film as (
			Select rd.customer_id, f.title, rd.rental_date
			From rental_data as rd
			join inventory as i
			using (inventory_id)
			join film as f
			using (film_id)
			where rn = total_count
			)

, avg_time_bw_rentals as (

Select b.customer_id, avg(b.diff) as average_time_bw_rentals 
from (select 
	      a.customer_id
	    , a.rental_date
	    , a.next_rental
	    , date_part('day',b.rental_date::timestamp - b.next_rental::timestamp) as diff 
      from (select 
		  customer_id
		, rental_date
		, lag(rental_date,1) over(partition by customer_id) as next_rental 
	    from rental_data
	    ) as a
	) as b
Group by b.customer_id
                      )

, total_revenue as (
		  Select customer_id, sum(amount) as total_revenue
		  From payment
		  Group by customer_id
		  )

, favourite_actor as (
      Select      
          b.customer_id
          ,b.first_name || ‘  ‘ || last_name as full_name
          ,b.rn
      From (
            Select 
                a.customer_id
                ,a.actor_id
                ,a.favourite
                ,row_number() over(partition by a.customer_id order by a.favourite desc) as rn
            from(
                  Select
                      rd.customer_id
                      ,fa.actor_id
                      ,count(*) as favourite
                   From rental_data as rd
                   
                   Join inventory as i Using (inventory_id)
                   Join film as f Using (film_id)
                   Join film_actor as fa using (film_id)
                   group by 1,2
                ) as a
            ) as b
      Join actor as a Using (actor_id)
      Where rn <=3
	          	)

Insert into schema.customer_lifecycle

Select 
		 rf.customer_id
		,rf.spend as revenue_first30days

		,case when rf.spend < vt.low then ‘tier1’
			  when rf.spend >= vt.low and rf.spend < vt.med then ‘tier2’
			  when rf.spend >= vt.med and rf.spend < vt.high then ‘tier3’
			  when rf.spend >= vt.high then ‘tier4’
			else ’N/A’
		  end as value_tier
		
		,ff.title as first_film
		,lf.title as last_film
		,lf.rental_date as last_rental_date
		,at.average_time_bw_rentals 
		,tr.total_revenue
		,(case when fa.rn = 1 then fa.full_name end) as top1_actor
                ,(case when fa.rn = 2 then fa.full_name end) as top2_actor
                ,(case when fa.rn = 3 then fa.full_name end) as top3_actor
		,current_date + current_time as last_updated

From schema.revenue_first30days as rf
Join schema.value_tier as vt 
Using (customer_id)
Join first_film as ff
Using (customer_id)
Join last_film as lf
Using (customer_id)
Join avg_time_bw_rentals as at
Using (customer_id)
Join total_revenue as tr
Using (customer_id)
Join favourite_actor as fa
Using (customer_id)
```

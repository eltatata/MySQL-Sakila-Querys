# Taller Sakila

## statement 1

Insert a record into the 'film' table using dummy values, ensuring referential integrity with other tables.

```
insert into film (title, description, release_year, language_id, original_language_id, rental_duration, rental_rate, length, replacement_cost, rating, special_features, last_update)
values ('Film', 'This is a dummy film description.', 2024, 1, null, 5, 2.99, 120, 19.99, 'G', 'Trailers', now());
```
![Screenshot (217)](https://github.com/eltatata/MySQL-Sakila-Querys/assets/91573911/be890939-9cde-4923-829a-cd8f38053062)


## statement 2

Which films are longer than the average duration of films?

```
select title, length
from film
where length > (
    select avg(length)
    from film
);
```
![Screenshot (219)](https://github.com/eltatata/MySQL-Sakila-Querys/assets/91573911/0e78e8d0-ee23-4703-a8da-a2fa81c056c4)


## statement 3

Which films are currently rented at the store with store_id = 1?

```
select f.title, i.store_id
from film f
join inventory i on f.film_id = i.film_id
join rental r on i.inventory_id = r.inventory_id
where r.return_date is null
and i.store_id = 1;
```
![Screenshot (220)](https://github.com/eltatata/MySQL-Sakila-Querys/assets/91573911/77dc3930-c432-4695-9108-53bd710d9ad0)


## statement 4

Of the movies at the store with store_id = 1, which ones were rented for a longer duration than the average rental period?

```
select f.film_id, f.title, i.store_id
from film f
join inventory i on f.film_id = i.film_id
join rental r on i.inventory_id = r.inventory_id
where i.store_id = 1
and datediff(r.return_date, r.rental_date) > (
    select avg(datediff(return_date, rental_date))
    from rental
    where inventory_id in (
        select inventory_id
        from inventory
        where store_id = 1
    )
)
group by f.film_id, f.title;
```
![Screenshot (221)](https://github.com/eltatata/MySQL-Sakila-Querys/assets/91573911/4e171f5d-a787-4035-8890-5f11dd74f91d)


## statement 5

Which actors are part of the cast of 5 or fewer movies?

```
select a.actor_id, a.first_name, a.last_name
from actor a
join film_actor fa on a.actor_id = fa.actor_id
group by a.actor_id, a.first_name, a.last_name
having count(distinct fa.film_id) <= 5;
```
![Screenshot (222)](https://github.com/eltatata/MySQL-Sakila-Querys/assets/91573911/151d3d6b-16ea-4ac2-8553-be1a7ded1744)


## statement 6

Which last names do not repeat among different actors?

```
select distinct last_name from actor;
```
![Screenshot (223)](https://github.com/eltatata/MySQL-Sakila-Querys/assets/91573911/23b021d5-89bc-4a98-9381-28a7a7ac4982)


## statement 7

Create a view with the top 3 genres that generate the highest revenue. List them in descending order, considering the 'amount' field from the payment table for the calculation.

```
create view top_3_revenue_genres as
select fc.category_id, c.name as genre_name, sum(p.amount) as total_revenue
from payment p
join rental r on p.rental_id = r.rental_id
join inventory i on r.inventory_id = i.inventory_id
join film f on i.film_id = f.film_id
join film_category fc on f.film_id = fc.film_id
join category c on fc.category_id = c.category_id
group by fc.category_id, c.name
order by total_revenue desc
limit 3;
```
![Screenshot (225)](https://github.com/eltatata/MySQL-Sakila-Querys/assets/91573911/13cc9682-4fe9-4bb7-9c2e-de76a3eb131e)


## statement 8

Select the top two most-viewed movies in each city.

```
select 
    city_id,
    city,
    film_id,
    title,
    views
from (
    select 
        c.city_id,
        c.city,
        f.film_id,
        f.title,
        count(*) as views,
        row_number() over (partition by c.city_id order by count(*) desc) as row_num
    from city c
    join address a on c.city_id = a.city_id
    join customer cu on a.address_id = cu.address_id
    join rental r on cu.customer_id = r.customer_id
    join inventory i on r.inventory_id = i.inventory_id
    join film f on i.film_id = f.film_id
    group by c.city_id, c.city, f.film_id, f.title
) as numbered_views
where row_num <= 2
order by views desc;
```
![Screenshot (226)](https://github.com/eltatata/MySQL-Sakila-Querys/assets/91573911/e7e5f27a-6ff1-480c-8714-d6e1ec1ea03b)


## statement 9

Select the first name, last name, and email of all customers from the United States who have not made any film rentals in the last three months.

```
select c.customer_id, c.first_name, c.last_name, c.email
from customer c
join address a on c.address_id = a.address_id
join city ci on a.city_id = ci.city_id
join country co on ci.country_id = co.country_id
left join rental r on c.customer_id = r.customer_id
where co.country = 'united states' and (r.rental_id is null or r.rental_date < date_sub(now(), interval 3 month))
group by c.customer_id, c.first_name, c.last_name, c.email;
```
![Screenshot (227)](https://github.com/eltatata/MySQL-Sakila-Querys/assets/91573911/0a18ee6b-1595-43e0-b1ba-7d07671f80e0)


## statement 10

Select the top 3 customers from each store based on the number of rentals made. Utilize the Rank, Dense_Rank, and Row_Number functions, and create an additional boolean field indicating records where these three functions return the same value (0) and records where these three functions do not return the same value (1).

```
with ranked as (
    select
        c.customer_id,
        i.store_id,
        count(r.rental_id) as rental_count,
        rank() over (partition by i.store_id order by count(r.rental_id) desc) as rank_value,
        dense_rank() over (partition by i.store_id order by count(r.rental_id) desc) as dense_rank_value,
        row_number() over (partition by i.store_id order by count(r.rental_id) desc) as row_num
    from 
        customer c
    join 
        rental r on c.customer_id = r.customer_id
    join 
        inventory i on r.inventory_id = i.inventory_id
    group by 
        c.customer_id, 
        i.store_id
)
select 
    *,
    case
        when rank_value = dense_rank_value and dense_rank_value = row_num then 0
        else 1
    end as same_rank
from 
    ranked
where 
    rank_value <= 3;
```
![Screenshot (230)](https://github.com/eltatata/MySQL-Sakila-Querys/assets/91573911/49c11841-8d5d-4ad9-89a8-4f3ed41507c6)


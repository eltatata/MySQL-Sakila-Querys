# Taller Sakila

## statement 1

Insert a record into the 'film' table using dummy values, ensuring referential integrity with other tables.

```
insert into film (title, description, release_year, language_id, original_language_id, rental_duration, rental_rate, length, replacement_cost, rating, special_features, last_update)
values ('Film', 'This is a dummy film description.', 2024, 1, null, 5, 2.99, 120, 19.99, 'G', 'Trailers', now());
```

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

## statement 5

Which actors are part of the cast of 5 or fewer movies?

```
select a.actor_id, a.first_name, a.last_name
from actor a
join film_actor fa on a.actor_id = fa.actor_id
group by a.actor_id, a.first_name, a.last_name
having count(distinct fa.film_id) <= 5;
```

## statement 6

Which last names do not repeat among different actors?

```
select distinct last_name from actor;
```

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

## statement 8

Select the top two most-viewed movies in each city.

```
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

## statement 10

Select the top 3 customers from each store based on the number of rentals made. Utilize the Rank, Dense_Rank, and Row_Number functions, and create an additional boolean field indicating records where these three functions return the same value (0) and records where these three functions do not return the same value (1).

```
select
    store_id,
    customer_id,
    rentals,
    rank() over (partition by store_id order by rentals desc) as rank_val,
    dense_rank() over (partition by store_id order by rentals desc) as dense_rank_val,
    row_number() over (partition by store_id order by rentals desc) as row_num_val,
    case
        when rank() over (partition by store_id order by rentals desc) = dense_rank() over (partition by store_id order by rentals desc)
            and dense_rank() over (partition by store_id order by rentals desc) = row_number() over (partition by store_id order by rentals desc)
            then 0
        else 1
    end as same_value
from (
    select
        s.store_id,
        r.customer_id,
        count(r.rental_id) as rentals
    from
        rental r
    join
        inventory i on r.inventory_id = i.inventory_id
    join
        store s on i.store_id = s.store_id
    group by
        s.store_id,
        r.customer_id
) as rental_counts;
```
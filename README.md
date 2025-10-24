# Analytics-for-a-ride-sharing-company
# Ride Sharing SQL Analytics (PostgreSQL)

This project explores advanced SQL concepts using a ride-sharing database.  
It focuses on complex analytical queries involving clients, drivers, rides, ratings, and performance over time.

> ⚠️ **Disclaimer:**  
> This project is based on the *CSC343/CSC443 Ride Sharing Database Assignment (© 2022 Diane Horton and Marina Tawfik, University of Toronto)*.  
> The purpose of this repository is educational — to demonstrate SQL query design, data analysis, and problem-solving in PostgreSQL.

---

## 🧠 Project Overview
The goal is to write and test SQL queries that answer challenging analytical questions about ride-sharing data, such as:
- How often clients ride and how their habits change over time  
- Which drivers may violate rest-period bylaws  
- Whether driver training affects performance  
- Client spending trends, ratings comparisons, and month-over-month changes  

The project was built and tested using **PostgreSQL**.

---

## 🧩 Key Learning Objectives
- Designing **complex SQL queries** with joins, aggregates, and subqueries  
- Using **window functions**, **intervals**, and **views**  
- Handling **NULLs**, **CASE expressions**, and **COALESCE**  
- Working with **date/time arithmetic** and **PostgreSQL functions**

---

## 📊 Query Tasks Overview

| # | Query Title | Purpose |
|:-:|--------------|---------|
| 1 | **Months** | Count how many different months each client used the service |
| 2 | **Lure Them Back** | Identify high-spending clients whose activity declined |
| 3 | **Rest Bylaw** | Detect drivers violating the rest-period rule |
| 4 | **Do Drivers Improve?** | Compare early vs. late driver ratings (trained vs untrained) |
| 5 | **Bigger and Smaller Spenders** | Compare client spending against the monthly average |
| 6 | **Frequent Riders** | Find top and bottom riders by year |
| 7 | **Ratings Histogram** | Count how many 1–5 star ratings each driver received |
| 8 | **Scratching Backs?** | Compare how clients rate drivers vs. how drivers rate them |
| 9 | **Consistent Raters** | Find clients who rate every driver they’ve ridden with |
| 10 | **Rainmakers** | Compare drivers’ 2020 vs 2021 mileage and billings |

---

## ⚙️ Tools & Technologies
- **PostgreSQL 14+**
- **SQL Views, Joins, Aggregations, CASE, COALESCE**
- **Date/Time and Interval Functions**

---
**QUESTION 1** Months. For each client, report their client ID, email address, and the number of different months in which
they have had a ride. January 2021 and January 2022, for example, would count as two different months
```sql
CREATE TABLE q1(
    client_id INTEGER,
    email VARCHAR(30),
    months INTEGER
);
INSERT INTO q1
select c.client_id,c.email,count(distinct r.datetime::date) from client c 
left join request r on c.client_id=r.client_id left join dropoff d on
r.request_id=d.request_id
group by c.client_id,email;

```
**QUESTION 2** Lure them back. The company wants to lure back clients who formerly spent a lot on rides, but whose
ridership has been diminishing.
Find clients who had rides before 2020 costing at least $500 in total, have had between 1 and 10 rides (inclusive)
in 2020, and have had fewer rides in 2021 than in 2020
```sql
-- You must not change the next 2 lines or the table definition.
SET SEARCH_PATH TO uber, public;
DROP TABLE IF EXISTS q2 CASCADE;

CREATE TABLE q2(
    client_id INTEGER,
    name VARCHAR(41),
  	email VARCHAR(30),
  	billed FLOAT,
  	decline INTEGER
);

-- Do this for each of the views that define your intermediate steps.  
-- (But give them better names!) The IF EXISTS avoids generating an error 
-- the first time this file is imported.
DROP VIEW IF EXISTS intermediate_step CASCADE;

DROP VIEW IF EXISTS customer_analysis;

-- Define views for your intermediate steps here:

create  view customer_analysis as 
select r.client_id,count(r.*) filter ( where extract(year from r.datetime::date)::integer=2020 ) as count_20,
count(r.*) filter ( where extract(year from r.datetime::date)=2020 ) as count_21,
 sum(b.amount) filter ( where extract(year from r.datetime::date)::integer<2020 ) as bill
 from request r join billed b on r.request_id=b.request_id
 group by client_id;

-- Your query that answers the question goes below the "insert into" line:

INSERT INTO q2
select c.client_id,c.firstname,c.email,r.bill as billed,(count_20-count_21) as difference from client c
 join customer_analysis
 r on c.client_id=r.client_id 
where r.count_20>1 and r.count_20<=10 and r.count_21<r.count_20 and r.bill>500

);

```
**QUESTION 3** Rest bylaw. A break is the time elapsed between one drop-off by a driver and their next pick-up on that
same day (even if the pickup of the first ride was on a different day). The duration of a ride is the time elapsed
between pick-up and drop-off (If a ride has a pick-up time recorded but no drop-off time, it is incomplete and
does not have a duration). The total ride duration of a driver for a day is the sum of all ride durations of that
driver for rides whose pickup and drop-off are both recorded and were both on that day.
We will treat a day as going from midnight to midnight.
A city bylaw says that no driver may have three consecutive days where on each of these days they had a total
ride duration of 12 hours or more yet never had a break lasting more than 15 minutes. Keep in mind that
a driver could have a day with a single ride and nothing that counts as a break. They would by definition
violate the bylaw on that day if the ride was long enough.
Find every driver who broke the bylaw. Report their driver ID, the date on the first of the three days when
they broke the bylaw, their total ride duration summed over the three days, and their total break time summed
over the three days.
If a driver has broken the bylaw on more than one occasion, report one row for each. Don’t eliminate
overlapping three-day stretches. For example, if a driver had four long workdays in a row, they may have
broken the bylaw on the sequence of days d1, d2 and d3, and also on the sequence of days d2, d3, and d4.
There would be two rows in your table to describe this.
Your query should return an empty table if no driver ever broke the bylaw
## 🚀 How to Run
1. Create and populate the database using the provided schema and data files.
2. Run each SQL query script (`q1.sql`, `q2.sql`, …, `q10.sql`) in PostgreSQL.
3. Compare results with expected outputs.

```sql
-- Rest bylaw.

-- You must not change the next 2 lines or the table definition.
SET SEARCH_PATH TO uber, public;
DROP TABLE IF EXISTS q3 CASCADE;

CREATE TABLE q3(
    driver_id INTEGER,
    start DATE,
    driving INTERVAL,
    breaks INTERVAL
);

-- Do this for each of the views that define your intermediate steps.  
-- (But give them better names!) The IF EXISTS avoids generating an error 
-- the first time this file is imported.
DROP VIEW IF EXISTS intermediate_step CASCADE;

DROP VIEW IF EXISTS driver_analysis;

-- Define views for your intermediate steps here:

create view driver_analysis as 
select driver_id,sum(driving) as driving,datetime,sum(breaks) as breaks from (select s.driver_id,s.driving,s.datetime::date,
SUM(CASE WHEN c.breaks is not null THEN c.breaks ELSE ((s.drop_time::date + interval '1' day)::timestamp without time zone-s.drop_time) END) as breaks
from (select dr.driver_id,sum(d.datetime-p.datetime)  as driving,p.datetime,d.datetime as drop_time from driver dr join clockedin c on dr.driver_id=c.driver_id
join dispatch f on f.shift_id=c.shift_id join request r on f.request_id=r.request_id
join pickup p on r.request_id=p.request_id  join
dropoff d on p.request_id=d.request_id and p.datetime::date=d.datetime::date  where d.datetime is not null 
group by dr.driver_id,p.datetime,drop_time) s
join (
select a.driver_id,sum(a.datetime-b.datetime)  as breaks,a.datetime  from (
select p.*,dr.driver_id,rank() over (partition by dr.driver_id,p.datetime::date order by p.datetime) from driver dr join clockedin c on dr.driver_id=c.driver_id
join dispatch f on f.shift_id=c.shift_id join request r on f.request_id=r.request_id
join pickup p on r.request_id=p.request_id) a 
left join 
 (
select p.*,dr.driver_id,rank() over (partition by dr.driver_id,p.datetime::date order by p.datetime) from driver dr join clockedin c on dr.driver_id=c.driver_id
join dispatch f on f.shift_id=c.shift_id join request r on f.request_id=r.request_id
join dropoff p on r.request_id=p.request_id where p.datetime is not null) b on a.datetime::date=b.datetime::date and a.rank=b.rank+1 and a.driver_id=b.driver_id 
group by a.driver_id,a.datetime)c on s.driver_id=c.driver_id and s.datetime=c.datetime
group by s.DROP_TIME,s.driver_id,s.driving,s.datetime::date) l
group by driver_id,datetime;

-- Your query that answers the question goes below the "insert into" line:

INSERT INTO q3
select c.driver_id,c.datetime as start,c.driving,c.breaks from driver_analysis a join driver_analysis b 
on a.datetime + interval '1' day=b.datetime  and a.driver_id=b.driver_id join driver_analysis c
on b.datetime + interval '1' day=c.datetime  and b.driver_id=c.driver_id
where a.driving>='12:00:00'::interval and a.breaks<'00:15:00'::interval and 
b.driving>='12:00:00'::interval and b.breaks<'00:15:00'::interval and
c.driving>='12:00:00'::interval and c.breaks<'00:15:00'::interval;

```


---

## ✨ What I Learned
- Writing production-grade analytical SQL queries.
- Managing complex time-based and conditional logic.
- Debugging large relational queries efficiently.
- Applying best practices for PostgreSQL query design and readability.

---

## 📚 Credits
This work is based on:
> Horton, D. & Tawfik, M. (2022). *CSC343/CSC443 Ride Sharing Database Assignment*, University of Toronto.  
Used for educational and portfolio purposes under academic fair use.

---

## 👤 Author
**Farrukh Saeed**  
📍 Mississauga, ON  

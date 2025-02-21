# 🍊 Case Study #8 - Fresh Segments
## D. Index Analysis
The `index_value` is a measure which can be used to reverse calculate the average composition for Fresh Segments’ clients.

Average composition can be calculated by dividing the `composition` column by the `index_value` column rounded to 2 decimal places.

### 1. What is the top 10 interests by the average composition for each month?
```TSQL
with avg_composition as (select  map.id as interest_id, interest_name, month_year,
round(composition / index_value,2) as avg_composition,
dense_rank() over(partition by month_year order by composition / index_value desc) as rn
FROM fresh_segments.interest_metrics metrics
JOIN fresh_segments.interest_map map 
ON metrics.interest_id = map.id
WHERE metrics.month_year IS NOT NULL)
select * from avg_composition
where rn <=10;

```
140 rows for 14 months in total. The first 10 rows:

| interest_id | interest_name                 | month_year | avg_composition | rnk  |
|-------------|-------------------------------|------------|-----------------|------|
| 6324        | Las Vegas Trip Planners       | 2018-07-01 | 7.36            | 1    |
| 6284        | Gym Equipment Owners          | 2018-07-01 | 6.94            | 2    |
| 4898        | Cosmetics and Beauty Shoppers | 2018-07-01 | 6.78            | 3    |
| 77          | Luxury Retail Shoppers        | 2018-07-01 | 6.61            | 4    |
| 39          | Furniture Shoppers            | 2018-07-01 | 6.51            | 5    |
| 18619       | Asian Food Enthusiasts        | 2018-07-01 | 6.1             | 6    |
| 6208        | Recently Retired Individuals  | 2018-07-01 | 5.72            | 7    |
| 21060       | Family Adventures Travelers   | 2018-07-01 | 4.85            | 8    |
| 21057       | Work Comes First Travelers    | 2018-07-01 | 4.8             | 9    |
| 82          | HDTV Researchers              | 2018-07-01 | 4.71            | 10   |

---
### 2. For all of these top 10 interests - which interest appears the most often?
```TSQL
WITH avg_composition AS (
    SELECT  
        map.id AS interest_id, 
        interest_name, 
        month_year,
        ROUND(composition / index_value, 2) AS avg_composition,
        DENSE_RANK() OVER (PARTITION BY month_year ORDER BY composition / index_value DESC) AS rn
    FROM 
        fresh_segments.interest_metrics metrics
    JOIN 
        fresh_segments.interest_map map ON metrics.interest_id = map.id
    WHERE 
        metrics.month_year IS NOT NULL
),
occu_freq AS (
    SELECT  
        interest_id, 
        interest_name,
        COUNT(1) AS occu_freq
    FROM 
        avg_composition
    WHERE 
        rn <= 10
    GROUP BY 
        interest_id, interest_name
)
SELECT *
FROM occu_freq 
WHERE occu_freq IN (SELECT MAX(occu_freq) FROM occu_freq);

```
| interest_id | interest_name            | occu_freq  |
|-------------|--------------------------|------------|
| 7541        | Alabama Trip Planners    | 10         |
| 5969        | Luxury Bedding Shoppers  | 10         |
| 6065        | Solar Energy Researchers | 10         |

---
### 3. What is the average of the average composition for the top 10 interests for each month?
```TSQL
with avg_composition as (select  map.id as interest_id, interest_name, month_year,
round(composition / index_value,2) as avg_composition,
dense_rank() over(partition by month_year order by composition / index_value desc) as rn
 FROM fresh_segments.interest_metrics metrics
  JOIN fresh_segments.interest_map map 
    ON metrics.interest_id = map.id
  WHERE metrics.month_year IS NOT NULL)
select month_year, avg(avg_composition) as avg_composition_by_month_year
from avg_composition
where rn <=10
group by month_year;
```
| month_year | avg_composition_by_month_year  |
|------------|--------------------------------|
| 2018-07-01 | 6.038                          |
| 2018-08-01 | 5.945                          |
| 2018-09-01 | 6.895                          |
| 2018-10-01 | 7.066                          |
| 2018-11-01 | 6.623                          |
| 2018-12-01 | 6.652                          |
| 2019-01-01 | 6.399                          |
| 2019-02-01 | 6.579                          |
| 2019-03-01 | 6.168                          |
| 2019-04-01 | 5.75                           |
| 2019-05-01 | 3.537                          |
| 2019-06-01 | 2.427                          |
| 2019-07-01 | 2.765                          |
| 2019-08-01 | 2.631                          |

---
### 4. What is the 3 month rolling average of the max average composition value from September 2018 to August 2019 and include the previous top ranking interests in the same output shown below.

* Create a CTE `avg_compositions` to calculate the average composition value and the maximum of the average composition value for each `month_year`. In this case, I use the window function `MAX() OVER` to keep the corresponding `interest_id` and `month_year`.
* From the CTE `avg_compositions` above, create a new CTE `max_avg_compositions` to filter `interest_id` that has the average composition value equal to the maximum of the average composition value for each `month_year`.
* Next, from the CTE `max_avg_compositions`, JOIN with the table `interest_map` to take out relevant `interest_name` and create 2 last columns.
* To create the 3-month moving averages column, use the window function `AVG() OVER` with  `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`
* To create the last 2 column, use `LAG() OVER`. Remember to specify the second argument in `LAG()` for `2_month_ago` column. Then, cast those moving values to string to concatenate with the corresponding `interest_name`. 
* Finally, filter rows that have `month_year` from September 2018 to August 2019.


```TSQL
WITH avg_composition AS (
    SELECT  
        map.id AS interest_id, 
        interest_name, 
        month_year,
        ROUND(composition / index_value, 2) AS avg_composition,
        ROUND(MAX(composition / index_value) OVER (PARTITION BY month_year), 2) AS max_avg_composition
    FROM 
        fresh_segments.interest_metrics metrics
    JOIN 
        fresh_segments.interest_map map ON metrics.interest_id = map.id
    WHERE 
        metrics.month_year IS NOT NULL
),
max_avg_composition AS (
    SELECT * 
    FROM avg_composition
    WHERE avg_composition = max_avg_composition
),
moving_avg_compositions AS (
    SELECT 
        month_year, 
        interest_name, 
        max_avg_composition,
        ROUND(AVG(max_avg_composition) OVER (ORDER BY month_year ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS '3_month_rolling_avg',
        LAG(interest_name, 2, 2) OVER (ORDER BY month_year) + ':' + CAST(LAG(max_avg_composition) OVER (ORDER BY month_year) AS VARCHAR(5)) AS '2_month_ago',
        LAG(interest_name) OVER (ORDER BY month_year) + ':' + CAST(LAG(max_avg_composition) OVER (ORDER BY month_year) AS VARCHAR(5)) AS '1_month_ago'
    FROM 
        max_avg_composition mac
)
SELECT *
FROM moving_avg_compositions
WHERE month_year BETWEEN '2018-09-01' AND '2019-08-01';
```

| month_year | interest_name                       | max_avg_composition | 3_month_rolloing_avg | 2_month_ago                           | 1_month_ago                           |
|------------|-------------------------------------|----------------------|-----------------------|----------------------------------------|----------------------------------------|
| 2018-09-01 | Work Comes First Travelers          | 8.26                 | 7.61                  | Las Vegas Trip Planners:7.21          | Las Vegas Trip Planners:7.21          |
| 2018-10-01 | Work Comes First Travelers          | 9.14                 | 8.20                  | Las Vegas Trip Planners:8.26          | Work Comes First Travelers:8.26       |
| 2018-11-01 | Work Comes First Travelers          | 8.28                 | 8.56                  | Work Comes First Travelers:9.14       | Work Comes First Travelers:9.14       |
| 2018-12-01 | Work Comes First Travelers          | 8.31                 | 8.58                  | Work Comes First Travelers:8.28       | Work Comes First Travelers:8.28       |
| 2019-01-01 | Work Comes First Travelers          | 7.66                 | 8.08                  | Work Comes First Travelers:8.31       | Work Comes First Travelers:8.31       |
| 2019-02-01 | Work Comes First Travelers          | 7.66                 | 7.88                  | Work Comes First Travelers:7.66       | Work Comes First Travelers:7.66       |
| 2019-03-01 | Alabama Trip Planners               | 6.54                 | 7.29                  | Work Comes First Travelers:7.66       | Work Comes First Travelers:7.66       |
| 2019-04-01 | Solar Energy Researchers            | 6.28                 | 6.83                  | Work Comes First Travelers:6.54       | Alabama Trip Planners:6.54            |
| 2019-05-01 | Readers of Honduran Content        | 4.41                 | 5.74                  | Alabama Trip Planners:6.28            | Solar Energy Researchers:6.28          |
| 2019-06-01 | Las Vegas Trip Planners            | 2.77                 | 4.49                  | Solar Energy Researchers:4.41          | Readers of Honduran Content:4.41      |
| 2019-07-01 | Las Vegas Trip Planners            | 2.82                 | 3.33                  | Readers of Honduran Content:2.77      | Las Vegas Trip Planners:2.77          |
| 2019-08-01 | Cosmetics and Beauty Shoppers      | 2.73                 | 2.77                  | Las Vegas Trip Planners:2.82          | Las Vegas Trip Planners:2.82          |

---

### 5. Provide a possible reason why the max average composition might change from month to month? Could it signal something is not quite right with the overall business model for Fresh Segments?

The max average composition decreased overtime because top interests were mostly travel-related services, which were in high seasonal demands for some months throughout a year. Customers wanted to go on a trip during the last and first 3 months of a year. You can see `max_index_composition` were high from September 2018 to March 2019. 

This also means that Fresh Segments's business heavily relied on travel-related services. Other products and services didn't receive much interest from customers.

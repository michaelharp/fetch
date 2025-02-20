# Fetch Technical Assessment - James Michael Harp
This repo hosts my technical assessment responses for the Fetch Data Analyst role. Below you will find my responses with short code snippets included. Please note that all answers are written in PostgreSQL, however, I am most proficient and comfortable with Snowflake. Thanks for reading!

## Data Quality Issues - Products

Starting with the `products` table, I found the following issues with the data quality:
- The NULLs in this table are being represented by empty strings instead of true NULL values. These should be replaced with true NULLs. Additionally, there is no dedicated primary key for this table, which is especially relevant given that there are empty values in the barcode column.
```
select count(*) 
from public.products p
where barcode = ''
and barcode is not null;
```
- PLACEHOLDER MANUFACTURER, NEEDS REVIEW category, and BRAND NOT KNOWN all feel like bad practice to me. These values should be replaced with NULL.
- There seem to be some non-standard UPC's in the barcode column, i.e. not UPC-A, UPC-E, EAN, PLU, or GTIN-14. Specifically, some 6-digit, 7-digit, 9-digit, and 10-digit UPC's.
```
select length(barcode), count(*)
from public.products p
group by 1
order by length(barcode);
```
- The assessment page states that barcode should be represented as an integer, which isn't possible since UPC's often lead with a 0.

  ## Data Quality Issues - Transactions

  I found the following issues with the data quality of the `transactions` table:
  - There appear to be a lot of duplicate rows where either final_quantity is "zero" or final_sale is a single whitespace. Perhaps this is due to an issue with the OCR model output? These rows should be deleted. Further, all empty/whitespace strings should be converted to NULLs. Unless these rows are deleted, the `final_quantity` and `final_sale` columns cannot be properly converted to `numeric` datatype for analysis. 
```
select * 
from transactions
where final_quantity = 'zero';
```
  - The rows in which barcode is an empty string make sense conceptually if this is OCR output from a receipt that might not contain UPC data, but the absence of product names in this data set means there's no way to connect some of these rows back to actual product entities for analysis.
```
select count(*)
from transactions
where barcode = ' ';
```
  - Unfortunately, the OCR model seems to really be struggling on some of these store names (i.e. /MART, B, 1AINTING CUSVAL BISTRO, A CME DURP SUPERAK, etc.) It may be outside of the scope of this assignment, but it seems it would be useful for the OCR model to have a pre-set list of available retailer names to choose from through string similarity matching (trigram matching, jarowinkler similarity, etc).
  - `final_sale` is a bit confusing as a column. It's unclear to me whether this is the total price * quantity of the given receipt line or if this is simply the price per unit. For the purposes of this exercise, I will assume that this is the final price of the receipt line (unit price * quantity).
  - Again, the assessment page states that barcode should be represented as an integer in this table, which isn't possible since UPC's often lead with a 0.

## Data Quality Issues - Users
I found the following issues with the data quality of the `users` table:
- I noticed that user_id `5f31fc048fa1e914d38d6952` has a birth_date > created_date for the account, which isn't possible. There need to be greater guardrails placed on the frontend for birth date selection.
- There are very few user_ids in the `transactions` table that are actually present here. It seems that this table must be missing a lot of data. 
- There's really no need for created_date and birth_date to be represented as timestamps. These should just be date values. 
- There are again several instances in which empty strings are present in the data set and should be replaced by NULLs. 
- In the gender column, "unknown" should also be replaced by NULLs. "Non-Binary" and "non_binary" should be consolidated into a single value, as well as "Prefer not to say" with "prefer_not_to_say", and "not_listed" with "My gender isn't listed". I imagine there would need to be some sort of collaboration with the front-end team to ensure standardization in the dropdown for gender selection.

# SQL Assessment
1. What are the top 5 brands by receipts scanned among users 21 and over?
- Answer: Unfortunately, there seems to be very little overlap between user_id's in the `transactions` table and ids in the `users` table. However, if taking the distribution of ages in the `users` table at face value and assuming a similar distribution exists for users in the `transactions` table, then it can be assumed that 89.9% of users scanning receipts are over the age of 21.
  - Furthermore, given how much of a gap there is in receipt scan count between the top five brands and the remaining brands in the general `transactions` set, it can be assumed with some degree of confidence that the top five brands for users over the age of 21 would be the same as the top five for the entire user set, assuming that the estimated 11.1% of users under the age of 21 don't deviate in order behavior to an extreme amount from the over 21 population. 
    - Unfortunately, since there are no users in the `transactions` table that can be proven to be under the age of 21, we cannot make any confident assumptions one way or another about the order behavior of this user set.
- If assuming somewhat similar order behavior from the under-21 set as the over 21-set, the top 5 brands by receipts scanned among users 21 and over are as follows:
  1. COCA-COLA
  2. GREAT VALUE
  3. PEPSI
  4. EQUATE
  5. LAY'S

```
select brand, count(distinct receipt_id) as distinct_receipts_count
from transactions t
join products p on p.barcode = t.barcode
where brand is not null
group by 1
order by 2 desc
```

2. What are the top 5 brands by sales among users that have had their account for at least six months?
- Answer: Given the lack of overlap between the `users` table and `transactions` table mentioned above, similar extrapolation must take place to provide an answer here. If the user base in the `transactions` table is assumed to have the same distribution of "created_date" as the user base present in the `users` table, then those with accounts older than 6 months would represent 98.3% of scanned transactions.
- This being the case, it can be assumed that the top 5 brands by sales among users that have had their account for >= 6 months are as follows:
  1. PEPSI
  2. COCA-COLA
  3. EQUATE
  4. GREAT VALUE
  5. HERSHEY'S
```
select brand, sum(t.final_quantity*t.final_sale) as sum_gmv
from transactions t
join products p on p.barcode = t.barcode
where brand is not null
group by 1
order by 2 desc;
```

3. What is the percentage of sales in the Health & Wellness category by generation?
- Again, given the little overlap between the users and transactions table, we can only assume the below to be true if the small subset of users that _do_ overlap between these two tables are representative of the entire population of active users in the transactions set. 
- If assuming the above, the percentage of sales in the Health & Wellness category by generation breaks down as follows:
  1. Boomers: 44.34%
  2. Gen X: 24.49%
  3. Millenials: 31.18%
```
with health_wellness_gmv_by_generation as (
	select 
		case
			when date_part('year', age(current_date, birth_date)) > 98
				then 'WWII'
			when date_part('year', age(current_date, birth_date)) between 80 and 97
				then 'Post-War'		
			when date_part('year', age(current_date, birth_date)) between 61 and 79
				then 'Boomers'
			when date_part('year', age(current_date, birth_date)) between 45 and 60
				then 'Gen X'
			when date_part('year', age(current_date, birth_date)) between 29 and 44
				then 'Millenials'
			when date_part('year', age(current_date, birth_date)) between 13 and 28
				then 'Gen Z'
			else 'Gen Alpha'
		end as generation
	, category_1, sum(t.final_sale) as sum_gmv
	from transactions t
	join products p on p.barcode = t.barcode
	join users u on u.id = t.user_id
	where category_1 = 'Health & Wellness'
	group by 1,2)
	
select generation, sum_gmv, round(100*sum_gmv/(select sum(sum_gmv) from health_wellness_gmv_by_generation),2) as percent_gmv
from health_wellness_gmv_by_generation;     
```
4. Who are Fetch's power users?
- If only analyzing users that can be connected to the transactions table, then I would define a "power user" as someone with 3 or more transactions logged in Fetch, as 3 is the highest number of logged transactions per user in this set.
- Given the above, Fetch's power users are overwhelmingly female, an average age of 44 years old, and have had an account for a little over 3 years on average. 
```
  with power_users as (
	select user_id, count(distinct receipt_id) as transaction_count
		from transactions t
		join users u on u.id = t.user_id
		group by 1
		having count(distinct receipt_id) >= 3)
	
	select mode() within group (order by gender) as most_common_gender, avg(date_part('year',age(current_date, birth_date))) as average_age, avg(age(current_date, created_date)) as average_time_since_account_creation
	from users 
	where id in (select user_id from power_users);
 ```
5. Which is the leading brand in the Dips & Salsa category?
- Answer: The leading brand in the Dips & Salsa category for 2024 is Tostitos with 36 measured receipt scans and $260.99 in GMV.
```
select date_part('year', purchase_date) as year, brand, count(distinct t.receipt_id) as receipt_scans, sum(final_sale) as sum_gmv
from transactions t
join products p on p.barcode = t.barcode
where category_2 = 'Dips & Salsa'
group by 1,2
order by 4 desc;
```
6. At what percent has Fetch grown year over year?
- Answer: Since the supplied transaction table only includes 2024 transactions, this answer can only be answered as a measure of growth in the size of the user base in `users`.
- Fetch has managed to increase its user base by 50.4% YoY on average since 2014, although the rate of increase has been falling YoY from 2017 onward. 
```
with user_data as (
  select
    date_part('year', created_date) as year,
    count(*)
  from users
  group by 1
),

yearly_sums as (

select
  year,
  sum(count) over (order by year asc rows between unbounded preceding and current row) as total_user_base
  from user_data),
  
 yoy as (
  
  select *,
 (total_user_base - lag(total_user_base) over (order by year))/total_user_base as year_over_year_increase
  from yearly_sums y)  
  
  select * --avg(year_over_year_increase) as avg_user_base_growth_per_year  
  from yoy;
```

# Example Communication

Hi there!

I was recently tasked with some analysis on the `users`, `transactions`, and `products` tables, and I wanted to share my findings with you. 

A few key takeaways:
- We seem to have a real problem with whitespace and empty strings in our tables where values should be NULL, which could really lead to inaccurate analysis if someone isn't careful.
- There seems to be an issue with the `users` table-- perhaps it's somehow been truncated? Very few of the user_id's in the `transactions` table are represented in the `users` table, which makes certain types of demographic analysis impossible or unreliable.
- Unfortunately, the OCR model seems to really be struggling on some of the store names in `transactions` (i.e. /MART, B, 1AINTING CUSVAL BISTRO, A CME DURP SUPERAK, etc.) Perhaps it would be useful for the model to have a pre-set list of available retailer names to choose from through string similarity matching (trigram matching, jarowinkler similarity, etc)?

Additionally, I did notice an interesting trend in the `users` table where the overwhelming majority of inputted birth_date values are set to `1970-01-01 00:00:00`. Is this the default birth_date set in the UI when users are asked to provide their DOB? Between these, many having no DOB provided, and many users with inputted ages of over 100 or under 10, I suspect that this data set might not be particularly reliable. Perhaps we need some sort of more robust verification process? 

I'm very interested to hear your thoughts! Let me know if I can be of any assistance with tightening up our OCR matching for store names or assisting with anomaly detection on the user-inputted demographic data. Hope this is helpful. 

Thanks!
Michael Harp





# 🧹 Data Loading & Cleansing: `crm_orders` (Bronze ➝ Silver Layer)


> This script performs data quality checks and cleansing operations on the `silver.crm_orders`.  
> The goal is to ensure that all records are complete, clean, and logically consistent.

---
## Initial DDL Script to load `crm_orders` from broze layer (no structure changes)


```sql
IF OBJECT_ID('silver.crm_orders', 'U') IS NOT NULL
	DROP TABLE silver.crm_orders;
GO

CREATE TABLE silver.crm_orders (
	order_id NVARCHAR(50),
	customer_id NVARCHAR(50),
    	order_status NVARCHAR(50),
	order_purchase_timestamp DATETIME,
	order_approved_at DATETIME,
	order_delivered_carrier_date DATETIME,
	order_delivered_customer_date DATETIME,
	order_estimated_delivery_date DATETIME,
	dwh_create_date DATETIME2 DEFAULT GETDATE()
    );
GO


INSERT INTO silver.crm_orders(
	order_id , customer_id, order_status,
	order_purchase_timestamp,order_approved_at,
	order_delivered_carrier_date,order_delivered_customer_date,
	order_estimated_delivery_date
	)

SELECT order_id ,
	customer_id,
	order_status,
	order_purchase_timestamp,
	order_approved_at,
	order_delivered_carrier_date,
	order_delivered_customer_date,
        order_estimated_delivery_date
FROM bronze.crm_orders
```
| order_id                             | customer_id                         | order_status | order_purchase_timestamp | order_approved_at        | order_delivered_carrier_date | order_delivered_customer_date | order_estimated_delivery_date |
|-------------------------------------|-------------------------------------|--------------|---------------------------|---------------------------|-------------------------------|-------------------------------|-------------------------------|
| 06a6627d9cc91a04e9d146bf65fee0a2    | 61449fa1b8b8998c9c3f3a7f0ae954ef    | delivered    | 2017-07-16 10:27:45.000   | 2017-07-16 10:35:14.000   | 2017-07-18 12:34:04.000       | 2017-07-21 19:59:36.000       | 2017-08-04 00:00:00.000       |
| 15bed8e2fec7fdbadb186b57c46c92f2    | f3f0e613e0bdb9c7cee75504f0f90679    | processing   | 2017-09-03 14:22:03.000   | 2017-09-03 14:30:09.000   | NULL                          | NULL                          | 2017-10-03 00:00:00.000       |
| 6942b8da583c2f9957e990d028607019    | 52006a9383bf149a4fb24226b173106f    | shipped      | 2018-01-10 11:33:07.000   | 2018-01-11 02:32:30.000   | 2018-01-11 19:39:23.000       | NULL                          | 2018-02-07 00:00:00.000       |

## ✅ Checks Summary

| **Type**           | **Category**           | **Check Description**                                                                |
|--------------------|------------------------|--------------------------------------------------------------------------------------|
| **DATA INTEGRITY** | Check Lenght           | Ensure `order_id` and `customer_id` have 32 alphanumeric characters                  |
|                    | Check Duplicates       | Check for duplicates on `order_id`                                                   |
|                    | Distinct Values        | Validate distinct status values on `order_status`                                    |
|**DATA CONSISTENCY**| Status-Date Consistency| Ensure date fields align with expected `order_status` behavior (created → approved → processing → invoiced → shipped → delivered)|
|                    | Temporal Logic         | Validate correct sequence between status timestamps  |
| **DATA VALIDATION**| Date Range Validation  | Check min/max ranges for all date fields                                             |

---

## `order_id` cleaning
### 1) Check order_id lenght
```sql
SELECT LEN(order_id) AS lenght_order_id,
	   COUNT(*) AS counting
FROM silver.crm_orders
GROUP BY LEN(order_id)
ORDER BY LEN(order_id) DESC
-- All the order_id has 32 characters
```

### 2) Check duplicates
```sql
SELECT order_id,
	   COUNT(*) AS counting
FROM silver.crm_orders
GROUP BY order_id
HAVING COUNT(*)>1
-- NO duplicates detected
```
---


## `customer_id` cleaning
### 1) Check customer_id lenght
```sql
SELECT LEN(customer_id) AS lenght_customer_id,
	   COUNT(*)
FROM silver.crm_orders
GROUP BY LEN(customer_id)
ORDER BY LEN(customer_id) DESC
-- All the customer_id has 32 characters
```
---


## `order_status` cleaning
### 1) Analyze distinct values
```sql
SELECT DISTINCT order_status
FROM silver.crm_orders
/*No anomalies values.
  We have the following results:

| order_status   |
|----------------|
| created        |
| approved       |
| processing     |
| invoiced       |
| shipped        |
| delivered      |
| unavailable    |
| canceled       |
```


### 2) Verify correcteness of order_status and dates
Verify when order is only **CREATED** `order_approved_at` , `order_delivered_carrier_date` and `order_delivered_customer_date` should be NULL
```sql
SELECT *
FROM silver.crm_orders
WHERE order_status = 'created'
  AND ( order_approved_at IS NOT NULL OR
	order_delivered_carrier_date IS NOT NULL OR 
	order_delivered_customer_date IS NOT NULL );
	-- No anomalies
```

Verify when order is **PROCESSING, APPROVED OR INVOICED** `order_delivered_carrier_date` and `order_delivered_customer_date` should be NULL
```sql
SELECT *
FROM silver.crm_orders
WHERE order_status IN ( 'processing','approved', 'invoiced')
  AND (order_delivered_carrier_date IS NOT NULL OR 
       order_delivered_customer_date IS NOT NULL);
	-- No anomalies
```

Verify when order is **SHIPPED** `order_delivered_customer_date` should be NULL
```sql
SELECT *
FROM silver.crm_orders
WHERE order_status = 'shipped'
  AND order_delivered_customer_date IS NOT NULL;
	-- No anomalies
```

Verify when order is **DELIVERED** `order_delivered_customer_date` should be NOT NULL
```sql
SELECT *
FROM silver.crm_orders
WHERE order_status = 'delivered'
  AND order_delivered_customer_date IS NULL;
	-- 8 Anomalies detected, in this case the status should be only shipped

| Order ID | Cust ID | Status    | Purchase Timestamp  | Approved At         | Carrier Date        | Cust Date | Est. Delivery|
|----------|---------|-----------|---------------------|---------------------|---------------------|-----------|--------------|
| 2d1e2d.. | eca6d.. | delivered | 2017-11-28 17:44:07 | 2017-11-28 17:56:40 | 2017-11-30 18:12:23 | NULL      | 2017-12-18   |
| f5d788.. | 5e028.. | delivered | 2018-06-20 06:58:43 | 2018-06-20 07:19:05 | 2018-06-25 08:05:00 | NULL      | 2018-07-16   |
| 2ebd15.. | 29f40.. | delivered | 2018-07-01 17:05:11 | 2018-07-01 17:15:12 | 2018-07-03 13:57:00 | NULL      | 2018-07-30   |
| e69f17.. | cfd0c.. | delivered | 2018-07-01 22:05:55 | 2018-07-01 22:15:14 | 2018-07-03 13:57:00 | NULL      | 2018-07-30   |

-- UPDATE statement: fix `order_status` to 'shipped' when `order_delivered_customer_date` IS NULL
UPDATE silver.crm_orders
SET order_status = 'shipped'
WHERE order_status = 'delivered' AND order_delivered_customer_date IS NULL
```


Verify when order is **UNAVAILABLE** `order_delivered_carrier_date` & `order_delivered_customer_date` should be  NULL
```sql
SELECT *
FROM silver.crm_orders
WHERE order_status = 'unavailable' AND order_delivered_carrier_date IS NOT NULL AND order_delivered_customer_date IS NOT NULL
	-- No anomalies
```

Verify when order is **CANCELED** `order_delivered_customer_date` should be  NULL
```sql
SELECT *
FROM silver.crm_orders
WHERE order_status = 'canceled' AND order_delivered_customer_date IS NOT NULL
-- we have 6 anomalies , so in this case the status should be delivered

| Order ID | Cust ID   | Status   | Purchase Timestamp  | Approved At | Carrier Date        | Cust Date           | Est. Delivery |
|----------------------|--------------------------------|-------------|---------------------|---------------------|---------------|
| 26f224.. | 5ds6a6d.. | canceled | 2017-11-28 17:44:07 | 2017-11-28  | 2017-11-30 18:12:23 | 2017-12-03 11:32:23 | 2017-12-18    |
| f5d2b7.. | df89028.. | canceled | 2018-06-20 06:58:43 | 2018-06-20  | 2018-06-25 08:05:00 | 2018-06-28 08:05:00 | 2018-07-16    |
| fc4f15.. | 51sd540.. | canceled | 2018-07-01 17:05:11 | 2018-07-01  | 2018-07-03 13:57:00 | 2018-07-14 09:37:00 | 2018-07-30    |
| 52s717.. | dsfa40c.. | canceled | 2018-07-01 22:05:55 | 2018-07-01  | 2018-07-03 13:57:00 | 2018-07-13 11:58:00 | 2018-07-30    |

-- UPDATE statement: fix `order_status` to 'delivered' when `order_delivered_customer_date` IS NOT NULL
UPDATE silver.crm_orders
SET order_status = 'delivered'
WHERE order_status = 'canceled' AND order_delivered_customer_date IS NOT NULL
```
---


## `order_purchase_timestamp`, `order_approved_at`, `order_delivered_carrier_date`,`order_delivered_customer_date`,`order_estimated_delivery_date` cleaning

### 1) Check dates range 
```sql
SELECT 
	   MIN(order_purchase_timestamp) AS min_date_purchase,
	   MAX(order_purchase_timestamp) AS max_date_purchase,
	   MIN(order_approved_at) AS min_date_approved_purchase,
	   MAX(order_approved_at) AS max_date_approved_purchase,
	   MIN(order_delivered_carrier_date) AS min_date_delivered_carrier,
	   MAX(order_delivered_carrier_date) AS max_date_delivered_carrier,
	   MIN(order_delivered_customer_date) AS min_date_delivered_customer,
	   MAX(order_delivered_customer_date) AS max_date_delivered_customer,
	   MIN(order_estimated_delivery_date) AS min_date_estimated_delivery,
	   MAX(order_estimated_delivery_date) AS max_date_estimated_delivery
FROM silver.crm_orders
/*
Date order purchase --> from 04/09/2016 to 17/10/2018
Date approved purchase --> from 15/09/2016 to 03/09/2018
Date delivered carrier --> from 08/10/2016 to 11/09/2018
Date delivered customer --> from 11/10/2016 to 17/10/2018
date estimatd delivery --> from 30/09/2016 to 12/11/2018
*/
```

### 2) Verify correctness of date sequence
```sql
SELECT order_id,
       customer_id,
       order_status,
       order_purchase_timestamp,
       order_approved_at,
       order_delivered_carrier_date,
       order_delivered_customer_date,
       order_estimated_delivery_date,
       CASE WHEN order_approved_at < order_purchase_timestamp THEN 1 ELSE 0 END AS approved_before_purchase,
       CASE WHEN order_delivered_carrier_date < order_approved_at THEN 1 ELSE 0 END AS delivered_carrier_before_approved,
       CASE WHEN order_delivered_customer_date < order_delivered_carrier_date THEN 1 ELSE 0 END AS delivered_customer_before_carrier
FROM silver.crm_orders
WHERE (order_approved_at < order_purchase_timestamp)
   OR (order_delivered_carrier_date < order_approved_at)
   OR (order_delivered_customer_date < order_delivered_carrier_date)

ORDER BY order_id;

/*No issue with order_approved_at < order_purchase_timestamp

1382 rows with issue about order_delivered_carrier_date < order_approved_at &
order_delivered_customer_date < order_delivered_carrier_date

These regards CRM issues during the data ingestion. These issues will be fixed based on Business Rules.
The following rules work only when the CRM has this kind of issues*/


/*Fix when order_approved_at > order_delivered_carrier_date adding 1 day because the order will be sent
 the day after the approval (BUSINESS RULE)*/

UPDATE silver.crm_orders
SET order_delivered_carrier_date = DATEADD(DAY, 1, order_approved_at)
WHERE  order_approved_at > order_delivered_carrier_date ;

/*Fix when order_delivered_carrier_date > order_delivered_customer_date adding 3 day because the order will be delivered
 3 days after the carrier_date as standard carrier contract(BUSINESS RULE)*/

UPDATE silver.crm_orders
SET order_delivered_customer_date = DATEADD(DAY, 3, order_delivered_carrier_date)
WHERE  order_delivered_carrier_date > order_delivered_customer_date ;
```
---

## Referential Check with `silver.crm_order_payments`
```sql
SELECT DISTINCT(o.order_id) AS order_id_orders_table,
	   op.order_id AS order_id_payments_table
FROM silver.crm_orders o
LEFT JOIN silver.crm_order_payments op
ON o.order_id= op.order_id
WHERE op.order_id IS NULL 
-- There is one order_id not present in the silver.crm_order_payments table
-- The rows for this order_id will be eliminated (also in silver.crm_order_item and silver-crm_order_reviews)

--DELETE statement: remove rows for the order_id not present in payments table
DELETE FROM silver.crm_orders
WHERE order_id ='bfbd0f9bdef84302105ad712db648a6c'
```
---

✅ Data cleaned!

## Final DDL script with the new changes for `crm_orders`
No changes necessary to apply to structure, datatype and columns of this table. Initial DDL script unchanged.






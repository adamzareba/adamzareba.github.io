---
layout: post
title: Multidimensional reporting with CROSS APPLY and PIVOT in MS SQL Server
comments: true
---

In this post we're going to demonstrate how to use PIVOT relational operator to transform data from table-valued into another table. 
As an example we will use simple Data Warehouse (DWH) that stores annual company reports for harvesting fruits. 
The goal is to display report showing annual reports of sold fruits for each year. 

## DWH schema

Our simple database stores information about annual reports of sold fruits grouped by companies:

![database-schema](https://raw.githubusercontent.com/adamzareba/adamzareba.github.io/master/images/posts/2018-03-20/db_schema.PNG)

Following query: 

```sql
SELECT NAME, APPLE, GRAPE, YEAR
  FROM dbo.HARVESTING_FRUITS
  INNER JOIN COMPANY ON HARVESTING_FRUITS.COMPANY_ID = COMPANY.ID
```

returns data:

| NAME              | APPLE     | GRAPE     | YEAR   |
| ----------------- | --------- | --------- | ------ |
| Fruit Garden Inc. | 1000      | 200000    | 2015   |
| Fruit Garden Inc. | 5000      | 10000     | 2016   |
| Fruit Garden Inc. | 10000     | 20000     | 2017   |
| Village Fruits    | 7000      | 7000      | 2016   |
| Village Fruits    | 5000      | 1000000   | 2017   |
| Best Fruits Inc.  | 100000    | 1000000   | 2015   |
| Best Fruits Inc.  | 200000    | 2000000   | 2017   |

## Transform reports to separate rows - (CROSS) APPLY

We are going to use CROSS APPLY operator to populate the same operation for each record from left side - HARVESTING_FRUITS.
In below query we want to return pair of values for fruits reports based on year. Since we have known amount of fruit types, we will return concatenated strings, like:
* APPLES + YEAR
* GRAPES + YEAR

as FRUIT_YEAR, and return value for given report as AMOUNT.

```sql
SELECT COMPANY.NAME, FRUITS_BY_YEAR.*
	FROM dbo.HARVESTING_FRUITS FRUITS
	INNER JOIN COMPANY ON FRUITS.COMPANY_ID = COMPANY.ID
	CROSS APPLY (
		VALUES
			(CONCAT('APPLES - ', YEAR), APPLE),
			(CONCAT('GRAPES - ', YEAR), GRAPE)
	) FRUITS_BY_YEAR (FRUIT_YEAR, AMOUNT)
```

The output is following:

| NAME              | FRUIT_YEAR     | AMOUNT    |
| ----------------- | -------------- | --------- |
| Fruit Garden Inc. | APPLES - 2015	 | 1000    	 |
| Fruit Garden Inc. | GRAPES - 2015  | 200000    |
| Fruit Garden Inc. | APPLES - 2016	 | 5000    	 |
| Fruit Garden Inc. | GRAPES - 2016  | 10000     |
| Fruit Garden Inc. | APPLES - 2017	 | 10000     |
| Fruit Garden Inc. | GRAPES - 2017  | 20000     |
| Village Fruits	| APPLES - 2016	 | 7000    	 |
| Village Fruits	| GRAPES - 2016  | 7000    	 |
| Village Fruits	| APPLES - 2017	 | 5000    	 |
| Village Fruits	| GRAPES - 2017  | 1000000   |
| Best Fruits Inc.  | APPLES - 2015	 | 100000    |
| Best Fruits Inc.  | GRAPES - 2015  | 1000000   |
| Best Fruits Inc.  | APPLES - 2017	 | 200000    |
| Best Fruits Inc.  | GRAPES - 2017  | 2000000   |

## Transform rows to columns - PIVOT

Since we have reports separated by fruit name and year, we can turn values from FRUIT_YEAR column into multiple columns. Help comes with [PIVOT](https://docs.microsoft.com/en-us/sql/t-sql/queries/from-using-pivot-and-unpivot) operator. 
PIVOT syntax requires to use aggregate function so for AMOUNT we can use MAX function to just get value. Columns will be displayed based on given order:
* [APPLES - 2015]
* [APPLES - 2016]
* [APPLES - 2017]
* [GRAPES - 2015]
* [GRAPES - 2016]
* [GRAPES - 2017]

```sql
SELECT *
	FROM (
		SELECT COMPANY.NAME, FRUITS_BY_YEAR.*
			FROM dbo.HARVESTING_FRUITS FRUITS
			INNER JOIN COMPANY ON FRUITS.COMPANY_ID = COMPANY.ID
			CROSS APPLY (
				VALUES
					(CONCAT('APPLES - ', YEAR), APPLE),
					(CONCAT('GRAPES - ', YEAR), GRAPE)
			) FRUITS_BY_YEAR (FRUIT_YEAR, AMOUNT)
		) COLLECTED_FRUITS
	PIVOT (
		MAX(AMOUNT)
		FOR FRUIT_YEAR IN (
			[APPLES - 2015], [APPLES - 2016], [APPLES - 2017], 
			[GRAPES - 2015], [GRAPES - 2016], [GRAPES - 2017]
		)
	) COMBINED_FRUITS
```

The final report is:

| NAME              | APPLES - 2015  | APPLES - 2016  | APPLES - 2017  | GRAPES - 2015  | GRAPES - 2016  | GRAPES - 2017  |
| ----------------- | -------------- | -------------- | -------------- | -------------- | -------------- | -------------- |
| Fruit Garden Inc. | 100000    	 | NULL           | 200000         | 1000000        | NULL           | 2000000        |
| Village Fruits	| 1000           | 5000           | 10000          | 200000         | 10000          | 20000          |
| Best Fruits Inc.  | NULL      	 | 7000           | 5000           | NULL           | 7000           | 1000000        |

## Summary

MS SQL Server comes with very useful operators that simplifies working with DWHs. Although operators syntax are easy, CROSS APPLY and PIVOT could be used for complex transformations. 
Script for example database creation you can find [here](https://raw.githubusercontent.com/adamzareba/adamzareba.github.io/master/images/posts/2018-03-20/dwh_setup.sql).
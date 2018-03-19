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

![select_all](https://raw.githubusercontent.com/adamzareba/adamzareba.github.io/master/images/posts/2018-03-20/select_all.PNG)

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

![select_cross_apply](https://raw.githubusercontent.com/adamzareba/adamzareba.github.io/master/images/posts/2018-03-20/select_cross_apply.PNG)

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

![select_pivot](https://raw.githubusercontent.com/adamzareba/adamzareba.github.io/master/images/posts/2018-03-20/select_pivot.PNG)

## Summary

MS SQL Server comes with very useful operators that simplifies working with DWHs. Although operators syntax are easy, CROSS APPLY and PIVOT could be used for complex transformations. 
Script for example database creation you can find [here](https://raw.githubusercontent.com/adamzareba/adamzareba.github.io/master/images/posts/2018-03-20/dwh_setup.sql).
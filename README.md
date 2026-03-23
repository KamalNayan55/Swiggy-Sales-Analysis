# Swiggy-Sales-Analysis
----SWIGGY SALES ANALSIS

use Swiggy_DB;

select * from Swiggy_Data;

---Data Validation & Cleaning
--Null check

SELECT
	SUM(CASE WHEN State IS NULL THEN 1 ELSE 0 END) AS null_state,
	SUM(CASE WHEN City IS NULL THEN 1 ELSE 0 END) AS null_city,
	SUM(CASE WHEN Order_Date IS NULL THEN 1 ELSE 0 END) AS null_Order_Date,
	SUM(CASE WHEN Restaurant_Name IS NULL THEN 1 ELSE 0 END) AS null_Restaurant_Name,
	SUM(CASE WHEN Location IS NULL THEN 1 ELSE 0 END) AS null_Location,
	SUM(CASE WHEN Category IS NULL THEN 1 ELSE 0 END) AS null_Category,
	SUM(CASE WHEN Dish_Name IS NULL THEN 1 ELSE 0 END) AS null_Dish_Name,
	SUM(CASE WHEN Price_INR IS NULL THEN 1 ELSE 0 END) AS null_Price_INR,
	SUM(CASE WHEN Rating IS NULL THEN 1 ELSE 0 END) AS null_Rating,
	SUM(CASE WHEN Rating_Count IS NULL THEN 1 ELSE 0 END) AS null_Rating_Count
from Swiggy_Data;

--Blank/Empty String Check
SELECT * 
FROM Swiggy_Data
WHERE 
	State='' OR City='' OR Restaurant_Name='' OR Location='' OR Category=''OR Dish_Name='';

---Duplicate Detection
Select 
	State, City, Order_Date, Restaurant_Name, Location, Category,
	Dish_Name, Price_INR, Rating, Rating_Count, Count(*) As CNT
from Swiggy_Data
group by
	State, City, Order_Date, Restaurant_Name, Location, Category,
	Dish_Name, Price_INR, Rating, Rating_Count
Having Count(*)>1;

---Duplicate Removal
WITH CTE AS(
SELECT * ,ROW_NUMBER() OVER(
PARTITION BY State, City, Order_Date, Restaurant_Name, Location, Category,
Dish_Name, Price_INR, Rating, Rating_Count
ORDER BY (SELECT NULL)
)AS RN
FROM Swiggy_data
)
DELETE FROM CTE WHERE RN>1

--CREATING SCHEMA
--DIMENSION TABLE
--DATE TABLE

CREATE TABLE dim_date(
	date_id int Identity(1,1) PRIMARY KEY,
	FULL_DATE Date,
	Year INT,
	Month INT,
	Month_name VARCHAR(20),
	Quarter INT,
	Day INT,
	Weeek INT
);

---Dim Location
Create Table dim_location(
	Location_id int identity(1,1) PRIMARY KEY,
	State VARCHAR(100),
	City VARCHAR(100),
	Location VARCHAR(200)
);

---Dim Restaurant
Create table dim_restaurnat(
	Restaurnat_id int identity(1,1) PRIMARY KEY,
	Restaurnat VARCHAR(200)
);

---Dim Category
Create table dim_category(
	Category_id INT IDENTITY(1,1) PRIMARY KEY,
	Category VARCHAR(200)
);

---Dim Dish
Create Table dim_dish(
	Dish_id INT IDENTITY(1,1) PRIMARY KEY,
	Dish_name VARCHAR(200)
);

--Fact Table
Create Table fact_Swiggy_ordrs(
	Order_id int identity(1,1) PRIMARY KEY,

	date_id INT,
	Price_INR DECIMAL(10,2),
	Rating DECIMAL(4,2),
	Rating_count INT,

	Location_id INT,
	Restaurnat_id INT,
	Category_id INT,
	Dish_id INT,

	FOREIGN KEY (date_id) REFERENCES dim_date(date_id),
	FOREIGN KEY (location_id) REFERENCES dim_location(location_id),
	FOREIGN KEY (Restaurnat_id) REFERENCES dim_restaurnat(Restaurnat_id),
	FOREIGN KEY (Category_id) REFERENCES dim_category(Category_id),
	FOREIGN KEY (Dish_id) REFERENCES dim_dish(Dish_id)
);

Select * from fact_Swiggy_ordrs;

--Insert data in Dim table
--dim_date
INSERT INTO dim_date(FULL_DATE, Year, Month, Month_name, Quarter, Day, Weeek)
SELECT DISTINCT
	Order_Date,
	Year(Order_Date),
	Month(Order_Date),
	DATENAME(MONTH,Order_Date),
	DATEPART(QUARTER,Order_Date),
	DAY(Order_Date),
	DATEPART(Week, Order_Date)
from Swiggy_Data
where Order_Date is not null;

Select * from dim_date;

--dim_location
INSERT INTO dim_location(State, City, Location)
select Distinct	
	State,
	City,
	Location
from Swiggy_Data;

Select * from dim_location;

--dim_restuarnat
INSERT INTO dim_restaurnat(Restaurnat)
SELECT DISTINCT	
	Restaurant_Name
from Swiggy_Data;

Select * from dim_restaurnat;

--dim_category
INSERT INTO dim_category(Category)
SELECT DISTINCT
	Category
from Swiggy_Data;

Select * from dim_category;

--dim_dish
INSERT INTO dim_dish(Dish_name)
SELECT DISTINCT
	Dish_Name
FROM Swiggy_Data;

Select * from dim_dish;

--INSERT INTO FACT TABLE

INSERT INTO fact_Swiggy_ordrs
(
	date_id,
	Price_INR,
	Rating,
	Rating_count,
	Location_id,
	Restaurnat_id,
	Category_id,
	Dish_id
)
SELECT 
	dd.date_id,
	s.Price_INR,
	s.Rating,
	s.Rating_count,

	dl.Location_id,
	dr.Restaurnat_id,
	dc.Category_id,
	ds.dish_id
from Swiggy_Data s

join dim_date dd
	ON dd.FULL_DATE=s.Order_Date

join dim_location dl
	ON dl.State=s.State
	and dl.City=s.City
	and dl.Location=s.Location

join dim_restaurnat dr
	ON dr.Restaurnat=s.Restaurant_Name

join dim_category dc
	ON dc.Category=s.Category

join dim_dish ds
	ON ds.Dish_name=s.Dish_Name

Select * from fact_Swiggy_ordrs;

Select * from fact_Swiggy_ordrs f
join dim_date d on d.date_id=f.date_id
join dim_location l on l.Location_id=f.Location_id
join dim_restaurnat r on r.Restaurnat_id=f.Restaurnat_id
join dim_category c on c.Category_id=f.Category_id
join dim_dish s on s.Dish_id=f.Dish_id

---KPI Development
--Total Orders

Select count(*) as Total_Orders
from fact_Swiggy_ordrs; 

--Total Revenue(INR Million)
SELECT 
FORMAT(SUM(CONVERT(float,Price_INR))/1000000,'N2')+'INR Million'
as [Total Revenue]
from fact_Swiggy_ordrs;

--Average Dish price
SELECT 
    FORMAT(AVG(Price_INR), 'N2') + ' INR' AS [Avg Dish Price]
FROM fact_Swiggy_ordrs;

--Average Rating
Select Round(AVG(Rating),0) as [Average Rating]
from fact_Swiggy_ordrs;


--Deep-Dive Business Analysis
--Monthly Orders Trends
SELECT
	d.Year,
	d.Month,
	d.Month_name,
	count(*) as Total_Orders
from fact_Swiggy_ordrs f
join dim_date d on f.date_id=d.date_id
group by
	d.Year,
	d.Month,
	d.Month_name
Order by Count(*) Desc
;

SELECT
	d.Year,
	d.Month,
	d.Month_name,
	Sum(Price_INR) as Total_Revenue
from fact_Swiggy_ordrs f
join dim_date d on f.date_id=d.date_id
group by
	d.Year,
	d.Month,
	d.Month_name
Order by Sum(Price_INR) Desc
;

---Quarterly order trends
SELECT
	d.Year,
	d.Quarter,
	count(*) as Total_Orders
from fact_Swiggy_ordrs f
join dim_date d on f.date_id=d.date_id
group by
	d.Quarter,
	d.Year
Order by count(*) desc
;

--Year-wise growth
SELECT
	d.Year,
	count(*) as Total_Orders
from fact_Swiggy_ordrs f
join dim_date d on f.date_id=d.date_id
group by
	d.Year
Order by count(*) desc
;

--Day-of-week patterns
--Order by Day of Week(Mon -sun)
SELECT
	DATENAME(weekday,d.full_date) as Day_name,
	Count(*) as Total_Orders
from fact_Swiggy_ordrs f
join dim_date d on d.date_id=d.date_id
Group by 
	DATENAME(weekday,d.full_date),
	Datepart(weekday,d.full_date)
Order by
	Datepart(weekday,d.full_date)
;

---Location-Based Analysis
--Top 10 cities by order volume
SELECT Top 10
	l.city,
	count(*) As Total_Orders
from fact_Swiggy_ordrs f
join dim_location l
ON l.Location_id=f.Location_id
Group by	
	l.City
Order by Count(*) desc
;

SELECT Top 10
	l.city,
	sum(f.Price_INR) As Total_Revenue
from fact_Swiggy_ordrs f
join dim_location l
ON l.Location_id=f.Location_id
Group by	
	l.City
Order by sum(f.Price_INR) ASC
;

--Revenue contribution by states
SELECT 
	l.State,
	sum(f.Price_INR) As Total_Revenue
from fact_Swiggy_ordrs f
join dim_location l
ON l.Location_id=f.Location_id
Group by	
	l.State
Order by sum(f.Price_INR) ASC
;

---Food Performance
--Top 10 restaurants by orders
SELECT Top 10
	R.Restaurnat,
	count(*) As Total_Orders
from fact_Swiggy_ordrs f
join dim_restaurnat R
ON R.Restaurnat_id=f.Restaurnat_id
Group by	
	R.Restaurnat
Order by Count(*) desc
;

SELECT Top 10
	R.Restaurnat,
	Sum(F.Price_INR) As Total_Revenue
from fact_Swiggy_ordrs f
join dim_restaurnat R
ON R.Restaurnat_id=f.Restaurnat_id
Group by	
	R.Restaurnat
Order by Sum(F.Price_INR) desc
;

--Top categories (Indian, Chinese, etc.)
SELECT Top 10
	C.Category,
	count(*) As Total_Orders
from fact_Swiggy_ordrs f
join dim_category C
ON C.Category_id=f.Category_id
Group by	
	C.Category
Order by Count(*) desc
;

--Most ordered dishes
SELECT Top 10
	d.Dish_name,
	count(*) As Total_Orders
from fact_Swiggy_ordrs f
join dim_dish d
ON d.Dish_id=f.Dish_id
Group by	
	d.Dish_name
Order by Count(*) desc
;
---Cuisine performance → Orders + Avg Rating
SELECT 
	C.Category,
	Count(*) AS Total_Orders,
	AVG(f.rating) AS AVG_Rating
from fact_Swiggy_ordrs f
join dim_category c
On c.Category_id=f.Category_id
Group by
	C.Category
Order by C.Category Desc

/*Customer Spending Insights
Buckets of customer spend:
Under 100
100–199
200–299
300–499
500+
With total order distribution across these ranges.
*/
SELECT 
	CASE
		WHEN CONVERT(FLOAT,Price_INR) < 100 THEN 'UNDER 100'
		WHEN CONVERT(FLOAT,Price_INR) BETWEEN 100 AND 199 THEN '100-199'
		WHEN CONVERT(FLOAT,Price_INR) BETWEEN 200 AND 299 THEN '200-299'
		WHEN CONVERT(FLOAT,Price_INR) BETWEEN 300 AND 499 THEN '300-499'
		ELSE '500+'
	END AS Price_range,
	count(*) AS Total_Orders
From fact_Swiggy_ordrs
Group by
	CASE
		WHEN CONVERT(FLOAT,Price_INR) < 100 THEN 'UNDER 100'
		WHEN CONVERT(FLOAT,Price_INR) BETWEEN 100 AND 199 THEN '100-199'
		WHEN CONVERT(FLOAT,Price_INR) BETWEEN 200 AND 299 THEN '200-299'
		WHEN CONVERT(FLOAT,Price_INR) BETWEEN 300 AND 499 THEN '300-499'
		ELSE '500+'
	END 
Order by Total_Orders DESC
;

--Ratings Analysis
--Distribution of dish ratings from 1–5.
Select
	Rating,
	Count(*) as Rating_count
from fact_Swiggy_ordrs
Group by Rating
Order By Rating DESC
;

---------------------------------------THE END-----------------------------------------------------------

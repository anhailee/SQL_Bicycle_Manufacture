# SQL_Bicycle_Manufacture
Utilized SQL in Google BigQuery to write and execute queries to find the desired data.

## **1. INTRODUCTION**

Adventure Works Cycles is a large, multinational manufacturing company produces and distributes metal bicycles for commercial markets in North America, Europe, and Asia.
In 2000, Adventure Works Cycles acquired a small manufacturing plant, Wide World Importers, located in Mexico City. 
In 2001, Wide World Importers became the sole manufacturer and distributor of the touring bicycle product line.  

## **2. THE GOAL OF PROJECT**

With the continuous development in the bicycle manufacturing and business sector, the Manager realizes that they need to grasp detailed information such as:

+ Revenue, orders, and discount strategies.
+ Customer information and retention rates to improve relationships and increase sales.
+ Inventory and supply chain data for effective management and meeting market demands.
+ Analytical and forecasting tools to make strategic decisions based on accurate data and market analysis.

## **3. DATASET ACCESS**

Dataset: adventureworks2019 (public Google BigQuery dataset)

Dataset dictionary: Please reach file "Data Dictionary" attached above

Dataset Schema: https://i0.wp.com/improveandrepeat.com/wp-content/uploads/2018/12/AdvWorksOLTPSchemaVisio.png?ssl=1

Dataset access:

1. Log in to your Google Cloud Platform account and create a new project.
2. Navigate to the BigQuery console and select your newly created project.
3. In the navigation panel, select "Add Data" and then "Star a project by name".
4. Enter the project name "adventurework2019"


## **4. READ AND EXPLAIN DATASET**

Dataset Dictionary: https://drive.google.com/file/d/1bwwsS3cRJYOg1cvNppc1K_8dQLELN16T/view?usp=share_link

## **5. EXPLORING DATASET**

***Query 01:*** Calc Quantity of items, Sales value & Order quantity by each Subcategory in L12M

**CODE:**
```sql
SELECT
      FORMAT_DATETIME('%b %Y', s.ModifiedDate) AS period
     ,ps.name                                  AS category
     ,SUM(s.OrderQty)                          AS qty_item
     ,ROUND(SUM(LineTotal),0)                  AS total_sales
     ,COUNT(SalesOrderID)                      AS order_cnt
FROM
  `adventureworks2019.Sales.SalesOrderDetail` AS S
LEFT JOIN
  `adventureworks2019.Production.Product` AS P
ON
  s.ProductID = p.ProductID
LEFT JOIN
  `adventureworks2019.Production.ProductSubcategory` AS PS
ON
  CAST(p.ProductSubcategoryID AS int) = ps.ProductSubcategoryID
GROUP BY 1,2
ORDER BY 1 DESC,2
```
**RESULT**
|      Period     |      category       | qty_item     |total_sales| order_cnt|
| :------------:|:-------------:|:-----:|:-----:|:-----:|
|    Sep 2013   | Bike Racks         |  312    |  22829.0   |71    |
|     Sep 2013  | Bike Stands        |   26   |  4134.0    |26   |
|    Sep 2013   |	Bottles and Cages  |    803  |  4677.0    |606    |
|    Sep 2013   | Bottom Brackets    |   60   |  3118.0   |  28   |
|     Sep 2013  | Brakes             |    100  |  6390.0  |39  |
|     Sep 2013  | Caps               |    440  |  2879.0   |203   |
|     Sep 2013  | Chains             |    62  |  753.0   |24  |
|     Sep 2013  | Cleaners           |    296  |  1612.0   |108  |
|     Sep 2013  | Cranksets          |    75  |  13956.0   |35   |
|     ...       | ...                |    ...  |  ...   | ...   |


***Query 02:*** Calc % YoY growth rate by SubCategory & release top 3 cat with highest grow rate. Can use metric: quantity_item. Round results to 2 decimal

**CODE:**
```sql
WITH sale_info AS(
  SELECT
    EXTRACT(year FROM s.ModifiedDate)           AS year
    ,ps.name                                    AS category
    ,SUM(s.OrderQty)                            AS qty_item
  FROM
    `adventureworks2019.Sales.SalesOrderDetail` AS s
  LEFT JOIN
    `adventureworks2019.Production.Product` AS p
  ON
    s.ProductID = p.ProductID
  LEFT JOIN
    `adventureworks2019.Production.ProductSubcategory` AS ps
  ON
    CAST(p.ProductSubcategoryID AS int) = ps.ProductSubcategoryID
  GROUP BY 1,2
  ORDER BY 2,1),

  sale_diff AS(
  SELECT
    year
    ,category
    ,qty_item
    ,LEAD(Qty_item) OVER(PARTITION BY category ORDER BY year DESC) AS prv_qty
  FROM
    sale_info)

SELECT
  category,
  qty_item,
  prv_qty,
  ROUND(((qty_item-prv_qty)/prv_qty),2) AS qty_diff
FROM
  sale_diff
ORDER BY 4 DESC
LIMIT 3
```
**RESULT**

|      category      |      qty_item       | prv_qty     |qty_diff|
| :------------:|:-------------:|:-----:|:-----:|
|    Mountain Frames        |        3168      |  510    |  5.21   |
|     Socks       |        2724      |   523   |  4.21    |
|     Road Frames     | 5564             |    1137  |  3.89   |


***Query 03:*** Ranking Top 3 TeritoryID with biggest Order quantity of every year. If there's TerritoryID with same quantity in a year, do not skip the rank number

**CODE:**
```sql
WITH sale_info AS(
  SELECT
    EXTRACT(year
    FROM
    sod.ModifiedDate)     AS year
    ,soh.TerritoryID
    ,SUM(sod.OrderQty)    AS order_cnt
  FROM
    `adventureworks2019.Sales.SalesOrderDetail` AS sod
  LEFT JOIN
    `adventureworks2019.Sales.SalesOrderHeader` AS soh
  ON
    sod.SalesOrderID = soh.SalesOrderID
  GROUP BY 1,2),

  sale_diff AS(
  SELECT
    *
    ,DENSE_RANK() OVER(PARTITION BY year ORDER BY order_cnt DESC) AS rk
  FROM
    sale_info
  ORDER BY
    1 DESC)

SELECT *
FROM sale_diff
WHERE
  rk IN (1,2,3)
```

**RESULT**

|      year      |      TerritoryID       | order_cnt     |rk|
| :------------:|:-------------:|:-----:|:-----:|
|    2014          |        4      |  11632    |  1  |
|     2014        |        6      |   9711   | 2   |
|     2014     | 2             |    8823  |  3  |
|    2013      |        4      |   26682   |  1   |
|     2013   | 6             |    22553  |  2 |
|     2013   | 1             |    17452  |  3 |
|     2012   | 4             |    17553  |  1 |
|     2012   | 6             |    14412  |  2 |
|     2012   | 1             |    8537  |  3 |
|     ...   | ...             |    ...  |  ...   |

***Query 04:*** Calc Total Discount Cost belongs to Seasonal Discount for each SubCategory

**CODE:**
```sql
SELECT
  EXTRACT(year FROM sod.ModifiedDate)                         AS year
  ,ps.name                                                    AS subcategory
  ,ROUND(SUM(sod.OrderQty*sod.UnitPrice*so.DiscountPct),0)    AS total_cost
FROM
  `adventureworks2019.Sales.SalesOrderDetail`                 AS sod
LEFT JOIN
  `adventureworks2019.Sales.SpecialOffer`                     AS so
ON
  sod.SpecialOfferID = so.SpecialOfferID
LEFT JOIN
  `adventureworks2019.Production.Product`                     AS p
ON
  sod.ProductID = p.ProductID
LEFT JOIN
  `adventureworks2019.Production.ProductSubcategory`          AS ps
ON
  CAST(p.ProductSubcategoryID AS INT) = ps.ProductSubcategoryID
WHERE
  LOWER(so.type) LIKE '%seasonal discount%'
GROUP BY 1,2
```

**RESULT**

|      year     |      subcategory       | total_cost     |
| :------------:|:-------------:|:-----:|
|    2012          |        Helmets     |  828.0  |
|     2013        |        Helmets     |   1606.0  |

***Query 05:*** Retention rate of Customer in 2014 with status of Successfully Shipped (Cohort Analysis)

**CODE:**
```sql
WITH raw_data_1 AS(
  SELECT
    EXTRACT(month FROM ShipDate) AS month_join
    ,CustomerID
    ,COUNT(SalesOrderID)         AS sales_cnt
  FROM
    `adventureworks2019.Sales.SalesOrderHeader`
  WHERE
    EXTRACT(year FROM ShipDate) = 2014
    AND status = 5
  GROUP BY 1,2
  ORDER BY 2),

  raw_data_2 AS(
  SELECT
    *
    ,ROW_NUMBER() OVER(PARTITION BY customerid ORDER BY month_join) AS num
  FROM
    raw_data_1
  ORDER BY 1),

  first_order AS(
  SELECT
    raw_data_2.month_join AS month_order,
    CustomerID
  FROM
    raw_data_2
  WHERE
    num = 1
  ORDER BY 2),

  all_join AS(
  SELECT
    r1.month_join,
    r1.CustomerID,
    f.month_order,
    CONCAT('M - ',r1.month_join - f.month_order) AS month_diff,
    r1.sales_cnt
  FROM
    raw_data_1 AS r1
  LEFT JOIN
    first_order AS f
  ON
    r1.customerID = f.customerID
  ORDER BY
    2)

SELECT
  month_join,
  month_diff,
  COUNT(customerID) AS cutomer_cnt
FROM
  all_join
GROUP BY 1,2
ORDER BY 1
```

**RESULT**

|      month_join      |      month_diff       |cutomer_cnt       |
| :------------:|:-------------:|:-------------:|
|     1        |        M - 0  |  2076  |  
|     2        |        M - 0   |  1805   |  
|     2        |        M - 1  |  78 |  
|     3        |        M - 1   | 51  |  
|     3        |        M - 2   |  89   |  
|     3        |        M - 0   |  1918  |  
|     4        |        M - 0 |  1906  |  
|     4        |        M - 1   |  43  |  
|     4        |        M - 2   |  61  |  
|     ...           | ...             |    ...  | 


***Query 06:*** Trend of Stock level & MoM diff % by all product in 2011. If %gr rate is null then 0. Round to 1 decimal

**CODE:**
```sql
WITH raw_data AS (
  SELECT
     p.Name AS name
    ,EXTRACT(year FROM w.ModifiedDate) AS yr
    ,EXTRACT(month FROM w.ModifiedDate) AS mth
    ,SUM(w.StockedQty) AS Stock_curr
  FROM
    `adventureworks2019.Production.Product`AS p
  LEFT JOIN
    `adventureworks2019.Production.WorkOrder` AS w
  ON
    p.ProductID = w.ProductID
  WHERE
    EXTRACT(year FROM w.ModifiedDate) = 2011
  GROUP BY 1,2,3
  ORDER BY 1,2,3 DESC
  ),

  pre_result AS(
  SELECT
     name
    ,yr
    ,mth
    ,Stock_curr
    ,LEAD(Stock_curr) OVER(PARTITION BY Name ORDER BY mth DESC) AS Stock_prv
  FROM
    raw_data
  ORDER BY
    1)
    
SELECT
   *
  ,CASE
    WHEN (100.0 * (Stock_curr - Stock_prv)/Stock_prv) IS NULL THEN 0
  ELSE
  ROUND((100.0 * (Stock_curr - Stock_prv)/Stock_prv),1)
END
  AS diff
FROM
  pre_result

```

**RESULT**

|      Period     |      yr       | qty_item     |total_sales| order_cnt|order_cnt|
| :------------:|:-------------:|:-----:|:-----:|:-----:|:-----:|
|    BB Ball Bearing  |2011        |  12   |  8475  |14544    |-41.7    |
|     BB Ball Bearing | 2011       |   11  |  14544 |19175    |-24.2   |
|    BB Ball Bearing  |	2011       |    10 |  19175 |8845   |116.8   |
|    BB Ball Bearing  | 2011       |   9   |  8845  |9666    |-8.5   |
|     BB Ball Bearing |2011        |    8  |  9666  |12837    | -24.7 |
|     BB Ball Bearing | 2011       |    7  |  12837 |5259   |  144.1 |
|    BB Ball Bearing  | 2011       |    6  |  5259  |null    |0.0 |
|     Blade           | 2011       |    12 |  1842  |3598   |-48.8  |
|     Blade           | 2011       |    11 |  3598  |4670    |-23.0   |
|     ...             | ...        |   ... |  ...   | ...  |...   |


***Query 07:*** Calc MoM Ratio of Stock / Sales in 2011 by product name. Order results by month desc, ratio desc. Round Ratio to 1 decimal

**CODE:**
```sql
WITH tbl_sales AS(
  SELECT
     EXTRACT (month FROM s.ModifiedDate)        AS mth
    ,EXTRACT (year FROM s.ModifiedDate)         AS yr
    ,s.ProductID
    ,p.name                                     AS category
    ,SUM(s.OrderQty) AS sales_cnt
  FROM
    `adventureworks2019.Sales.SalesOrderDetail` AS s
  LEFT JOIN
    `adventureworks2019.Production.Product`     AS p
  ON
    s.ProductID = p.ProductID
  WHERE
    EXTRACT (year FROM s.ModifiedDate) = 2011
  GROUP BY 1,2,3,4
  ORDER BY 1 DESC,2
  ),

  tbl_stock AS(
  SELECT
     EXTRACT(month FROM ModifiedDate) AS mth
    ,EXTRACT(year FROM ModifiedDate) AS yr
    ,productID
    ,SUM(StockedQty) AS stock_cnt
  FROM
    `adventureworks2019.Production.WorkOrder`
  WHERE
    EXTRACT(year
    FROM
      ModifiedDate) = 2011
  GROUP BY 1,2,3
  ORDER BY 1 DESC)
SELECT
   ts.mth
  ,ts.yr
  ,ts.ProductID
  ,ts.category
  ,ts.sales_cnt
  ,tsk.stock_cnt
  ,ROUND(tsk.stock_cnt/ts.sales_cnt,1) AS ratio
FROM
  tbl_sales AS ts
JOIN
  tbl_stock AS tsk
ON
  ts.productid = tsk.productid
  AND ts.mth = tsk.mth
  AND ts.yr = tsk.yr
ORDER BY 1 DESC,7 DESC
```

**RESULT**

|      mth       |      yr       | ProductID     |category|  sales_cnt  |  stock_cnt  |   ratio  |   
| :------------:|:-------------:|:-----:|:-----:|:-----:|:-----:|:-----:|
|    12          |        2011      |  745    |  HL Mountain Frame - Black, 48    |  1   |  27   | 27   |
|     12         |        2011      |   743   |  HL Mountain Frame - Black, 42    |  1   |  26    |26    |
|     12         | 2011             |    748  |  HL Mountain Frame - Silver, 38   |  2   |  32    |16    |
|     12         |        2011      |   722   |  LL Road Frame - Black, 58        |  4   |  47    |11.8   |
|     12         | 2011             |    747  |  HL Mountain Frame - Black, 38    |  3   |  31    |10.3   |
|     12         |        2011      |   726   |  LL Road Frame - Red, 48          |  5   |  36   |7.2    |
|     12         | 2011             |    738  |  	LL Road Frame - Black, 52       |  10   |  64   |6.4  |
|     12         |        2011      |   741   |  HL Mountain Frame - Silver, 48   |  5   |  27    |5.4    |
|     12         | 2011             |    730  |  LL Road Frame - Red, 62          |  7   |  38    |5.4   |
|     ...             | ...        |   ... |  ...   | ...  |...   |...   |


***Query 08:*** No of order and value at Pending status in 2014

**CODE:**
```sql
SELECT
   EXTRACT(year FROM ModifiedDate)  AS yr
  ,status
  ,COUNT(DISTINCT(PurchaseOrderID)) AS order_cnt
  ,SUM(TotalDue) AS value
FROM
  `adventureworks2019.Purchasing.PurchaseOrderHeader`
WHERE
  status = 1
  AND EXTRACT(year FROM ModifiedDate)= 2014
GROUP BY 1,2
```

**RESULT**

|      yr       |      status       | order_cnt     |value|    
| :------------:|:-------------:|:-----:|:-----:|
|    2014          |        1      |  224    |  3873579.0    |  

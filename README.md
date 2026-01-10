# Customer Churn Analysis

Customer churn is rarely caused by a single factor — it’s usually the result of multiple, interconnected drivers. In this project, my goal was not just to build dashboards, but to understand what truly drives customer exits and how different segments contribute to churn. This led me to explore analytical features in Power BI that go beyond standard slicing and dicing — especially the Decomposition Tree, which became a key enabler in uncovering root causes.

## Problem Statement

Although customer churn data was available, the analysis was limited to descriptive metrics and static reporting. There was no end-to-end pipeline that ensured data quality, proper modeling, advanced analytics, and interactive exploration. The objective was to build a scalable BI solution that not only measured churn, but also enabled root-cause and driver analysis using advanced Power BI capabilities.


### Steps followed 

#### SSMS

select * from dbo.ActiveCustomer
select * from dbo.Bank_Departure
select * from dbo.CreditCard
select * from dbo.CustomerInfo
select * from dbo.ExitCustomer
select * from dbo.Gender
select * from dbo.Geography

--basic max/min checks
SELECT
    COUNT(*) AS TotalRows,
    COUNT(DISTINCT CustomerId) AS DistinctCustomers,
    MIN([Bank DOJ]) AS MinBankDOJ,
    MAX([Bank DOJ]) AS MaxBankDOJ,
    MIN(Age) AS MinAge,
    MAX(Age) AS MaxAge,
    MIN(CreditScore) AS MinCreditScore,
    MAX(CreditScore) AS MaxCreditScore,
    MIN(Balance) AS MinBalance,
    MAX(Balance) AS MaxBalance,
    MIN(EstimatedSalary) AS MinSalary,
    MAX(EstimatedSalary) AS MaxSalary
FROM dbo.Bank_Departure;

--null checks
SELECT
    SUM(CASE WHEN RowNumber IS NULL THEN 1 ELSE 0 END) AS Null_RowNumber,
    SUM(CASE WHEN CustomerId IS NULL THEN 1 ELSE 0 END) AS Null_CustomerId,
    SUM(CASE WHEN CreditScore IS NULL THEN 1 ELSE 0 END) AS Null_CreditScore,
    SUM(CASE WHEN GeographyID IS NULL THEN 1 ELSE 0 END) AS Null_GeographyID,
    SUM(CASE WHEN GenderID IS NULL THEN 1 ELSE 0 END) AS Null_GenderID,
    SUM(CASE WHEN Age IS NULL THEN 1 ELSE 0 END) AS Null_Age,
    SUM(CASE WHEN Tenure IS NULL THEN 1 ELSE 0 END) AS Null_Tenure,
    SUM(CASE WHEN Balance IS NULL THEN 1 ELSE 0 END) AS Null_Balance,
    SUM(CASE WHEN NumOfProducts IS NULL THEN 1 ELSE 0 END) AS Null_NumOfProducts,
    SUM(CASE WHEN HasCrCard IS NULL THEN 1 ELSE 0 END) AS Null_HasCrCard,
    SUM(CASE WHEN IsActiveMember IS NULL THEN 1 ELSE 0 END) AS Null_IsActiveMember,
    SUM(CASE WHEN EstimatedSalary IS NULL THEN 1 ELSE 0 END) AS Null_EstimatedSalary,
    SUM(CASE WHEN Exited IS NULL THEN 1 ELSE 0 END) AS Null_Exited,
    SUM(CASE WHEN [Bank DOJ] IS NULL THEN 1 ELSE 0 END) AS Null_BankDOJ
FROM dbo.Bank_Departure;

--Duplicate checks

--Duplicate CustomerId
SELECT CustomerId, COUNT(*) AS Cnt
FROM dbo.Bank_Departure
GROUP BY CustomerId
HAVING COUNT(*) > 1
ORDER BY Cnt DESC;

-- Duplicate rows
WITH cte AS (
    SELECT *,
           CONCAT(
             RowNumber,'|',CustomerId,'|',CreditScore,'|',GeographyID,'|',GenderID,'|',
             Age,'|',Tenure,'|',Balance,'|',NumOfProducts,'|',HasCrCard,'|',
             IsActiveMember,'|',EstimatedSalary,'|',Exited,'|',CONVERT(varchar(10), [Bank DOJ], 120)
           ) AS row_signature
    FROM dbo.Bank_Departure
)
SELECT row_signature, COUNT(*) AS Cnt
FROM cte
GROUP BY row_signature
HAVING COUNT(*) > 1

--Domain checks

--Flags should be only 0/1

--for hascredircard
SELECT
    HasCrCard AS BadValue,
    COUNT(*) AS CountRows
FROM dbo.Bank_Departure
WHERE HasCrCard IS NULL
   OR HasCrCard NOT IN (0, 1)
GROUP BY HasCrCard;

--for isactivemember
SELECT
    IsActiveMember AS BadValue,
    COUNT(*) AS CountRows
FROM dbo.Bank_Departure
WHERE IsActiveMember IS NULL
   OR IsActiveMember NOT IN (0, 1)
GROUP BY IsActiveMember;

--for exited
SELECT
    Exited AS BadValue,
    COUNT(*) AS CountRows
FROM dbo.Bank_Departure
WHERE Exited IS NULL
   OR Exited NOT IN (0, 1)
GROUP BY Exited;

--range/sanity checks

-- CreditScore sanity (300–850)
SELECT COUNT(*) AS Bad_CreditScore
FROM dbo.Bank_Departure
WHERE CreditScore IS NULL OR CreditScore < 300 OR CreditScore > 850;

-- Age sanity (18–110)
SELECT COUNT(*) AS Bad_Age
FROM dbo.Bank_Departure
WHERE Age IS NULL OR Age < 18 OR Age > 110;

-- Tenure sanity (0–10)
SELECT COUNT(*) AS Bad_Tenure
FROM dbo.Bank_Departure
WHERE Tenure IS NULL

-- Balance should not be negativ
SELECT COUNT(*) AS Bad_Balance
FROM dbo.Bank_Departure
WHERE Balance IS NULL OR Balance < 0;

-- EstimatedSalary should not be negative
SELECT COUNT(*) AS Bad_Salary
FROM dbo.Bank_Departure
WHERE EstimatedSalary IS NULL OR EstimatedSalary < 0;

--Logical consistency checks

-- DOJ should not be in the future
SELECT max(Bank_DOJ) AS Future_BankDOJ
FROM dbo.Bank_Departure

SELECT min(Bank_DOJ) AS Future_BankDOJ
FROM dbo.Bank_Departure

--Key distribution checks

SELECT GeographyID, COUNT(*) AS Cnt
FROM dbo.Bank_Departure
GROUP BY GeographyID
ORDER BY Cnt DESC;

SELECT GenderID, COUNT(*) AS Cnt
FROM dbo.Bank_Departure
GROUP BY GenderID
ORDER BY Cnt DESC;

--altering fields where exit date doesnt match is_active

alter table dbo.Bank_Departure
add Exit_Date Date

--for analysis purpose
update dbo.Bank_Departure
set Exit_Date = Bank_DOJ
where Exited = 1

update dbo.Bank_Departure
set IsActiveMember = 0
where Exited = 1

alter table dbo.Bank_Departure
drop column RowNumber

--extra columns for analysis in power bi

alter table dbo.Bank_Departure
add Credit_Score_Category varchar(20)

update dbo.Bank_Departure
set Credit_Score_Category = 
(case
when CreditScore between 800 and 850 then 'Excellent'
when CreditScore between 740 and 799 then 'Very Good'
when CreditScore between 670 and 739 then 'Good'
when CreditScore between 580 and 669 then 'Fair'
else 'Poor'
end)

alter table dbo.Bank_Departure
add Age_Category varchar(20)

update dbo.Bank_Departure
set Age_Category = 
(case
when Age <= 30 then '< 30'
when Age between 31 and 40 then '31 - 40'
when Age between 41 and 50 then '41 - 50'
when Age between 51 and 60 then '51 - 60'
when Age between 61 and 70 then '61 - 70'
when Age between 71 and 80 then '71 - 80'
when Age between 81 and 90 then '81 - 90'
else '> 90'
end)

alter table dbo.Bank_Departure
drop column Salary_Bands

update dbo.Bank_Departure
set Salary_Bands = 
(case
when EstimatedSalary < 5000000 then 'Low'
when EstimatedSalary between 5000000 and 15000000 then 'Medium'
when EstimatedSalary between 670 and 739 then 'Good'
when EstimatedSalary between 580 and 669 then 'Fair'
else 'Poor'
end)

select max(EstimatedSalary)
from dbo.Bank_Departure

select min(EstimatedSalary)
from dbo.Bank_Departure

select avg(EstimatedSalary)
from dbo.Bank_Departure

-- >15000000, 10000000 - 15000000, 5000000 - 10000000, <5000000

alter table dbo.Bank_Departure
add Salary_Bracket varchar(20)

update dbo.Bank_Departure
set Salary_Bracket =
(case
when EstimatedSalary>15000000 then 'Very High'
when EstimatedSalary between 10000000 and 15000000 then 'High'
when EstimatedSalary between 5000000 and 9999999 then 'Medium'
else 'Low'
end)

alter table dbo.Bank_Departure
drop column Salary_Bracket

select max(Balance)
from dbo.Bank_Departure

--0, 26000000
 
alter table dbo.Bank_Departure
add Balance_Band varchar(20)

update dbo.Bank_Departure
set Balance_Band =
(case
when Balance>20000000 then 'High Balance'
when Balance between 10000000 and 20000000 then 'Medium Balance'
when Balance between 1 and 9999999 then 'Low Balance'
else 'Zero Balance'
end)


select C.CustomerId
from dbo.Bank_Departure C
where C.CustomerID in 
(
select C2.CustomerID
from dbo.Bank_Departure C2
where C.CustomerId <> C2.CustomerID and
C.row_number = C2.row_number
)
custid

15565701 - 3/1

select customerid, count(*) from Bank_Departure
group by CustomerId
having count(*)>1

sp_help

create table customers1
(
customerid int primary key,
customername varchar(20) not null
)

create table orders1
(
orderid int primary key,
customerid int references customers1 (customerid)
)

insert into customers1
values (1,'shan'),(2,'Man')

insert into orders1
values (1,1),(2,2),(3,1)

delete from customers1 where customerid=1

sp_help orders1

alter table orders1
drop constraint FK__orders1__custome__74AE54BC

alter table orders1
add constraint FK_Cust_ID FOREIGN KEY (customerid) references customers1(customerid)
on delete cascade
on update no action

delete from customers1 where customerid = 1

select * from customers1
select * from orders1

update customers1 
set customerid =3
where customerid =2

create table products1
(
productid int,
productname varchar(20) not null,
price int,
category varchar(30) default 'General',
constraint PK_prodid_prod primary key(productid),
constraint chk_price_prod check (price>0)
)

#### Power BI

Following measures are used:
Active Customers = CALCULATE([Total Customer], Fact_Bank_Departure[IsActiveMember] = 1)
Churn Contribution % = 
DIVIDE (
    [Exit Customers],
    CALCULATE (
        [Exit Customers],
        REMOVEFILTERS ( Dim_Geography ),
        REMOVEFILTERS ( Dim_ActiveCustomer ),
        REMOVEFILTERS ( Dim_Bal_Band )
    )
)
Churn Rate % = DIVIDE([Exit Customers],[Total Customer])
Credit Card Holders = CALCULATE([Total Customer], Dim_CreditCard[Category] = "credit card holder")
Exit Customers = CALCULATE([Total Customer], Fact_Bank_Departure[Exited]=1)
Inactive Customers = [Total Customer] - [Active Customers]
Non Credit Card Holders = CALCULATE([Total Customer], Dim_CreditCard[Category] = "non credit card holder")
Previous Month Exit Customer = CALCULATE([Exit Customers], PREVIOUSMONTH(Date_Dim[Date]))
Retain Customer = CALCULATE([Total Customer], Dim_ExitCustomer[ExitCategory]="Retain")
Total Customer = count(Fact_Bank_Departure[CustomerId])
Parameter by Churn Rate % = {
    ("Age_Category", NAMEOF('Dim_Age_Category'[Age_Category]), 0),
    ("Balance_Band", NAMEOF('Dim_Bal_Band'[Balance_Band]), 1),
    ("Credit_Score_Category", NAMEOF('Dim_Credit_Type'[Credit_Score_Category]), 2),
    ("Category", NAMEOF('Dim_CreditCard'[Category]), 3),
    ("GenderCategory", NAMEOF('Dim_Gender'[GenderCategory]), 4),
    ("GeographyLocation", NAMEOF('Dim_Geography'[GeographyLocation]), 5),
    ("Salary_Bracket", NAMEOF('Dim_Sal_Bracket'[Salary_Bracket]), 6),
    ("Tenure", NAMEOF('Fact_Bank_Departure'[Tenure]), 7),
    ("NumOfProducts", NAMEOF('Fact_Bank_Departure'[NumOfProducts]), 8)
}

#### Power BI

Visualizations Used:
1. Key Influencer Visualization
2. Decomposition Tree
3. Smart Narrative
4. Column Chart
5. Bar Chart
6. Line Chart
7. Field Parameter
8. Cards
9. Pie Chart
10. Donut Chart
11. Matrix

Features Used:
1. Slicers
2. Bookmarks, Buttons
3. RLS

<img width="721" height="405" alt="image" src="https://github.com/user-attachments/assets/cd93212f-18ba-4ed3-9101-7c1b05a74cf0" />




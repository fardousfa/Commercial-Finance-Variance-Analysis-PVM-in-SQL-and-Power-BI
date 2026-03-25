# Commercial-Finance-Variance-Analysis-PVM-in-SQL-and-Power-BI
This project uses SQL Server to prepare and structure Price-Volume-Mix revenue variance data, then connects the dataset to Power BI to build an interactive dashboard for Budget vs Actual analysis, KPI monitoring, monthly trends, and product-level insights.

```sql
WITH cte_product_summary AS(
	SELECT 
		m.MonthKey,
		m.MonthStartDate,
		s.ScenarioID,
		p.ProductID,
		p.ProductName,
		SUM(s.netrevenue) as total_revenue,
		SUM(s.units) as total_units,
		MAX(p.standardcost) as product_standard_cost,
		MAX(p.ListPrice) as sales_price
	FROM sales.FactSalesScenarioMonthly s
	JOIN MasterData.Products p
	  ON p.ProductID = s.ProductID
	JOIN MasterData.Months m
	  ON m.MonthKey = s.MonthKey
	WHERE s.ScenarioID IN (1,2) and MonthStartDate >='2025-01-01' and ProductFamily <> 'Services'
	GROUP BY 	m.MonthKey,
	        	m.MonthStartDate,
	        	s.ScenarioID,
	        	p.ProductID,
	        	p.ProductName
	)
,

cte_rev_unit AS(
	SELECT 
		MonthStartDate,
		ProductID,
		ProductName,
		product_standard_cost,
		sales_price,
		SUM(case WHEN scenarioID = 1 THEN total_revenue ELSE 0 END) as Actual_revenue,
		SUM(case WHEN scenarioID = 2 THEN total_revenue ELSE 0 END) as Budget_revenue,
		SUM(case WHEN scenarioID = 1 THEN total_units ELSE 0 END) as Actual_Units,
		SUM(case WHEN scenarioID = 2 THEN total_units ELSE 0 END) as Budget_Units
	FROM cte_product_summary
	GROUP BY 	MonthStartDate,
				ProductID,
				ProductName,
	      		product_standard_cost,
		   		sales_price
		)
	,

cte_price AS(
	SELECT
	  	MonthStartDate,
	  	ProductID,
	  	ProductName,
	  	actual_revenue,
	  	budget_revenue,
	  	Actual_Units,
	  	Budget_Units,
	  	CAST(Actual_revenue / NULLIF(Actual_Units, 0) as decimal(18,2)) as actual_price,
	  	CAST(Budget_revenue / NULLIF(Budget_Units, 0) as decimal(18,2)) as budget_price
	FROM cte_rev_unit
	)
,

cte_monthly_total_units AS(
	SELECT
		MonthStartDate,
		SUM(actual_units) as total_actual_units,
		SUM(budget_units) as total_budget_units
	FROM cte_price
	GROUP BY
			MonthStartDate
		)
,

cte_mix AS(

	SELECT 
	  	p.MonthStartDate,
	  	p.ProductID,
	  	p.ProductName,
	  	p.actual_revenue,
	  	p.budget_revenue,
	  	p.Actual_Units,
	  	p.Budget_Units,
	  	p.actual_price,
	  	p.budget_price,
	  	m.total_actual_units,
	  	m.total_budget_units,
	    CAST(p.Budget_Units *1.0 / NULLIF(m.Total_Budget_Units, 0) AS DECIMAL(18,6)) AS Budget_Mix_Pct,
	    CAST(m.Total_Actual_Units *1.0 * (p.Budget_Units *1.0 / NULLIF(m.Total_Budget_Units, 0)) AS DECIMAL(18,4)) AS           Expected_Units_At_Budget_Mix
		FROM cte_price p
		LEFT JOIN cte_monthly_total_units m
		  ON p.MonthStartDate = m.MonthStartDate
	)

SELECT
    MonthStartDate,
    ProductID,
    ProductName,
    Actual_Revenue,
    Budget_Revenue,
    Actual_Revenue - Budget_Revenue AS Revenue_Variance,
    Actual_Units,
    Budget_Units,
    Actual_Price,
    Budget_Price,
    Expected_Units_At_Budget_Mix,
    CAST((Actual_Price - Budget_Price) * Actual_Units AS DECIMAL(18,2)) AS Price_Variance,
    CAST((Expected_Units_At_Budget_Mix - Budget_Units) * Budget_Price AS DECIMAL(18,2)) AS Volume_Variance,
    CAST((Actual_Units - Expected_Units_At_Budget_Mix) * Budget_Price AS DECIMAL(18,2)) AS Mix_Variance,
    CAST(
        ((Actual_Price - Budget_Price) * Actual_Units)
      + ((Expected_Units_At_Budget_Mix - Budget_Units) * Budget_Price)
      + ((Actual_Units - Expected_Units_At_Budget_Mix) * Budget_Price)
        AS DECIMAL(18,2)
    ) AS PVM_Check
FROM cte_mix;

```

<img width="1918" height="978" alt="image" src="https://github.com/user-attachments/assets/1aa5725f-a1ea-4ecb-a39d-64427415f81d" />


## Connect SQL Server to Power BI 

<img width="1914" height="913" alt="image" src="https://github.com/user-attachments/assets/79b9c39d-bdfb-4e0d-9af5-17d9f0c96ea9" />


**-In Excel**
<img width="1117" height="493" alt="image" src="https://github.com/user-attachments/assets/a08f375a-982d-4d6f-b10b-b881ef60ff38" />




## Insight

January revenue came in below budget, driven mainly by a volume shortfall. Pricing was slightly ahead of plan and helped offset part of the miss, while mix had only a limited overall impact.
Overall, this suggests the gap is more demand / sales-execution related than a pricing issue at this point.

## Recommendation

The near-term priority should be to recover volume rather than respond with pricing action. Current price realisation looks broadly healthy, so immediate discounting would likely address the wrong issue and could create unnecessary margin pressure.
Focus should instead be on pipeline quality, conversion, deal timing, and any signs of competitive pressure.

## Actions

**Commercial**

Review whether pipeline build is tracking below plan.
Check if conversion rates are weaker than expected at any stage of the funnel.
Assess whether competitors are taking share in key products or segments.
Confirm whether deals have slipped into later months rather than being fully lost.

**Finance**

Reforecast revenue and volume assumptions if the shortfall continues for three consecutive months.
Investigate the root cause in more detail, including churn, run-rate, and order trends.
Update Q1 and full-year assumptions based on latest trading evidence.
Quantify the EBITDA impact of the volume shortfall.
Track whether this is a one-off timing issue or the start of a broader trend.

## Report Improvement Plan

This solution can be deployed in multiple ways based on business needs, scalability, and sharing requirements.

The preferred approach is to design the data model and build the report in Power BI Desktop, then publish it to the Power BI Service for enterprise distribution. This enables automated refresh scheduling, centralized report access, and stronger governance. Where the source database is hosted locally, an On-Premises Data Gateway is required to support refresh functionality. After deployment, user access and security can be controlled through Azure AD, Microsoft Entra ID, or security groups.

A secondary option is to connect SQL Server data to Excel using Power Query. This allows the analyst to perform data transformation in Power Query, build the model in Power Pivot, and create charts or graphs for reporting. While this method is suitable for offline analysis or ad hoc reporting, it does not offer the same level of collaboration, scalability, or controlled sharing as Power BI.

For this project, I performed the core analysis in SQL Server, connected the processed data to Power BI, and translated the outputs into business impact, key insights, and action-oriented recommendations. To strengthen the long-term value of the solution, I have also included an improvement plan outlining the next steps for optimization and future development.

# Google Data Analytics Certificate
This repository is part of Capstone project in the Data Analytics Professional Certificate, offered by Google on [Coursera](https://www.coursera.org/professional-certificates/google-data-analytics).


# Google Data Analytics Professional Certificate - Capstone

This is a case study in order to complete the Google Certificate. I choose the option 1, which is describing in the document available on GitHub.

The process of data analysis is composed by 6 steps:
1.   Ask
2.   Prepare
3.   Process
4.   Analyze
5.   Share
6.   Act

These steps will guide in this case study.

---
## **Ask**

The Ask phase is when the data analyst clarifies the problem that the company needs to be solved. In this study, the questions provided was the follow:

1. How do annual members and casual riders use Cyclistic bikes differently?
2. Why would casual riders buy Cyclistic annual memberships?
3. How can Cyclistic use digital media to influence casual riders to become members?


The case study assigned to me to answer the first question: How do annual members and casual riders use Cyclistic bikes differently?

To complete the taks, I will need to produce a report with the following deliverables:
1. A clear statement of the business task
2. A description of all data sources used
3. Documentation of any cleaning or manipulation of data
4. A summary of your analysis
5. Supporting visualizations and key findings
6. Your top three recommendations based on your analysis


### Ask tasks

#### Defining the problem

**Business problem**: The scenario was considered of a data analyst working in the marketing analyst team at Cyclistic, a bike-share company in Chicago. The director of maketing, Lily Moreno, believes the success of the company in the next years depends on increasing the number of annual memberships, what introduce the concept of recurring revenue to the company and help the company to better forescasting the expected results for the coming periods. In order to do this, I need to understand how casual riders can be converted to a annual members and direct the new marketing strategy to the right customers.

**Stakeholders**: The primary stakeholder are the Cyclistic executive team, what have a notoriously detail-oriented executive team, that will decide whether to approve the recommended marketing program, based on my analysis. The second stakeholder is Lily Moreno, the director of marketing and my manager. 


---
# **Prepare**

In this step I need to collect and store data that I will use for the upcoming analysis step. All datasets are available on this [link](https://divvy-tripdata.s3.amazonaws.com/index.html). For this activity, I will need to download and store in a database the previous 12 months of Cyclistic trip data. 

The datasets are made up of 12 zip files, which have a total size of 258MB. After uncompressed files, the set of csv files has 1,27GB. This is a huge amount of data to process in a spreadsheet. Then I will use a database solution.

**Database**: I have available a SQL Server 15.0.200.5 version to store the data and perform the next steps of analysis. I will upload the datasets manually in the database, but an amazing option is to use the [Machine Learning Services](https://docs.microsoft.com/en-us/sql/machine-learning/?view=sql-server-ver15), which provides a tool to run scripts Python or R inside the SQL query. With this, I can upload the data automatically to database using Python, and build an ETL pipeline to perform this activity in the future.

The option to import manually data is available after clicking with right mouse button, then Tasks - Import Flat File. 

After I imported all datasets to database, I did some verifications. The relation of tables with quantity of registers and size of tables is related below.

| TABLE_NAME				      | QT_REGISTERS	|TOTAL_SIZE_MB	|
|-------------------------|---------------|---------------|
| 202004-divvy-tripdata		| 84.776				|20,0			      |
| 202005-divvy-tripdata		| 200.274				|47,1			      |
| 202006-divvy-tripdata		| 343.005				|80,5			      |
| 202007-divvy-tripdata		| 551.480				|128,9			    |
| 202008-divvy-tripdata		| 622.361				|145,1			    |
| 202009-divvy-tripdata		| 532.958				|123,1			    |
| 202010-divvy-tripdata		| 388.653				|88,3			      |
| 202011-divvy-tripdata		| 259.716				|58,7			      |
| 202012-divvy-tripdata		| 131.573				|33,5			      |
| 202101-divvy-tripdata		| 96.834				|24,7			      |
| 202102-divvy-tripdata		| 49.622				|12,7			      |
| 202103-divvy-tripdata		| 228.496				|58,8			      |
| 202104-divvy-tripdata		| 337.230				|86,3			      |
| 202105-divvy-tripdata		| 531.633				|134,1			    |
| 202106-divvy-tripdata		| 729.595				|183,3			    |
| 202107-divvy-tripdata		| 822.410				|207,3			    |
| 202108-divvy-tripdata		| 804.352				|203,3			    |
| 202109-divvy-tripdata		| 756.147				|189,7			    |


This relation can be obtained with the following code:
```SQL
SELECT
    TABLES.NAME											 AS TABLE_NAME,
    FORMAT( PARTITIONS.rows , 'N0', 'pt-br' )               AS QT_REGISTERS, 
    FORMAT( (SUM(UNITS.total_pages) * 8)
			/CONVERT(FLOAT,1024) , 'N1', 'pt-br' )          AS TOTAL_SIZE_MB
FROM
    sys.tables            AS TABLES
INNER JOIN
    sys.indexes            AS INDEXES
ON
    TABLES.OBJECT_ID = INDEXES.object_id
INNER JOIN
    sys.partitions        AS PARTITIONS
ON
        INDEXES.object_id = PARTITIONS.OBJECT_ID
    AND INDEXES.index_id = PARTITIONS.index_id
INNER JOIN
    sys.allocation_units    AS UNITS
ON
    PARTITIONS.partition_id = UNITS.container_id
WHERE
        TABLES.NAME NOT LIKE 'dt%'
    AND TABLES.is_ms_shipped = 0
    AND INDEXES.OBJECT_ID > 255
    --AND TABLES.NAME LIKE 'nome_tabela%'
GROUP BY
    TABLES.Name, PARTITIONS.Rows
ORDER BY
    TABLES.NAME
```


### Data Schema



Each table is related of one file that was downloaded, and contains the following columns/features:
1. [ride_id]
2. [rideable_type]
3. [started_at]
4. [ended_at]
5. [start_station_name]
6. [start_station_id]
7. [end_station_name]
8. [end_station_id]
9. [start_lat]
10. [start_lng]
11. [end_lat]
12. [end_lng]
13. [member_casual]

To manipulate the data, I will concatenated these 12 tables in one final table, solve the problem of some NULL registers and estimate the possibility to separate the columns in a start schema model, in order to obtain a better storage performance.


---
# **Process**

In this phase I need to find and eliminate any errors and inaccuracies that can get in the way the results. Furthermore, I will cleaning the data, transforming it into more useful format, combining two or more datasets to make sure information more complete and removing outliers.

The first activity is creating the new table to combining the datasets, accordning the next script. I add a new column [ride_length_seconds] to verify how long was each trip and [day_of_week] wich contains the day of week based on started datetime, which can bring a new insight for analysis. To calculate these new columns, I used the formulas:
> - DATEDIFF( SECOND, [started_at], [ended_at]) 
> - DATEPART(weekday, [started_at])

I add too a [columnstore index](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-columnstore-index-transact-sql?view=sql-server-ver15) to the table, which efficiently run real-time operational analytics or to improve data compression and query performance.

```SQL
CREATE TABLE [dbo].[TB_DIVVY_TRIPDATA](
	[ride_id]				    [nvarchar](100) NULL,
	[rideable_type]			  [nvarchar](100) NULL,
	[started_at]			     [datetime2](7) NULL,
	[ended_at]				   [datetime2](7) NULL,
	[ride_length_seconds]	    [float] NULL,
	[day_of_week]			    [tinyint] NULL,
	[start_station_name]	     [nvarchar](100) NULL,
	[start_station_id]		   [nvarchar](100) NULL,
	[end_station_name]		   [nvarchar](100) NULL,
	[end_station_id]		     [nvarchar](100) NULL,
	[start_lat]				  [float] NULL,
	[start_lng]				  [float] NULL,
	[end_lat]				    [float] NULL,
	[end_lng]				    [float] NULL,
	[member_casual]			  [nvarchar](100) NULL
) ON [PRIMARY]
GO
CREATE CLUSTERED COLUMNSTORE INDEX [CCI_DIVVY_TRIPDATA] ON [dbo].[TB_DIVVY_TRIPDATA] WITH (DROP_EXISTING = OFF, COMPRESSION_DELAY = 0) ON [PRIMARY]
GO
```

The process of combining the datasets into one table is performing using the following script. I use a while statement in SQL to scroll through all tables and a GROUP BY to remove possible duplicated rows (what didn't happen according execution log). After 2020-12 the station id was change, and I need to investegate further.

```SQL
SELECT TABLE_NAME
INTO #TEMP_TABLES
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_NAME like '20%tripdata'
ORDER BY
	TABLE_NAME

WHILE EXISTS (SELECT TOP 1 TABLE_NAME FROM #TEMP_TABLES)
BEGIN
	
	DECLARE @CURRENT_TABLE AS NVARCHAR(100) = (SELECT TOP 1 TABLE_NAME FROM #TEMP_TABLES ORDER BY TABLE_NAME)

	DECLARE @SQL AS NVARCHAR(MAX) = '
	INSERT INTO [dbo].[TB_DIVVY_TRIPDATA]
		SELECT 
			[ride_id]
			,[rideable_type]
			,[started_at]
			,[ended_at]
			, DATEDIFF( SECOND, [started_at], [ended_at])	AS [ride_length_seconds]
			, DATEPART(weekday, [started_at])				AS [day_of_week]
			,[start_station_name]
			,[start_station_id]
			,[end_station_name]
			,[end_station_id]
			,[start_lat]
			,[start_lng]
			,[end_lat]
			,[end_lng]
			,[member_casual]
		FROM [GOOGLE_CAPSTONE].[dbo].['+ @CURRENT_TABLE +']
		GROUP BY
			[ride_id]
			,[rideable_type]
			,[started_at]
			,[ended_at]
			,[start_station_name]
			,[start_station_id]
			,[end_station_name]
			,[end_station_id]
			,[start_lat]
			,[start_lng]
			,[end_lat]
			,[end_lng]
			,[member_casual]
		ORDER BY
			[started_at]
	'
	EXEC(@SQL)

	DELETE FROM #TEMP_TABLES WHERE TABLE_NAME = @CURRENT_TABLE
	
END

DROP TABLE IF EXISTS #TEMP_TABLES
```

After combining the datasets with the code above, the final table having 384,2 MB of total size and 7.471.115 registers. Comparing with the with 1.825,4 MB size of all 12 tables, the use of columnstore index **reduced the total size in 78,9%**. 

| TABLE_NAME				| QT_REGISTERS			|TOTAL_SIZE_MB	|
|---------------------------|-----------------------|---------------|
| TB_DIVVY_TRIPDATA			| 7.471.115				| 384,2			|



### Cleaning Process

The dataset contains 830.565 rows with NULL values in columns:
 - [start_station_name]
 - [start_station_id]	
 - [end_station_name]	
 - [end_station_id]	
 - [end_lat]			
 - [end_lng]			

I need to address this in a new version further.


---
# **Analyse**


In Analyse phase I will using tools to transform and organize the information. Then I can draw useful conclusions, make predictions and drive informed decision making. 

I use [Microsoft Power BI](https://powerbi.microsoft.com/en-us/) to analyse data and to get some insights. This tool is like the Tableau and I have some experience working in my job. Taking in account the amount of rows, about 7.4 millions, the Power BI can easily handle with this and provides a useful tool box of visualizations.

To perform the analysis, I will do this actions:
 * Calculate the mean of ride_length
 * Calculate the max ride_length
 * Calculate the mode of day_of_week
 * Create a pivot table to quickly calculate and visualize the data, with this options:
  * Calculate the average ride_length for members and casual riders. 
of ride_length.
  * Calculate the average ride_length for users by day_of_week. 
  * Calculate the number of rides for users by day_of_week by adding Count of trip_id to Values.

The first visualization with the riders trip in the last 12 months is the following:
![image](https://user-images.githubusercontent.com/48371088/140591081-7ecc4fcc-bb9a-406e-b59a-e0f6a24aaf6a.png)

In the first view I can see some insights:
 * The mean ride length is about 24 minutes;
 * The max ride length apparently is a problem with data or a user that doesn't return the bike in the station. This issue need to be deliver and a further investigation is necessary;
 * Saturday is when the most number of rides happens;
 * The station Streeter Dr & Grand Ave is the most common started ride;
 * Normally, the majority of rides are from memberships. But this bring a problem, I don't have the atual number os membership in the company, and I need to compare this measure over time;
 * Casual clients have the longest rides. It could be explained that these clients use the bikes for tourism or a leisure activity.

**Max Ride Length**:
Looking at the longest rides, I can see that many clients use the bike for more than 1 day or 24 hours.
The first 22 longest rides are list below. 

![image](https://user-images.githubusercontent.com/48371088/140618385-d8f72abc-1410-45f4-bb1d-dca73bddd457.png)

Using SQL, I aggregate the information for rides with more than 24 hours. 
We can see in the table below that had 5158 casual clients using bikes 
in a mean ride length of 71 hours. Furthermore, member clients used bikes 
in a mean ride length of 39 hours in 627 cases of more than 1 day.
I believe this can happen when clients forgot to give back the bikes or 
if they need bike to go for home and don't had a station to return the bike around there. 

| member_casual	| number_of_rides	| mean_ride_length	|
| ------------- | --------------	| ----------------	|
| casual	| 5158			| 71,84			|
| member	| 627			| 39,3			|



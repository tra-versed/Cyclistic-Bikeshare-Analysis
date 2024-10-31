# Divvy-Bikeshare-Analysis
Increasing membership subscriptions using trip data from 2023

## Introduction
This analysis will examine a scenario posed by a fictional bikeshare company operating out of Chicago. The company, Cyclistic, allows customers to use one of 5,824 bicycles to travel between one of 692 stations scattered around Chicago. There are three pricing options: single-use passes, daily-use passes, and annual memberships. Given that annual memberships comprise the majority of Cyclistic's profits, this analysis will explore the differences between single-use and daily-use riders (refered to as casual riders) versus riders with annual memberships (referred to as members) - and determine how to convert casual riders to members. 

### The Business Task
  * How do casual riders use Cyclistic bikes differently from members? 
  * What actions can be taken to encourage casual riders to subscribe to an annual membership?

### Data Source
Although the outlined scenario is fictional, this analysis uses 2023 trip data from Divvy: (https://divvybikes.com/system-data).

### Tools Used
  * Excel - data cleaning and organization
  * BigQuery - SQL querying and data aggregation
  * Tableau - data visualization


## Data Organization and Cleaning (Excel)
### Overview
The raw data for 2023 trips were divided into twelve tables, one for each month of the year. Across all twelve tables, there were a total of 5,719,877 trips for 2023. Given the vast amount of rows, each table was cleaned individually in Excel. Each table contained the following 13 columns:
  * **ride_id** - alphanumeric ID unique to each trip
  * **rideable_type** - type of bike: classic_bike (pedal-operated), electric_bike, or docked_bike
  * **started_at** - starting datetime of trip, formatted "m/d/yyyy h:mm"
  * **ended_at** - ending datetime of trip 
  * **start_station_name** - starting station of trip (eg "Clinton St & Lake St" or "Shedd Aquarium")
  * **start_station_id** - alphanumeric ID (unique to each station) of start station
  * **end_station_name** - ending station of trip
  * **end_station_id** - alphanumeric ID of end station
  * **start_lat** - coordinates of start station latitude
  * **start_lng** - coordinates of start station longitude
  * **end_lat** - coordinates of end station latitude
  * **end_lng** - coordinates of end station longitude
  * **member_casual** - indicates rider's membership status: "member" (annual subscription) or "casual" (single-ride pass/ day-use pass)

In order to return the time elapsed and day of week for each trip for further analysis, two columns were added:
  * **ride_length** - ride length in minutes; used formula `=ROUND((D2-C2) * 1440, 2)` where D2 = **ended_at** and C2 = **started_at**, formatted as general number
  * **day_of_week** - day of week for trip; used formula `=WEEKDAY(C2)` where C2 = **started_at**, default return type

### Data Cleaning
1. First, each column in each of the twelve tables were filtered for blanks. The majority of deleted rows across all months (22-26% deleted rows for each month) were due to missing values in at least one of four columns: start_station_name, start_station_id, end_station_name, and end_station_id. Some trips were only missing one of these four columns, while others were missing all four. There were noticeably extreme ride_length values in these rows (often more than one day), prompting their deletion. 

*There was one exception to these deletions: the Feburary table contained two stations names with missing station ID's for *all* accompanying rows - these stations were presumed to be new additions (not present in January) and were not deleted. 

2. January contained several idiosyncracies not present in the other tables:
   * Deleted 127 rows missing end station latitude or longitude data
   * Deleted 5 duplicate trip_id values
     
These rows were discovered before the rows with missing station info were deleted (22% of the table), unlike the other months where the missing station info was deleted first.

3. Starting in April, every table had several rows with negative ride_length values, in which the ended_at datetime occured *before* the started_at datetime - usually less than a minute before. However, the November table contained 31 values (of the same date) in which trip times were approximately negative 50 minutes. All rows with negative ride_length values were deleted.

With every table cleaned, the monthly dated was uploaded to BigQuery to be combined and queried for analysis. There were 3,509,156 total remaining trips.

## Data Analysis (BigQuery)
### Combining the Data
The analysis would examine four metrics comparing member rider versus casual rider activity: trips per month, average trip length by month, trips per day of week, and trips by time of day. These metrics required only five columns from the original tables (including the two added columns): **ride_id**, **started_at**, **member_casual**, **ride_length**, and **day_of_week**. Since each month contained the same columns, all twelve tables were combined in BigQuery via `UNION ALL` to create a single table named 'year'.

```sql
SELECT
  'Jan' AS month,
  ride_id,
  started_at,
  member_casual,
  ride_length,
  day_of_week
FROM `bikes_2023.jan`
UNION ALL
SELECT
  `Feb` AS month,
  ride_id,
  started_at,
  member_casual,
  ride_length,
  day_of_week
FROM `bikes_2023.feb`
UNION ALL
--(continued for each month until December:)
SELECT
 'Dec' AS month,
  ride_id,
  started_at,
  member_casual,
  ride_length,
  day_of_week
FROM `bikes_2023.dec`
```
*As seen above, an additional sixth column was added to further distinguish the **month** for each row. However, extracting the month from the 'started_at' timestamps proved to be more effective for sorting the analysis results than using a three-character string. In the future, this month ID column could instead be formatted as a two-digit number (eg '01' for January), in lieu of extracting the value from timestamps.

### Querying the Data
1. First, to examine trips per month, the following query counted all trips and grouped by rider type and month:
   
   ```sql
   SELECT
     EXTRACT(month FROM started_at) AS month,
     member_casual,
     COUNT(*) AS num_trips
   FROM `bikes_2023.year`
   GROUP BY month, member_casual
   ORDER BY month
   ``` 
2. To examine average trip length per month (in minutes), this query averaged (and rounded to two decimal places) trip ride lengths, again grouped by rider type and month:

   ```sql
   SELECT
     EXTRACT(month FROM started_at) AS month,
     member_casual,
     ROUND(AVG(ride_length), 2) AS mean_ride_length
   FROM `bikes_2023.year`
   GROUP BY month, member_casual
   ```
3. To examine trips by day of week, this query counted every occurence of each day, grouped by day of week and rider type:

   ```sql
   SELECT
     day_of_week,
     member_casual,
     COUNT(day_of_week) AS num_trips
   FROM `bikes_2023.year`
   GROUP BY day_of_week, member_casual
   ORDER BY day_of_week, member_casual
   ```
4. Finally, to examine trips by start time, this query rounded all start times to the nearest hour and then counted each trip, grouped by the rounded start time and rider type:

   ```sql
   SELECT
     TIME(TIMESTAMP_TRUNC(TIMESTAMP_ADD(started_at, interval 30 minute), hour)) AS hour_started,
     member_casual,
     COUNT(*) AS num_trips
   FROM `bikes_2023.year`
   GROUP BY hour_started, member_casual
   ORDER BY hour_started, member_casual
   ```
Combining `TIMESTAMP_TRUNC` and `TIMESTAMP_ADD` in this manner effectively rounded each starting timestamp to the nearest hour. Any time at or beyond the thirty minute mark would be 'pushed' into the next hour value.

The decision to use `TIME()` instead of `EXTRACT(hour FROM ...)` was determined by requiring the starting hour to be formatted as a time (rather than an integer) when transferring the data to Tableau for visualization.
   


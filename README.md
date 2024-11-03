# Divvy-Bikeshare-Analysis
Increasing membership subscriptions using trip data from 2023

## Introduction
This analysis will examine a scenario posed by a fictional bikeshare company operating out of Chicago. The company, Cyclistic, allows customers to use one of 5,824 bicycles to travel between one of 692 stations scattered around Chicago. There are three pricing options: single-use passes, daily-use passes, and annual memberships. Given that annual memberships comprise the majority of Cyclistic's profits, this analysis will explore the differences between single-use and daily-use riders (refered to as casual riders) versus riders with annual memberships (referred to as members) - and determine how to convert casual riders to members. 

### The Business Task
  * How do casual riders use Cyclistic bikes differently from members? 
  * What actions can be taken to encourage casual riders to subscribe to an annual membership?

### Data Source
Although the outlined scenario is fictional, this analysis uses 2023 trip data from [Divvy](https://divvybikes.com/system-data).

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
The queried data was then transferred to Tableau for visualization.

## Conclusions
1. As seen below, the majority of Cyclistic bike trips in 2023 were done by members, with casual riders represented in 35% of the trips. This is important to consider for each of the following visualizations, as the quantity of member trips significantly outnumbered that of casual trips. 

<kbd>![Rider Type Distribution](https://github.com/user-attachments/assets/fe390d23-658e-425f-979c-b493444c1a09)</kbd>

---

2. This relationship can be further explored by observing member and casual trips across each month. There was an overall increase in trips during the warmer months, but with some variation between rider types. Member trips began increasing in March, peaked in August, and decreased by December. Casual trip activity was slightly more confined: increasing in April, peaked in July - one month before peak member trips - and subsided after October. 

<kbd>![Monthly Trip Totals](https://github.com/user-attachments/assets/54218976-6c5c-4244-9a62-23117aa1e326)</kbd>

---

3. Examining average monthly ride lengths for each rider type offers an additional perspective. The surge in casual rider trips around July was reflected in the surge of average ride lengths among casual riders. While member rider lengths had a slight surge, the averages were significantly more consistent throughout the year. Additionally, casual riders averaged longer trip times than members across 2023, although the sample size of casual trips used to calculate these averages was significantly smaller than member trips. 

<kbd>![Average Monthly Ride Length](https://github.com/user-attachments/assets/762d91b8-976a-4217-9bfe-e4c87245a76c)</kbd>

---

4. Comparing rider activity between each day of the week suggests differing priorities among the riders. Member riders were more active during weekdays - suggesting trip use for commuting - while casual riders were concentrated during the weekend - suggesting trip use during leisure time.

<kbd>![Trips by Day of Week](https://github.com/user-attachments/assets/00a7fc3d-d193-4acb-aa82-2f775d99c936)</kbd>

---

5. This ridership dynamic is further supported by examining the starting hour of each trip. Member riders were most active during morning and late afternoon rush hour periods, while casual riders displayed a broader surge of activity over the course of the day.

<kbd>![Trips by Time of Day(1)](https://github.com/user-attachments/assets/fb2152f4-fe52-4e11-98ef-b1a655cc4353)</kbd>

---

## Recommendations
**Our goal for this project is to convert casual riders into Cyclistic members**. Having looked at the Cyclistic (well, technically [Divvy](https://divvybikes.com/system-data) trip data from 2023, we now know:
 * Casual riders only represented 35% of all bike trips.
 * Casual ridership drastically increased in the warmer months of the year: roughly April through October, peaking in July. This timeframe is slightly narrower than member trip activity.
 * On average, casual riders used the bikes for longer durations than members.
 * Casual riders predominantly used the bikes for leisure purposes (trips concentrated in weekends and late afternoon/evening times), a completely different priority from members using the bikes for commuting.

Rephrasing that last statement, the Cyclistic membership program appears to be most popular among commuters. Purchasing an annual pass for repeat commutes resonates with the demands of commuters, while people using the bikes in their free time would only need to purchase the one-time (or day-use) pass. 

**Here are some suggested actions for reaching out to non-member riders:**
 * Advertising campaign for annual passes targetting casual riders starting in April or May, anticipating the summer surge in casual rider trips.
 * Offer two-day weekend passes, and/or offer a month-use passes - once again anticipating the casual rider summer surge. *However, since this option does not address converting annual subscribers...*
 * Annual passes have reduced fare rates on weekends (or more free ride time before the fare starts), addressing an alternative purpose for subscribing. This could be a modification to the current annual pass, or be added as a seperate option for annual subscription.

**Additionally, paths for further analysis could include:**
 * Research the conversion rate of casual riders to members. Is there a specific time of year that sees increased subscriptions? 
 * Examine any differences in membership activity, accounting for repeat trips by the same individuals. Do all members use Cyclistic for commuting, or are there repeat users weighing the data? 
   



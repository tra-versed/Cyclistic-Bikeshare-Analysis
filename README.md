# Divvy-Bikeshare-Analysis
Increasing membership subscriptions using trip data from 2023

## Introduction
This analysis will examine a scenario posed by a fictional bikeshare company operating out of Chicago. The company, Cyclistic, allows customers to use one of 5824 bicycles to travel between one of 692 stations scattered around Chicago. There are three pricing options: single-use passes, daily-use passes, and annual memberships. Given that annual memberships comprise the majority of Cyclistic's profits, this analysis will explore the differences between single-use and daily-use riders (refered to as casual riders) versus riders with annual memberships (referred to as members) - and determine how to convert casual riders to members. 

### The Business Task
  * How do casual riders use Cyclistic bikes differently from members? 
  * What actions can be taken to encourage casual riders to subscribe to an annual membership?

### Data Source
Although the outlined scenario is fictional, this analysis uses 2023 trip data from Divvy: (https://divvybikes.com/system-data).

### Tools Used
  * Excel - data cleaning and organization
  * BigQuery - SQL querying and data aggregation
  * Tableau - data visualization


## Data Organization and Cleaning
### Overview
The raw data for 2023 trips were divided into twelve tables, one for each month of the year. Across all twelve tables, there were a total of 5,719,877 trips for 2023. Because of this, each table was cleaned individually in Excel. Each table contained the following 13 columns:
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
1. First, each column in each of the twelve tables were filtered for blanks. The majority of deleted rows across all months (22-26% deleted) were due to missing values in at least one of four columns: start_station_name, start_station_id, end_station_name, and end_station_id. Some trips were only missing one of these four columns, while others were missing all four. There were noticeably extreme ride_length values in these rows (often more than one day), prompting their deletion. 

*There was one exception to these deletions: the Feburary table contained two stations names with missing station ID's for *all* accompanying rows - these stations were presumed to be new addition (not present in January) and were not deleted. 

2. January contained several idiosyncracies not present in the other tables:
   * 127 rows missing end station latitude or longitude data
   * 5 duplicate trip_id values
These rows were discovered and deleted before all of the rows with missing station info were deleted (22% of the table), unlike the other months where the missing station info was deleted first.

3. Starting in April, every table had several rows with negative ride_length values, in which the ended_at datetime occured *before* the started_at datetime - usually less than a minute before. However, the November table contained 31 values of the same date in which trip times were approximately negative 50 minutes. All rows with negative ride_length values were deleted. 


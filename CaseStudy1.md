## Cyclistic Case Study #1

#### Prompt
In 2016, Cyclistic launched a successful bike-share offering. Since then, the program has grown to a fleet of 5,824 bicycles that are
geotracked and locked into a network of 692 stations across Chicago. The bikes can be unlocked from one station and returned to
any other station in the system anytime.
Until now, Cyclistic’s marketing strategy relied on building general awareness and appealing to broad consumer segments. 

One
approach that helped make these things possible was the flexibility of its pricing plans: single-ride passes, full-day passes, and
annual memberships. Customers who purchase single-ride or full-day passes are referred to as casual riders. Customers who
purchase annual memberships are Cyclistic members.


Cyclistic’s finance analysts have concluded that annual members are much more profitable than casual riders. Although the pricing
flexibility helps Cyclistic attract more customers, Moreno believes that maximizing the number of annual members will be key to
future growth. Rather than creating a marketing campaign that targets all-new customers, Moreno believes there is a very good
chance to convert casual riders into members. She notes that casual riders are already aware of the Cyclistic program and have
chosen Cyclistic for their mobility needs.


Moreno has set a clear goal: Design marketing strategies aimed at converting casual riders into annual members. In order to do
that, however, the marketing analyst team needs to better understand how annual members and casual riders differ, why casual
riders would buy a membership, and how digital media could affect their marketing tactics. Moreno and her team are interested in
analyzing the Cyclistic historical bike trip data to identify trends.

Moreno has assigned us the first question to answer: 
*How do annual members and casual riders use Cyclistic bikes differently?*

#### Step 1: Download, Import and Load the data with Google Sheets

Starting with 2023-01 data, we download the .csv file and import it into google sheets to be manipulated. 

Once loaded, we have the following column names: 

1. ride_id
2. rideable_type
3. started_at
4. ended_at
5. start_station_name
6. start_station_id
7. end_station_name
8. end_station_id
9. start_lat
10. start_lng
11. end_lat
12. end_lng
13. member_casual


The first thing we will do is create a ride length column by subtracting the start time from end time and autofilling every row with that calculation. 

Next I created a second column for day of the week. Note: 1 = Sunday and 7 = Saturday. 

I noticed some empty cells in the data as well, so I will replace those with NULL using the built in Find and Replace tool within Google sheets. 
Next, I removed duplicates, in this case there was only 1 duplicate. 

Ok some light cleaning aside, we are on to the next step. 

### Step 2: Analysis with Spreadsheets

Some questions we may ask to help us answer how each member type uses the bike service differently: 

* What is the average ride length for each member type?
* Which bike type does either member use more frequently?
* What is the max or min ride length for each member type? 
* How does each day of the week's ridership compare? 

We'll begin by doing some quick calculations in a separate sheet to get a read on the data. We'll look at average ride length, max & min ride length and mode of the day of the week - all separated by rider type (i.e. Member vs. Casual).

Starting with the average ride lengths using the AVERAGEIF() function I calculated the average ride lengths and found an interesting statistic. 

* Member avg ride length = 10:22
* Casual avg ride length = 22:55

Next I looked at max ride lengths and discovered that the maximum ride length for a casual viewer was 560 hours... This led me to sort the entire table by ride length descending and found 38 instances where casual riders rode for over 25 hours or more. 

For the purposes of this data analayis I will exclude any rides over 3 hours under the assumption that these are outlier scenarios and to give us cleaner data on daily useage.

I also calculated mode day of the week and found Tuesday to be the most common occuring day of the week Cyclistic bike users rode bikes. 

Last I created a pivot table showing the Ride length average and number of rides per day of the week for each member type. I then created two separate bar charts. The first compares the Ride length of each member type across the days of the week. The other compares the ride count of each member type across the days of the week. 

It is clear from this spreadsheet analysis that casuals spend about twice as much time on the bike per ride BUT members conduct nearly four times as many rides as casuals. 

Now I will do all of the above for the entire year (2023).

Due to size constraints, it's time now to switch to using SQL so that we can download all 11 months of available 2023 Cyclistic bike data and perform the same analysis to see if our observations hold true. 

### Step 3: Data Import with SQL 

First, I'll open up pgAdmin4 and PostGreSQL to create a database and the first table within the database. 

Once I have inputted all of the correct column names and imported the .csv the first time I will be able to import the other 10 .csv files into their own respective tables. 

I have made `ride_id` the primary key. 

Now that the first months' data is imported, I will import the other 10 .csv files into their own respective tables. 

Once all of the tables are created, I will put together a large Union query to combine all 11 tables and store that as a `View = bike_data`.

```sql
SELECT * FROM cyclistic_data_2023_01
UNION
SELECT * FROM cyclistic_data_2023_02
...
```

This query took 42 seconds to run and resulted in 5,495,804 rows. 

Next, I will store all of this into a new permanent table called bike_2023 using the following query code: 

```sql
CREATE TABLE bike_2023 AS 
SELECT
	*,
	(ended_at - started_at) AS ride_length
FROM bike_data
```
This also added a column called `ride_length`, just like the one we created in our spreadsheet application. 

Let's quickly count the number of rows: 

```sql
SELECT 
	COUNT(*)
FROM bike_2023
```
The result is 5,495,804 rows.

<!-- Once I have this I will create two separate views for members and casuals: 

* bike_members (3,488,297 rows)
* bike_casuals (2,007,507 rows)

We can use the count() function in the SELECT statement to verify the number of rows is correct. 

The thought process with now creating two separate views is that I can begin querying for summarizing key statistics, such as average ride lengths, without querying the entire database.  -->

Now that I have a new table with all of the data from 2023 plus a ride_length column it's time to do a few more data processing steps to ensure our analysis is as accurate as possible. These steps include: 

- Searching for and eliminating duplicates.
- Searching for and eliminating rows with null values. 
- Create new rows for day of the week and month for easier future analysis.

Let's start with duplicates:

```sql
SELECT
	ride_id,
	COUNT(ride_id)
FROM bike_2023
GROUP BY (ride_id)
HAVING 
	COUNT(ride_id) > 1
```

This results in zero duplicates, so we can move onto rows with null values. 

Strictly using SQL and PGAdmin4 the simplest way to find this is to run the following script, then substituting and re-running again the `ride_id IS NULL` with the next column name until we get a match: 

```sql
SELECT
	ride_id,
	rideable_type,
	started_at,
	ended_at,
	start_station_name,
	start_station_id,
	end_station_name,
	end_station_id,
	start_lat,
	start_lng,
	end_lat,
	end_lng,
	member_casual
FROM bike_2023
WHERE
	ride_id IS NULL
```
With each iteration, we find that:

-  840,006 rows do not have a `start_station_name` value. 
- 840,138 rows do not have a `start_station)id` value.
- 891,278 rows do not have an `end_station_name` value.
- 891,419 rows do not have an `end_station_id` value. 
- 6,751 rows do not have an `end_lat` or `end_lng` value. 

For higher data integrity, we need to remove these rows from the table. 

I will count the # of rows we will be removing first:

```sql
SELECT 
	COUNT(*)
FROM bike_2023
WHERE
	start_station_name IS NULL OR
	start_station_id IS NULL OR
	end_station_name IS NULL OR
	end_station_id IS NULL OR
	end_lat IS NULL OR
	end_lng IS NULL;
	
```

The result is 1,331,240 rows that contain a NULL in any of those columns. 

Before I delete 1/5 of the data, I will save a backup .csv to my hard drive.  I then proceed to delete: 

```sql 
DELETE 
FROM bike_2023
WHERE
	start_station_name IS NULL OR
	start_station_id IS NULL OR
	end_station_name IS NULL OR
	end_station_id IS NULL OR
	end_lat IS NULL OR
	end_lng IS NULL;
```
Query returned successfully in 25 secs 362 msec and deleted 1,331,240 rows. 

Let's do a quick count to make sure: 

```sql
SELECT 
	COUNT(*)
FROM bike_2023
```
There are 4,164,564 remaining rows. 

Last in the process phase I would like to create a column for month and another for Day of the Week.
```sql
ALTER TABLE bike_2023
	ADD COLUMN ride_month int
```

```sql
UPDATE bike_2023 
	SET ride_month = date_part('month',started_at::timestamptz)
```
This created a column called `ride_month` that stores the month of the `started_at` value as an integer for each month of the year. 

```sql
ALTER TABLE bike_2023
	ADD COLUMN day_of_week varchar
```
```sql
UPDATE bike_2023 
	SET day_of_week = to_char(started_at, 'Day')
```
This created a column `day_of_week` that stores the week date of the `started_at` value as the day of the week. 


**Beautiful**, we now have a cleaned and updated data table for future analysis. 


### Step 4: Analysis with SQL 

Recall, the prompt is how annual members and casual riders use Cyclistic bikes differently. So let's start with a simple summary query finding the average ride length and count of each member types rides.

```sql
SELECT
	AVG(ride_length) as avg_ride_length,
	COUNT(ride_id) as num_of_rides,
	member_casual
FROM bike_2023
WHERE
	ride_length BETWEEN '00:01:00' AND '03:00:00'
GROUP BY
	member_casual
```
Below is the output: 

| avg_ride_length | num_of_rides | member_casual |
| --- | --- | --- |
| 00:20:42 | 1460023 | casual |
| 00:11:57 | 2608443 | member |

Next let's view a summary where each month is summarized by ride counts for each member type. 

```sql
SELECT
	COUNT(ride_id) as num_of_rides,
	member_casual,
	ride_month
FROM bike_2023
WHERE
	ride_length BETWEEN '00:01:00' AND '03:00:00'
GROUP BY
	member_casual,
	ride_month
ORDER BY 
	ride_month
```
For brevity, I have included the first two months in the output below:

| num_of_rides | member_casual | ride_month |
| --- | --- | --- |
| 28897 | casual | 1 |
| 115333 | member | 1 |
| 31990 | casual | 2 |
| 113592 | member | 2 |

Now let's do the same but subsitute `num_of_rides` with `AVG(ride_length)`

```sql
SELECT
	AVG(ride_length),
	member_casual,
	ride_month
FROM bike_2023
WHERE
	ride_length BETWEEN '00:01:00' AND '03:00:00'
GROUP BY
	member_casual,
	ride_month
ORDER BY 
	ride_month
```
For brevity, I have included the first two months in the output below:

| avg ride length | member_casual | ride_month |
| --- | --- | --- |
| 00:12:51 | casual | 1 |
| 00:09:47 | member | 1 |
| 00:15:29 | casual | 2 |
| 00:10:09 | member | 2 |

Last we will analyze the difference in days of the week for both `AVG(ride_length)` and `num_of_rides`

```sql
SELECT
	AVG(ride_length),
	COUNT(ride_id) as num_of_rides,
	member_casual,
	day_of_week
FROM bike_2023
WHERE
	ride_length BETWEEN '00:01:00' AND '03:00:00'
GROUP BY
	member_casual,
	day_of_week
ORDER BY 
	day_of_week 
```
| avg ride length | num_of_rides |member_casual | day_of_week |
| --- | --- | --- | --- |
| 00:23:57 | 243161 |casual | Sunday |
| 00:13:26 | 287455 |member | Sunday |
| 00:20:28 | 167887 |casual | Monday |
| 00:11:25 | 361595 |member | Monday |



### Step 5: Visualization with Tableau

The final step in analysis was importing the data into Tableau for visualization. I created four visualizations: 

1. Ride Count (Member vs. Casual) throughout 2023
2. Ride Count (Member vs. Casual) Day of the Week
3. Average Ride Length (Member vs. Casual) for 2023
4. Preferred Ride Types (Member vs. Casual) Monthly

Starting with visualizing the difference in ride count across all 11 months of available data it is is clear that Members typically double or even triple the number of rides they embark on compared to casual users. 

Key insights: 

- Ridership peaks in the Summer months for both user groups.
- Members double casual ridership in Summer. 
- Members ridership peaks in August whereas Casuals ridership peaks in July. 
- In the winter months, Members nearly triple useage over Casuals. 

![viz1](https://github.com/gunnarconley/capstone-1/blob/main/viz1.png?raw=true)

After charting the average ridership across the 7 days of the week, we see a clear trend where ride count encroaches on being identical over the weekend and then widens during the week, plummiting for casuals on Mondays before steadily increasing as the week progresses. 

![viz2](https://github.com/gunnarconley/capstone-1/blob/main/viz2.png?raw=true)

Ride Length tells a different story... 

It is clear that Casual riders spend nearly double the amount of time on their bikes as Members do. 

- Ride Length is relatively flat for the year for Members. 
- Ride Length peaks in the Summer months and slowly falls as we enter Winter. 

![viz3](https://github.com/gunnarconley/capstone-1/blob/main/viz3.png?raw=true)

Lastly, preferred ride types follow basically the same trend thoughout the year. The larger ride numbers are inflated by the higher number of rides the Member class goes on. 

![viz4](https://github.com/gunnarconley/capstone-1/blob/main/viz4.png?raw=true)

[View the full visualization on Tableau's website here](https://public.tableau.com/views/CyclisticBike-ShareMembervs_CasualVisualization/Dashboard1?:language=en-US&:display_count=n&:origin=viz_share_link) 



### Conclusions

It seems that Casual riders conduct 50-70% less rides but ride for significantly longer than Members do. Members ride nearly 3x as much but for much shorter trips. 

Per the data, I have the following recommendations for the Cyclistic Team: 

1. Determine a pricing model that incentivizes Casuals to convert to Annual Members such as a "punch-card" style promotion where a user could earn a discount toward their membership for completing X number of rides.
2. Spend more of the marketing and advertising budget on promoting Casual users during the Winter months (November-March).
2. Reward Members for embarking on longer rides via ridership credits or discounts they can share with their friends to become Members. 




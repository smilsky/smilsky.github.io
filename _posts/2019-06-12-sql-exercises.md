---
title: "SQL Exercises"
date: 2019-06-12
tags: [data analytics]
header:
  image: "/images/SQL/SQL.jpg"
excerpt: "SQL, practice queries"
---
This is an example of the SQL querying that I have done in the past. The following queries are written using BigQuery syntax, since the databases are the ones that are publicly available on BigQuery.

### bigquery-public-data.hacker_news.comments
As taken from the BigQuery website, "this dataset contains all stories and comments from Hacker News from its launch in 2006 to present. Each story contains a story ID, the author that made the post, when it was written, and the number of points the story received."

*Q1. What are the top 100 authors by the number of comments?*
```SQL
    SELECT author, COUNT(*) FROM `bigquery-public-data.hacker_news.comments`
    GROUP BY author
    ORDER BY COUNT(*) DESC
    LIMIT 100;
```
<img src="/images/SQL/1.JPG" alt="">

*Q2. Which authors had least 10% year-over-year difference of annual number of comments at least once for all time available in dataset?*
```SQL
    SELECT t3.author
    FROM (
    SELECT t1.author AS Author, t2.extracted_year as Year_1, t2.num_of_comments AS Year_1_Comments, t1.extracted_year AS Year_2, t1.num_of_comments AS Year_2_Comments,
       CASE WHEN t2.extracted_year IS NOT NULL THEN (t1.num_of_comments - t2.num_of_comments) / t2.num_of_comments ELSE NULL END AS diff
    FROM
    (
        SELECT author, EXTRACT(YEAR FROM time_ts) AS extracted_year, COUNT(*) AS num_of_comments
        FROM `bigquery-public-data.hacker_news.comments`
        GROUP BY author, extracted_year
    ) t1
    LEFT JOIN
    (
        SELECT author, EXTRACT(YEAR FROM time_ts) AS extracted_year, COUNT(*) AS num_of_comments
        FROM `bigquery-public-data.hacker_news.comments`
        GROUP BY author, extracted_year
    ) t2
    ON t1.author = t2.author AND t2.extracted_year = t1.extracted_year - 1
    WHERE (t1.num_of_comments - t2.num_of_comments) / t2.num_of_comments >= 0.1
    ) t3
    GROUP BY t3.author
    HAVING COUNT(*) >=1
```
<img src="/images/SQL/2.jpg" alt="">

### bigquery-public-data.noaa_gsod.stations and bigquery-public-data.noaa_gsod.gsod2015
From the website: "This public dataset was created by the National Oceanic and Atmospheric Administration (NOAA) and includes global data obtained from the USAF Climatology Center. This dataset covers GSOD data between 1929 and present (updated daily), collected from over 9000 stations.

Global summary of the day is comprised of a dozen daily averages computed from global hourly station data. Daily weather elements include mean values of: temperature, dew point temperature, sea level pressure, station pressure, visibility, and wind speed plus maximum and minimum temperature, maximum sustained wind speed and maximum gust, precipitation amount, snow depth, and weather indicators. With the exception of U.S. stations, 24-hour periods are based upon UTC times."

*Q1. Which daily average temperature did MEADOW LAKE AIRPORT station record on the 4th of July 2015 in the state of Colorado?*
```SQL
SELECT a.temp
FROM `bigquery-public-data.noaa_gsod.stations` b INNER JOIN  `bigquery-public-data.noaa_gsod.gsod2015` a
ON a.stn = b.usaf  /* inner join by the ID of the stations */
WHERE b.name = 'MEADOW LAKE AIRPORT' AND b.state = 'CO' AND a.year = '2015' AND a.mo = '07' AND a.da = '04' /* setting the conditions */
```
<img src="/images/SQL/3.jpg" alt="">

*Q2. Have there been such days in Minnesota when none of the stations recorded either thunder or fog? If yes, which days did it happen?*
```SQL
SELECT a.mo, a.da
FROM `bigquery-public-data.noaa_gsod.stations` b INNER JOIN  `bigquery-public-data.noaa_gsod.gsod2015` a /* Знову іннер джойн*/
ON a.stn = b.usaf
WHERE b.state = 'MN'
GROUP BY a.mo, a.da
HAVING SUM(CAST(a.fog AS INT64)) = 0 AND SUM(CAST(a.thunder AS INT64)) = 0 /* Since thunder and fog columns are binary (but formated as characters), first they need to be converted, and then added up. If the totals are 0, there has been no thunder or fog. */
```
<img src="/images/SQL/4.jpg" alt="">

*Q3. For every calendar month, find out which Tuesday of the month was the hottest in Florida. Count how many stations recorded fog in those days.*
```SQL
SELECT COUNT(DISTINCT e.stn)
FROM `bigquery-public-data.noaa_gsod.gsod2015` e INNER JOIN `bigquery-public-data.noaa_gsod.stations` f ON e.stn = f.usaf
WHERE e.fog = '1' AND f.state = 'FL' AND
DATE(CAST(year AS INT64), CAST(mo AS INT64), CAST(da AS INT64)) IN (
SELECT d.`Date` FROM /* For convenience, I created a columns for the date and selected those dates which belong to the hottest Tuesdays – there are 12 of them, one for each month */
(
SELECT mo, MAX(Average) AS `Highest` FROM /* selecting month and the highest temperature value, which is received from the next subquery */
(
SELECT a.mo, a.da,
  EXTRACT(DAYOFWEEK FROM DATE(CAST(year AS INT64), CAST(mo AS INT64), CAST(da AS INT64))) AS `Weekday`, /*creating a column for the days of the week */
  EXTRACT(WEEK FROM DATE(CAST(year AS INT64), CAST(mo AS INT64), CAST(da AS INT64))) AS `Week`, /* extracting numbers of weeks, for identifying Tuesdays */
  DATE(CAST(year AS INT64), CAST(mo AS INT64), CAST(da AS INT64)) AS `Date`, /* creating a column for the date with the same purpose */
  ROUND(AVG(a.temp), 7) AS `Average` /* finding the average temperature */
FROM `bigquery-public-data.noaa_gsod.gsod2015` a INNER JOIN `bigquery-public-data.noaa_gsod.stations` b
ON a.stn = b.usaf
WHERE
EXTRACT(DAYOFWEEK FROM DATE(CAST(year AS INT64), CAST(mo AS INT64), CAST(da AS INT64))) = 2 /* extracting Tuesdays */
AND b.state = 'FL'
GROUP BY a.mo, a.da, `Weekday`, `Week`, `Date`
ORDER BY a.mo, a.da ASC
)
GROUP BY mo /* grouping by months, in order to have the highest average temperature for each month */
) c, /* іменуємо дане сабквері */
(
SELECT a.mo, a.da, /* the same subquery, but this time extracting only the date and the week number */
  EXTRACT(DAYOFWEEK FROM DATE(CAST(year AS INT64), CAST(mo AS INT64), CAST(da AS INT64))) AS `Weekday`,
  EXTRACT(WEEK FROM DATE(CAST(year AS INT64), CAST(mo AS INT64), CAST(da AS INT64))) AS `Week`,
  DATE(CAST(year AS INT64), CAST(mo AS INT64), CAST(da AS INT64)) AS `Date`,
  ROUND(AVG(a.temp), 7) AS `Average`
FROM `bigquery-public-data.noaa_gsod.gsod2015` a INNER JOIN `bigquery-public-data.noaa_gsod.stations` b
ON a.stn = b.usaf
WHERE
EXTRACT(DAYOFWEEK FROM DATE(CAST(year AS INT64), CAST(mo AS INT64), CAST(da AS INT64))) = 2
AND b.state = 'FL'
GROUP BY a.mo, a.da, `Weekday`, `Week`, `Date`
ORDER BY a.mo, a.da ASC
) d WHERE c.Highest = d.Average /* through inner join */
)
```
<img src="/images/SQL/5.jpg" alt="">

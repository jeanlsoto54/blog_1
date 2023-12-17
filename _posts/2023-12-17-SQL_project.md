# SQL Exercise on Chinook Database 

In this post, it will be explained a practical case about a database and how with the help with SQL language can be retrieved information in a relational database.

## About the database 

The Chinook data model possess data about artists, albums, media tracks, invoices, and customers from an Apple iTunes Library. Sensitive data already had been changed and now it can be used for academical porpouses.

![image_database!](/images/SQL/img1.png "relational database")


## Data cleaning

In this first part, it is going to be check how consistent the data is. It can happen when we are working on databases that there are problems with incorrect values as:
+ null values in an obligatory column
+ field defined as number that is containg text
+ characters not supported on a report and many more

At first we are going to study how the duration of the film has gone through the years. 
For it is going to be check firstly in the table metadata that there is no null values or missing values with the commands `IS NULL` and `like ""`. 



 ```sql
    Select movie_title, duration, title_year
    From movies.metadata
    WHERE movie_title IS NULL or duration like "" or title_year like "";
 ```

![image_database2!](/images/SQL/img2.png "result of blank values metadata")

It exist null values! So now it is going to be show the impact of the blank values in the metadata table

 ```sql
 Select count(*) as Total_records, 
--this new variable 'Total_missing_records'  will calculate how many rows are in blank
count(CASE WHEN movie_title IS NULL OR duration LIKE "" OR title_year LIKE "" THEN movie_title end) as Total_missing_records, 
-- 'Ratio' is going to calculate the ratio (%) of weight of missing values in the table
count(CASE WHEN movie_title IS NULL OR duration LIKE "" OR title_year LIKE "" THEN movie_title end)/count(*) * 100 as Ratio
From movies.metadata;
 ```

 The result is that 128 (2.5%) of the rows apply to this issue.

![image_database3!](/images/SQL/img3.png " ")
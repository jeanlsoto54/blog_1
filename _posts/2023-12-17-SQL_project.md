# SQL Exercise on Chinook Database 

In this post, it will be explained a practical case about a database and how with the help with SQL language can be retrieved information in a relational database.

Content's table:

1. TOC
{:toc}

## About the database 

The Chinook data model possess data about artists, albums, media tracks, invoices, and customers from an Apple iTunes Library. Sensitive data already had been changed and now it can be used for academical porpouses.

![image_database!](/images/SQL/img1.png "relational database")


## Data cleaning

In this first part, it is going to be check how consistent the data is. It can happen when we are working on databases that there are problems with incorrect values as:
+ null values in an obligatory column
+ field defined as number that is containg text
+ characters not supported on a report and many more

### Null Values

At first we are going to study how the duration of the film has gone through the years. 
For it is going to be check firstly in the table metadata that there is no null values or missing values with the commands `IS NULL` and `like ""`. 



 ```sql
    Select movie_title, duration, title_year
    From movies.metadata
    WHERE movie_title IS NULL or duration like "" or title_year like "";
 ```

![image_database2!](/images/SQL/img2.PNG "result of blank values metadata")

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

![image_database3!](/images/SQL/img3.PNG " ")


Now that the measurament about the blank values is known and we can validate that the impact is not important. We take the decision to keep out these rows on the query. It will selected the columns that are going to be used later as `movie_title`, `duration` and `title_years`

 ```sql
SELECT movie_title, duration, title_year
FROM movies.metadata
WHERE movie_title IS NOT NULL AND duration <> '' AND title_year <> '';
```

![image_database4!](/images/SQL/img4.PNG " ")


### Duplicate Values

Now that we have a dataset clean of null values, it is also necessary to check if the title of the movies, the duration of them and the year that were published present no repetition in the dataset. To catch this case, the next query is proposed on the metadata table. 

 ```sql
 -- In this line is defined the columns to fetch
 SELECT movie_title, duration, title_year, director_name, 
 -- 'Unique_values' is going to count the number of rows per group of duration, title year and director name
 COUNT(*) as Unique_values
FROM movies.metadata
-- In the filtering section it is defined to not keep the blank values
WHERE movie_title <> '' AND duration <> '' AND title_year <> ''
-- since we have define a calculation field (Unique values) the data needs to be group by the descriptive fields
GROUP BY movie_title, duration, title_year, director_name
-- Select the groups of movie title, duration and more that have more that one row
HAVING Unique_values <> 1
-- sorted by the highest values of 'Unique_values'
ORDER BY Unique_values DESC;
 ```

 ![image_database5!](/images/SQL/img5.PNG " ")


 Let's dimention how many cases of duplicated rows does the table have. For it, the next query is proposed. Also, to mention that in this query is introduced the concept of float tables with the command `WITH query_name AS ()` which let us to run inside queries in the main query that can be used in the final step of the query. 

```sql
--Create the subquery called DUPLICATES, that will have the number of duplicated rows per movie title
 WITH DUPLICATES AS (
-- Logic previously defined to know the number of duplicated rows
                        SELECT movie_title, duration, title_year, director_name, COUNT(*) as Unique_values
                        FROM movies.metadata
                        WHERE movie_title <> '' AND duration <> '' AND title_year <> ''
                        GROUP BY movie_title, duration, title_year, director_name
                        HAVING Unique_values <> 1
                        ORDER BY Unique_values DESC
                    )

SELECT 
    -- Sum all the values of 'Unique values' for all the group of movies that have duplicated rows
    (SELECT SUM(Unique_values) FROM DUPLICATES) / 
    -- Sum of all the rows that the table metadata have 
    (SELECT COUNT(*) FROM movies.metadata WHERE movie_title <> '' AND duration <> '' AND title_year <> '') * 100
    AS Proportion_of_total_records;
```
Around 4.8% of all the rows present duplicate issues.

![image_database6!](/images/SQL/img6.PNG " ")


Now, the maing query that we are using to fetch the values is needed to also take care of the duplicate values. 

```sql

-- RankedMovies is a query which will have an index that enumerates per group of the fields that we need to fetch   
WITH RankedMovies AS 
(
        SELECT *, 
        -- This command creates the index which has two rules:
        --1 rule the index will happen for each group of movie_title,duration, title_year and direction different.
        --2 rule the index will enumerate by the alphabetical order of the movie title in each group
           ROW_NUMBER() OVER (PARTITION BY movie_title, duration, title_year, director_name ORDER BY movie_title) as Enumeration 
        FROM movies.metadata
        -- Filter that get rid off of the blank values
        WHERE movie_title <> '' AND duration <> '' AND title_year <> ''
)

SELECT movie_title, duration, title_year, director_name
--query consult on the previous query RankedMovies
FROM RankedMovies
-- Only will kept the rows where the index(Enumeration) is 1 (implicitly keeping only the not duplicate rows)
WHERE Enumeration = 1;
```



### Incongruent Data

The check on the data related issues has finalized. However, is still need to do some evalution on the data by a business side standpoint. 
The table Metadata has relevant data about views, budget, costs, released year of the movies. It is going to be revised each column if they present some incongruent data.

In the next query it is being run a descriptive statistical analysis over the numerical fields


```sql
SELECT 'Duration' as field,MIN(duration) as min, MAX(duration) as max, AVG(duration) as mean FROM movies.metadata
UNION
SELECT 'director_facebook_likes' as field, MIN(director_facebook_likes) as min, MAX(director_facebook_likes) as min, AVG(director_facebook_likes)  as mean FROM movies.metadata
UNION
SELECT 'cast_total_facebook_likes' as field,MIN(cast_total_facebook_likes) as min, MAX(cast_total_facebook_likes)as min, AVG(cast_total_facebook_likes) as mean  FROM movies.metadata
UNION
SELECT 'title_year' as field, MIN(title_year) as min , MAX(title_year) as max, AVG(title_year) as mean FROM movies.metadata
UNION
SELECT 'num_voted_users' as field, MIN(num_voted_users) as min, MAX(num_voted_users) as max, AVG(num_voted_users) as mean FROM movies.metadata;
```

The results on the image give us some concernings about some fields:
+ the field `title year` should be a numerical field but it is containing string values as 'USA'
+ the field `num_voted_users` also has the same problem

![image_database7!](/images/SQL/img7.PNG " ")



Also, it is going to be check the categorical fields. In the case of  `content_rating` the next query is proposed to show if exists any other value different that the ones that the labels that the regulatory provides.

```sql
SELECT content_rating
FROM movies.metadata
WHERE 
content_rating NOT IN ('PG-13', 'PG', 'G', 'R', 'TV-14', 'TV-PG', 'TV-MA', 'TV-G', 'Not Rated', 'Unrated', 'TV-Y', 'TV-Y7');
```
Which as we can see in the result, exists some inconsistent data:
![image_database8!](/images/SQL/img8.PNG " ")


In the next query presents a summary of all the incongruent data that are in the table 

```sql
SELECT title_year, duration, gross, num_voted_users, budget,actor_3_facebook_likes,facenumber_in_poster,imdb_score, country
,content_rating, language, num_user_for_reviews, movie_imdb_link
FROM movies.metadata 
WHERE ( 
        -- Many of the numerical fields have not been defined as integer or float values. So in here  is being defined
        -- for each numerical field is there is any string outside the range of 0 and 9
        title_year REGEXP '^[^0-9]+$' OR duration REGEXP '^[^0-9]+$' OR num_voted_users REGEXP '^[^0-9]+$'
        OR budget REGEXP '^[^0-9]+$' OR title_year REGEXP '^[^0-9]+$' OR actor_3_facebook_likes REGEXP '^[^0-9]+$' 
        OR facenumber_in_poster REGEXP '^[^0-9]+$' OR imdb_score REGEXP '^[^0-9]+$' OR movie_imdb_link REGEXP '[^0-9]+$'
        OR num_user_for_reviews REGEXP '^[^0-9]+$' OR country REGEXP '[0-9]' OR language REGEXP '[0-9]' 
        -- Also, the movies have been create in the 20th century, if there is any case of a movie before this century should be selected
        OR title_year NOT BETWEEN 1900 AND 2023
        -- the ImDB scores that are outside the range of 1 and 10
        OR imdb_score NOT BETWEEN 1 AND 10
        -- Some categorical fields might have some https links, which is an error
        OR country LIKE "https%" OR language LIKE "https%" OR num_user_for_reviews LIKE "https%"
        -- Other values that do not belong to the field content_rating
        OR content_rating NOT IN ('PG-13', 'PG', 'G', 'R', 'TV-14', 'TV-PG', 'TV-MA', 'TV-G', 'Not Rated', 'Unrated', 'TV-Y', 'TV-Y7')
       )
       -- take out the blank fields
        AND movie_title <> '' AND duration <> '' AND title_year <> '';
```

![image_database9!](/images/SQL/img9.PNG " ")


Also, it is calculated the impact of the incongruent data that is on the table with the next query and the result
find is that is about 8% :

```sql
SELECT 
    (
        SELECT COUNT(*)
        FROM movies.metadata
        WHERE (title_year REGEXP '^[^0-9]+$' 
               OR duration REGEXP '^[^0-9]+$' 
               OR num_voted_users REGEXP '^[^0-9]+$'
               OR budget REGEXP '^[^0-9]+$' 
               OR actor_3_facebook_likes REGEXP '^[^0-9]+$' 
               OR facenumber_in_poster REGEXP '^[^0-9]+$' 
               OR imdb_score REGEXP '^[^0-9]+$' 
               OR num_user_for_reviews REGEXP '^[^0-9]+$'  
               OR movie_imdb_link REGEXP '[^0-9]+$'
               OR title_year NOT BETWEEN 1900 AND 2023
               OR imdb_score NOT BETWEEN 1 AND 10
               OR country REGEXP '[0-9]' 
               OR language REGEXP '[0-9]' 
               OR country LIKE "https%" 
               OR language LIKE "https%" 
               OR num_user_for_reviews LIKE "https%"
               OR content_rating NOT IN ('PG-13', 'PG', 'G', 'R', 'TV-14', 'TV-PG', 'TV-MA', 'TV-G', 'Not Rated', 'Unrated', 'TV-Y', 'TV-Y7')
              )
              AND movie_title <> '' 
              AND duration <> '' 
              AND title_year <> ''
    ) 
    / 
    (SELECT COUNT(*) FROM movies.metadata WHERE movie_title <> '' AND duration <> '' AND title_year <> '') 
    * 100 AS Proportion_of_total_records_percentage;
```

![image_database10!](/images/SQL/img10.PNG " ")


Now that we know the dimention of the incongruent data, we are going to stack the logic to get rid off these fields,along with the blank rows and duplicates in the final query to fecth the duration of the films through the years.


```sql
WITH 
-- RankedMovies has the table with index per the fields in examination and not keep the blank values
RankedMovies AS 
(
    SELECT *, 
           ROW_NUMBER() OVER (PARTITION BY movie_title, duration, title_year, director_name ORDER BY movie_title) as Enumeration 
    FROM movies.metadata
    WHERE movie_title <> '' AND duration <> '' AND title_year <> ''
)
,
-- In FINAL query it is keep only the non duplicate values
FINAL AS 
(
    SELECT * FROM RankedMovies WHERE Enumeration = 1
)

SELECT *
FROM FINAL
-- In this filter is taken out of the table all the incongruent cases
WHERE NOT (
    title_year REGEXP '^[^0-9]+$' 
    OR num_voted_users REGEXP '^[^0-9]+$' 
    OR budget REGEXP '^[^0-9]+$' 
    OR actor_3_facebook_likes REGEXP '^[^0-9]+$' 
    OR facenumber_in_poster REGEXP '^[^0-9]+$' 
    OR imdb_score REGEXP '^[^0-9]+$' 
    OR num_user_for_reviews REGEXP '^[^0-9]+$' 
    OR NOT (title_year BETWEEN 1900 AND 2023)
    OR NOT (imdb_score BETWEEN 1 AND 10)
    OR country REGEXP '[0-9]' 
    OR language REGEXP '[0-9]'
    OR country LIKE 'https%' 
    OR language LIKE 'https%' 
    OR content_rating NOT IN ('PG-13', 'PG', 'G', 'R', 'TV-14', 'TV-PG', 'TV-MA', 'TV-G', 'Not Rated', 'Unrated', 'TV-Y', 'TV-Y7')
);
```

## Joining tables

Now another task that we want to achieve is to include more data about the films scores in the query defined. To do it, we need to consult the next two tables:
+ ratings: Contains the overall raiting provided in the internet (have the fields `movieId` and `movie`)
+ metadata: is the current table that we are using and have the ImDB score (have the field `movie`)
+ movie: Contain the titles of the movies (have the fields `movieId` and `movie`) 

A problem that we have in this case is that we can not make a direct joining between the tables metadata and ratings because there is no cardinality between them (all vs all) and also there is no a certified key (movieId) in both tables.

To solve this issue the next steps are going to be regarded:
1. Define the table movie as a bridge between both ratings and metadata table since is the table that have  the certified key `movieId` and the key to be created `movie`
2. Standarize the field `movie` so it can be used as primary key
3. Reduce the dimention of the table ratings since it have a lot of ratings per movies
4. join all the refined tables.


### Standarize the primary keys

In the next query is an extract on how to standarize the format of the field `movie` in movies table,
which in this case is called as **title**.


```sql
-- trim: get rid off the spaces at the beginning and end of the value
-- lower: the value is put all in minuscule 
-- substr: the key movie in this case have the title and the year, to standarize it is being cut the last 6 characters of the field which contains the year
SELECT lower(trim( substr(title, 1, CHAR_LENGTH(title) -6))) AS ExtractedTitle
FROM movies.movies;
```


In the table metadata, when it was checked the field `movie` even though seem that only have string values, it was discovered that there is a hidden character  non ASCII in the last position of the field and is needed to adapt it.


```sql
SELECT lower(trim(substr(movie_title, 1, CHAR_LENGTH(movie_title) -1))) AS ExtractedTitle1
FROM movies.metadata;
```

### Reduce dimention of the table 

As was mentioned the table ratings has a lot of scoring for each film. To have only one score per movie,
the table is reduced in dimention to only have the mean of ratings per movie. 

```sql
SELECT movieID, ROUND(AVG(rating),2) AS Average_ratings
FROM movies.ratings
GROUP BY movieID;
```


### Joining the tables

Now we have know how to standarize the main field that would be the primary key to permit the joins with the tables. The results that are needed for each table is now the time to create a query that can merge the desired fields in one result.

The structure of this query is the next:

+ Rating1: Is the subquery that have the mean of the internet rating on the table ratings
+ Metadata: This subquery preprocess the table metadata with the previous steps defined on data cleaning and also is added the ImDB rating
+ Final table: generates the joining between the previous subqueries  with the help of the bridge table movie  

```sql
WITH Rating1 AS 
(
    SELECT movieID, ROUND(AVG(rating),2) AS Average_ratings
    FROM movies.ratings
    GROUP BY movieID
), 
Metadata AS 
( 
        WITH RankedMovies AS 
        (
            SELECT *, 
                ROW_NUMBER() OVER (PARTITION BY movie_title, duration, title_year, director_name ORDER BY movie_title) as Enumeration 
                FROM movies.metadata
                WHERE movie_title <> '' AND duration <> '' AND title_year <> ''
        )
        ,
        FINAL AS 
        (
            SELECT * FROM RankedMovies WHERE Enumeration = 1
        )

        SELECT  
        lower(trim(substr(movie_title, 1, CHAR_LENGTH(movie_title) -1))) AS ExtractedTitle1, 
        ROUND(AVG(imdb_score),2) AS Average_rating_per_movie
        FROM FINAL
        WHERE NOT (
            title_year REGEXP '^[^0-9]+$' 
            OR num_voted_users REGEXP '^[^0-9]+$' 
            OR budget REGEXP '^[^0-9]+$' 
            OR actor_3_facebook_likes REGEXP '^[^0-9]+$' 
            OR facenumber_in_poster REGEXP '^[^0-9]+$' 
            OR imdb_score REGEXP '^[^0-9]+$' 
            OR num_user_for_reviews REGEXP '^[^0-9]+$' 
            OR NOT (title_year BETWEEN 1900 AND 2023)
            OR NOT (imdb_score BETWEEN 1 AND 10)
            OR country REGEXP '[0-9]' 
            OR language REGEXP '[0-9]'
            OR country LIKE 'https%' 
            OR language LIKE 'https%' 
            OR content_rating NOT IN ('PG-13', 'PG', 'G', 'R', 'TV-14', 'TV-PG', 'TV-MA', 'TV-G', 'Not Rated', 'Unrated', 'TV-Y', 'TV-Y7')
                )
        GROUP BY 1
)

SELECT C.ExtractedTitle1, B.Average_ratings, C.Average_rating_per_movie

FROM Metadata AS C
LEFT JOIN movies.movies AS A
ON lower(trim( substr(A.title, 1, CHAR_LENGTH(A.title) -6))) = C.ExtractedTitle1
LEFT JOIN Rating1 AS B
ON A.movieId = B.movieId
WHERE A.title is not null;

```
![image_database11!](/images/SQL/img11.PNG " ")
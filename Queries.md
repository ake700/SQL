# The data used in this markdown can be found in the README file and in the repository

## The queries performed here are in MySQL 8.0 on Windows. The installation guide can be found [here](https://dev.mysql.com/doc/refman/8.0/en/windows-installation.html)

MySQL Workbench can be found in ```C:\Program Files\MySQL\MySQL Workbench 8.0\mysqlworkbench.exe```

### 0. General set-up in MySQL

Certain errors may be present in MySQL that prevents queries from running. One such example is a connection (with the server) that can be fixed by:
1. Going to the **Connection** tab in MySQL
2. Go to the **Advanced** sub-tab
3. In the **Others** section add the line 
```sql 
OPT_LOCAL_INFILE=1
```

### 1. To load CSV files from infile

Note: The following can cause an error
```sql
LOAD DATA INFILE "text.txt" INTO table mytable;
```
> The MySQL server is running with the --secure-file-priv option so it cannot execute this statement

To show the directory affected and for more information on the issue go [here](https://stackoverflow.com/questions/32737478/how-should-i-resolve-secure-file-priv-in-mysql)
```sql
SHOW VARIABLES LIKE "secure_file_priv";
``` 

Therefore, to load the local files based on  the *../MySQL Server 8.0/Uploads* folder we can run the following to create the database/tables and load the file(s)
```sql
CREATE database_name;
USE database_name;
CREATE TABLE table_name (
  column1 INTEGER NOT NULL AUTO_INCREMENT, # Unique ID for the record; column1 will be assigned a unique value if new entries added
  column2 INTEGER NOT NULL, # Integer can be replaced with VARCHAR() for text, replace the () with a value depending on length of the text
  # Add as many columns as needed
  PRIMARY KEY (column1) # This assigns a primary key for joining tables 
  );
```

Now that the table has been made, it's time to populate with the files

Don't forget to check that we can actually load local files
```sql
SET GLOBAL local_infile=1; # This allows local loading of filenames
```

For my CSVs \r worked best, but may also work with other line breaks as shown [here](https://stackoverflow.com/questions/44519478/in-file-csv-to-import-in-table-mysql-for-new-line-is-better-use-n-r-or-r)

**NOTE: If copying filepath directly, change all ```\``` into ```/```

```sql
USE database_name; 
LOAD DATA LOCAL INFILE "C:/FilePath/FilePath.csv" INTO TABLE table_name
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\r' 
IGNORE 1 ROWS;
```


Check that the file has been imported properly
```sql
SELECT *
FROM database_name.table_name LIMIT 10; 
```
I prefer writing database_name.table_name for when I have multiple tables in a database

### 2. Working with the Stats Canada table

### Queries are based on [W3Schools](https://www.w3schools.com/sql/)

Now that the general set-up for MySQL is complete and the database and table have been populated, the following will be based on my specific filepaths and database/table names

I originally was not happy with the table name, so to rename a table
```sql
RENAME TABLE table_name1 TO table_name2;
```

Some of the columns did not seem useful (e.g., not containing unique information or are sparsely populated), so I opted to remove them
```sql
ALTER TABLE cad_jobs
DROP COLUMN UOM
DROP COLUMN UOM_ID
DROP COLUMN scalar_factor
DROP COLUMN scalar_ID
DROP COLUMN status
DROP COLUMN symbol
DROP COLUMN terminated
DROP COLUMN decimals;
```

Let's learn a bit more about the table starting with looking at the date and geography of the data
```sql
SELECT DISTINCT ref_date, geo
FROM self.cad_jobs;
```
Looks like we have all the unique data between 1976-2021 for all the provinces. 

Check number of rows in the table
```sql
SELECT COUNT(ref_date)
FROM self.cad_jobs;
```

Check number of distinct entries in a column
```sql
SELECT COUNT(DISTINCT geo)
FROM self.cad_jobs;
```

How many entries are with a particular keyword in the column?
```sql
SELECT Count(geo)
FROM self.cad_jobs
WHERE geo = "Canada";
```
Check out other operators [here](https://www.w3schools.com/sql/sql_where.asp)

What are the unique provinces starting with N?
```sql
SELECT DISTINCT geo
FROM self.cad_jobs
WHERE geo LIKE 'N%';
```

How many entries are only in Alberta and British Columbia?
```sql
SELECT *
FROM self.cad_jobs
WHERE geo IN ("Alberta", "British Columbia");
```

Let's find all the available information in either 1996 or 2016 from any province starting with N
```sql
SELECT *
FROM self.cad_jobs
WHERE geo LIKE ("N%") and ref_date in ("1996", "2016");
```
**Note**: `%` is a wildcard. Other wildcards and their uses can be found [here](https://www.w3schools.com/sql/sql_wildcards.asp)

Order the column in descending order by ID for the geography that is not in Canada
```sql
SELECT *
FROM self.cad_jobs
WHERE geo = "Canada"
ORDER BY DGUID DESC;
```
```ORDER BY``` can be used for more than one column

Turns out we're missing a couple columns, let's see how to fix this. We want to add the **COORDINATES** and **VALUE** columns from our original CSV into our table

First let's make a temporary table then add specific columns from the CSV to that table
```sql
USE self;
CREATE TABLE a (
    DGUID INTEGER NOT NULL AUTO_INCREMENT,
    coordinates VARCHAR(150) ,
    amount INTEGER,
    PRIMARY KEY (DGUID)
    );
 
 # Now to only copy those columns from CSV into table a

LOAD DATA LOCAL INFILE "C:/ProgramData/MySQL/MySQL Server 8.0/Uploads/14100027.csv"
INTO TABLE a
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\r'
IGNORE 1 LINES
(@col1, @dummy, @dummy, @dummy, @dummy, @dummy, @dummy, @dummy, @dummy, @dummy, @dummy, @col12, @col13, @dummy, @dummy, @dummy, @dummy)
set DGUID = @col1, coordinate = @col12, amount = @col13;
```
**Note**: that the ```@dummy``` bypasses the other columns

Now let's add empty columns to populate in the original table. We also need to disable safeguard temporarily so we can combine via the primary key
```sql
SET SQL_SAFE_UPDATES = 0;
UPDATE self.cad_jobs 
INNER JOIN self.a ON cad_jobs.DGUID = a.DGUID
SET cad_jobs.coordinate = a.coordinates
SET cad_jobs.amount = a.amount;
SET SQL_SAFE_UPDATES = 1;
```

Double check the ```database.table1``` to make sure that the columns have been properly joined.
Looks like we have a couple duplicate columns actually, so let's drop those

```sql
ALTER TABLE self.cad_jobs
DROP COLUMN Coordinates,
DROP COLUMN Number;
```

Looking at the bottom of the table, which can be done by
```sql
SELECT *
FROM self.cad_jobs 
ORDER BY DGUID DESC LIMIT 10;
```

we can see that there's an extra row with no values at the bottom. Let's delete that.
```sql
DELETE FROM self.cad_jobs
WHERE DGUID = 223081;
```

### 3. More complex queries from the VPD Crimes data

Create the table first, load, then check
```sql
CREATE TABLE van_crime (
	ID INTEGER NOT NULL AUTO_INCREMENT, #Unique ID for the record; DGUID will be assigned a unique value if new entries added
    crime_type VARCHAR(150) NOT NULL,
    crime_year INTEGER NOT NULL,
    crime_month INTEGER NOT NULL,
    crime_day INTEGER NOT NULL, 
    crime_hour INTEGER NOT NULL,
    crime_minute INTEGER NOT NULL,
    crime_block VARCHAR(150),
    neighborhood VARCHAR(150) NOT NULL,
    coord_x integer, # Coordinate values projected in UTM Zone 10
	  coord_y integer, # Coordinate values projected in UTM Zone 10
    PRIMARY KEY (ID)
    );

LOAD DATA LOCAL INFILE "C:/ProgramData/MySQL/MySQL Server 8.0/Uploads/Crimes2015_2022.csv" INTO TABLE van_crime
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\r\n' # prevents the extra row from being added at the bottom
IGNORE 2 ROWS;

# To clean up the table - if needed
DELETE FROM self.van_crime 
WHERE ID = 303219	
```

#### Example questions from this data:
1. Does any of the data have duplicates? If not, create one then delete

Create a duplicate if needed
```sql
INSERT INTO self.van_crime
	(crime_type, crime_year, crime_month, crime_day, crime_hour, crime_minute, crime_block, neighborhood, coord_x, coord_y)
SELECT 
	crime_type, crime_year, crime_month, crime_day, crime_hour, crime_minute, crime_block, neighborhood, coord_x, coord_y
FROM self.van_crime
WHERE ; # Input the specific conditions (column values) and not the primary key
```
Check for duplicates
```sql
SELECT crime_type, crime_year, crime_month, crime_day, crime_hour, crime_minute, crime_block, neighborhood, coord_x, coord_y, COUNT(*)
FROM self.van_crime
GROUP BY crime_type, crime_year, crime_month, crime_day, crime_hour, crime_minute, crime_block, neighborhood, coord_x, coord_y
HAVING COUNT(*) > 1;
```
Remove duplicates
```sql
# create a new tmp_van_crime table
CREATE TABLE tmp_van_crime LIKE self.van_crime;
# add a unique constraint
ALTER TABLE tmp_van_crime ADD UNIQUE (crime_type, crime_year, crime_month, crime_day, crime_hour, crime_minute, crime_block, neighborhood, coord_x, coord_y);
# scan over the van_crime table to insert van_crime entries
INSERT IGNORE INTO tmp_van_crime SELECT * FROM van_crime ORDER BY ID;
# rename tables
RENAME TABLE van_crime TO backup_van_crime, tmp_van_crime TO van_crime;
```

2. What is the first and last 5 entries in the table?
Prints the top and bottom 5 entries for all rows
```sql
(SELECT *
FROM self.van_crime
ORDER BY ID ASC LIMIT 5)
UNION ALL
(SELECT *
FROM self.van_crime
ORDER BY ID DESC LIMIT 5)
```

3. What year/neighborhood has the most crimes?
```sql
SELECT neighborhood, COUNT(crime_type) as count
FROM van_crime
GROUP BY neighborhood
ORDER BY count DESC;
```

4. When is the safest and least safe time to be out and about?
Prints the top and bottom 5 hours and the associated count of crime type
```sql
(SELECT crime_hour, COUNT(crime_type) as count
FROM van_crime
GROUP BY crime_hour
ORDER BY count ASC LIMIT 5)
UNION ALL
(SELECT crime_hour, COUNT(crime_type) as count
FROM van_crime
GROUP BY crime_hour
ORDER BY count DESC LIMIT 5)
```

5. What is the most amount of (type of crime) in (some neighborhood)? 
Most frequent type of crime in CBD and Oakridge
```sql
SELECT crime_type, COUNT(crime_type) as count
FROM van_crime
WHERE neighborhood in ("Central Business District", "Oakridge")
GROUP BY crime_type
ORDER BY count DESC;
```

6. Fetch the block(s) given the type of crime
```sql
SELECT crime_block, COUNT(crime_type) as count
FROM van_crime
GROUP BY crime_block
ORDER BY count DESC;
```

7. How many crimes are committed after (hour)? 
```sql
SELECT crime_hour, COUNT(crime_type) as count
FROM van_crime
GROUP BY crime_hour
ORDER BY count DESC;
```

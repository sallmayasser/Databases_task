# Step 1: Create Two MySQL Tables

Table 1: All columns as VARCHAR(255) (no indexes)

```sql
CREATE TABLE people_raw (
    user_id VARCHAR(255),
    username VARCHAR(255),
    sex VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(255),
    dob VARCHAR(255),
    job_title VARCHAR(255)
);

```

Table 2: Correct data types + indexes

```sql
CREATE TABLE people (
    user_id CHAR(15) PRIMARY KEY,
    username VARCHAR(100),
    sex ENUM('Male','Female'),
    email VARCHAR(100) ,
    phone VARCHAR(15) unique,
    dob DATE,
    job_title VARCHAR(255),
    INDEX idx_sex(sex),
    INDEX idx_job_title(job_title)
);

```

Notes:

dob as DATE instead of VARCHAR.
ENUM for sex to save space.
PRIMARY KEY and UNIQUE for faster lookups.
Added indexes on commonly queried columns.

# Step 2: Load Data

If your CSV is /var/lib/mysql-files/people-1M.csv:

For Table 1:

```sql
LOAD DATA INFILE '/var/lib/mysql-files/people-1M.csv'
INTO TABLE people_raw
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
IGNORE 1 ROWS
(user_id, username, sex, email, phone, @dob, job_title)
SET dob = @dob; -- keeping as string
```

![people raw](images/load-raw-data.png)

For Table 2 (convert DOB to proper date format):

```sql
LOAD DATA INFILE '/var/lib/mysql-files/people-1M.csv'
INTO TABLE people
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
IGNORE 1 ROWS
(user_id, username, sex, email, phone, @dob, job_title)
SET dob = STR_TO_DATE(@dob, '%m/%d/%Y');
```

![people](images/load_data.png)

⚠️ Make sure all dates in CSV are in MM/DD/YYYY format for STR_TO_DATE to work.

![verfy convert ](images/verify-convert.png)

# Step 3: Performance Comparison

You can run the same queries on both tables to measure performance:

Aggregation examples:

1. COUNT total users

```sql
-- people_raw
EXPLAIN ANALYZE
SELECT COUNT(*) FROM people_raw;

-- people
EXPLAIN ANALYZE
SELECT COUNT(*) FROM people;
```

MySQL will output execution time in seconds for each table.

![Num of people](images/num_of_people.png)

2. COUNT by sex

```sql
-- people_raw
EXPLAIN ANALYZE
SELECT sex, COUNT(*) FROM people_raw GROUP BY sex;

-- people
EXPLAIN ANALYZE
SELECT sex, COUNT(*) FROM people GROUP BY sex;
```

![Group by](images/group_by.png)

3. Filter with WHERE

```sql
-- people_raw
EXPLAIN ANALYZE
SELECT * FROM people_raw WHERE sex='Female';

-- people
EXPLAIN ANALYZE
SELECT * FROM people WHERE sex='Female';
```

EXPLAIN ANALYZE shows whether MySQL used full table scan or index.

![Where ](images/where.png)

4. GROUP BY / AVG age

```sql
-- people_raw
EXPLAIN ANALYZE
SELECT sex, AVG(YEAR(CURDATE()) - YEAR(STR_TO_DATE(dob, '%m/%d/%Y'))) AS avg_age
FROM people_raw
GROUP BY sex;

-- people
EXPLAIN ANALYZE
SELECT sex, AVG(YEAR(CURDATE()) - YEAR(dob)) AS avg_age
FROM people
GROUP BY sex;
```

![alt text](images/groupby_avg.png)

5. Sorting dob DESC with LIMIT

```sql
-- people_raw
EXPLAIN ANALYZE
SELECT * FROM people_raw ORDER BY dob DESC LIMIT 10;

-- people
EXPLAIN ANALYZE
SELECT * FROM people ORDER BY dob DESC LIMIT 10;
```

- without index

  raw data is sigltly faster because :
  people.dob is a DATE → slightly more CPU per row than VARCHAR (tiny difference).
  people_raw.dob is VARCHAR → sorting strings is slightly faster here because MySQL can do byte-wise comparisons instead of date computations.

  ![withoot index](images/sort_without_idx.png)

- with index
  ![with index](images/sort_idx.png)

# Step 4: Backup & Automation

Backup Table 2:

```bash
mysqldump -u root -p mydatabase people > /backups/people_backup.sql
```

Automate weekly backup (Linux cron):

Create a script

```bash

sudo vim /usr/local/bin/backup_people.sh:
```

```bash
#!/bin/bash
DATE=$(date +%F)
mysqldump -u root -pYourPassword mydatabase people > /backups/people_$DATE.sql
```

```bash
# Make it executable:
chmod +x /usr/local/bin/backup_people.sh
# Add cron job:
crontab -e
# Add:
0 2 \* \* 0 /usr/local/bin/backup_people.sh
```

This runs every Sunday at 2 AM.

# Step 5: Restore Test

```sql
CREATE TABLE people_restore LIKE people;
```

-- Restore backup

```bash
mysql -u root -p mydatabase < /backups/people_backup.sql
```

Check data:

```sql
SELECT COUNT(*) FROM people_restore;
SELECT * FROM people_restore LIMIT 10;
```

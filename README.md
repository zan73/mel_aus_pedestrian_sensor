# mel_aus_pedestrian_sensor

## Data analysis and preparation
### sensor dataset
Initial import into an RDBMS was to a staging table with all varchar fields to allow analysis of data:
```
create table sensor_stg(
    sensor_id VARCHAR(400)
    ,sensor_description VARCHAR(400)
    ,sensor_name VARCHAR(400)
    ,installation_date VARCHAR(400)
    ,status VARCHAR(400)
    ,note VARCHAR(400)
    ,direction_1 VARCHAR(400)
    ,direction_2 VARCHAR(400)
    ,latitude VARCHAR(400)
    ,longitude VARCHAR(400)
    ,location VARCHAR(400)
)

LOAD DATA LOW_PRIORITY LOCAL INFILE 'Pedestrian_Counting_System_-_Sensor_Locations.csv' INTO TABLE `mel_aus_foot_traffic`.`sensor_raw` CHARACTER SET utf8 FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"' LINES TERMINATED BY '\n' IGNORE 1 LINES (`sensor_id`, `sensor_description`, `sensor_name`, `installation_date`, `status`, `note`, `direction_1`, `direction_2`, `latitude`, `longitude`, `location`);

```

- sensor_id appears to be the record identifier, location attribute unnecessary as it's a repeat of latitude and longitude fields.
- status field values would need to be confirmed with data steward and a lookup table implemented
- if sensor_ids are all numeric, would potentially be more efficient for query performance and storage to store as integer
- installation_date, latitude and longitude would be stored as date and decimal(12,9) provided they pass datatype conversions, these datatypes would be more efficient for querying and usage for date/GIS functions.
- Regular ETL would import the data and any records failing datatype conversion would be written to an error log

#### Data type/primary key testing:
sensor_id uniqueness:
```
--Returns 0 records - field is unique

SELECT sensor_id
FROM sensor_stg
GROUP BY sensor_id
HAVING COUNT(*) > 1
```

data type validations:
```
--No errors, proposed datatypes are acceptable

SELECT CAST(sensor_id AS INTEGER), CAST(installation_date AS DATE), CAST(latitude AS DECIMAL(12,9)), CAST(longitude AS DECIMAL(12,9))
FROM sensor_stg
```

#### Load to final table
varchar(400) fields could be reduced in size for storage efficiency, but for this exercise they have been left as is
```
CREATE TABLE sensor AS
SELECT
	CAST(sensor_id AS INTEGER) AS sensor_id
	,sensor_description
	,sensor_name
	,CAST(installation_date AS DATE) AS installation_date
	,status
	,note
	,direction_1
	,direction_2
	,CAST(latitude AS DECIMAL(12,9)) AS latitude
	,CAST(longitude AS DECIMAL(12,9)) AS longitude
FROM sensor_stg
```
---
### sensor counts dataset
Initial import into an RDBMS was to a staging table with all varchar fields to allow analysis of data:
```
create table sensor_counts_stg(
    ID VARCHAR(400)
    ,Date_Time VARCHAR(400)
    ,YEAR VARCHAR(400)
    ,MONTH VARCHAR(400)
    ,Mdate VARCHAR(400)
    ,DAY VARCHAR(400)
    ,TIME VARCHAR(400)
    ,Sensor_ID VARCHAR(400)
    ,Sensor_Name VARCHAR(400)
    ,Hourly_Counts VARCHAR(400)
)

LOAD DATA LOW_PRIORITY LOCAL INFILE 'Pedestrian_Counting_System_-_Monthly__counts_per_hour_.csv' INTO TABLE `mel_aus_foot_traffic`.`sensor_counts_stg` CHARACTER SET utf8 FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"' LINES TERMINATED BY '\n' IGNORE 1 LINES (`ID`, `Date_Time`, `YEAR`, `MONTH`, `Mdate`, `DAY`, `TIME`, `Sensor_ID`, `Sensor_Name`, `Hourly_Counts`);
```

- ID appears to be the record identifier - check for duplicates. Regular ETL would check for duplicates and potentially write to an error file to be checked whether record is an update to an existing record, etc. bigint rather than int is best, as int may not be large enough if the number of records is huge
- YEAR, MONTH, Mdate, DAY, TIME appear unnecessary - Date_Time field cast as timestamp will more efficiently store all required information
- Sensor_ID is our foreign key to sensor table. Regular ETL would perform checks to ensure that all Sensor_IDs in this dataset are matched to sensor_ids in sensor table. 
- Sensor_Name is unnecessary as it can be derived from sensor table. Check whether some sensor_count records do not align with sensor table. If the name changes over time, we could do some further analysis and capture name changes in a history table or make the sensor table a slowly-changing dimension table. 
- if sensor_ids are all numeric, would potentially be more efficient for query performance and storage to store as integer
- installation_date, latitude and longitude would be stored as date and decimal(12,9) provided they pass datatype conversions, these datatypes would be more efficient for querying and usage for date/GIS functions.
- Ensure Hourly_Counts column is numeric

#### Data type/primary key testing:
ID uniqueness:
```
--Returns 0 records - field is unique

SELECT ID
FROM sensor_counts_stg
GROUP BY ID
HAVING COUNT(*) > 1
```
Date_Time datatype conversion check:
```
--No errors returned, datatype convesion OK

SELECT STR_TO_DATE(Date_Time , '%M %d, %Y %h:%i:%s %p') AS date_time
FROM sensor_counts_stg
GROUP BY STR_TO_DATE(Date_Time , '%M %d, %Y %h:%i:%s %p')
```
Sensor_Name checks:
```
--Name has changed over time - possibly create a history table or SCD type 2 or analyse to confirm that id and . For now, ignore sensor_name

SELECT sensor_id FROM (
SELECT DISTINCT sensor_id, sensor_name
FROM sensor_counts_stg
) A
GROUP BY sensor_id
HAVING COUNT(*)>1
```


```mermaid
erDiagram
    sensor ||..o{ sensor_count : collects
      sensor {
        bigint sensor_id PK
        string sensor_description
        string sensor_name
	date installation_date
	string status
	string note
	string direction_1
	string direction_2
	decimal latitude
	decimal longitude
    }
    sensor_count {
        int ID PK
        timestamp date_time
        int sensor_id FK
	int hourly_counts
    }
    
```

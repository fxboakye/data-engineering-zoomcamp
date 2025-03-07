## Week 3 Homework
<b><u>Important Note:</b></u> <p>You can load the data however you would like, but keep the files in .GZ Format. 
If you are using orchestration such as Airflow or Prefect do not load the data into Big Query using the orchestrator.</br> 
Stop with loading the files into a bucket. </br></br>
<u>NOTE:</u> You can use the CSV option for the GZ files when creating an External Table</br>

<b>SETUP:</b></br>
Create an external table using the fhv 2019 data. </br>
Create a table in BQ using the fhv 2019 data (do not partition or cluster this table). </br>
Data can be found here: https://github.com/DataTalksClub/nyc-tlc-data/releases/tag/fhv </p>

## Question 1:
What is the count for fhv vehicle records for year 2019?
- 65,623,481
* [x] 43,244,696
- 22,978,333
- 13,942,414

```sql
-- Creating External table
CREATE EXTERNAL TABLE IF NOT EXISTS flv.fhvdata
OPTIONS (
  FORMAT= 'parquet',
  URIS=['gs://data-en/data/fhv/*.parquet']);
  
-- Querying rows count
SELECT COUNT(pickup_datetime) FROM flv.fhvdata ;

```
 

## Question 2:
Write a query to count the distinct number of affiliated_base_number for the entire dataset on both the tables.</br> 
What is the estimated amount of data that will be read when this query is executed on the External Table and the Table?

- 25.2 MB for the External Table and 100.87MB for the BQ Table
- 225.82 MB for the External Table and 47.60MB for the BQ Table
- 0 MB for the External Table and 0MB for the BQ Table
* [x] 0 MB for the External Table and 317.94MB for the BQ Table 

```sql
-- External Table Query
SELECT DISTINCT COUNT(affiliated_base_number) FROM flv.fhvdata;

--Native BigQuery Table Query
SELECT DISTINCT COUNT(affiliated_base_number) FROM flv.fhv_native;
```


## Question 3:
How many records have both a blank (null) PUlocationID and DOlocationID in the entire dataset?
* [X] 717,748
- 1,215,687
- 5
- 20,332

```sql
SELECT
  COUNT(pickup_datetime)
FROM flv.fhvdata
WHERE pulocationid IS NULL
AND dolocationid IS NULL;
```

## Question 4:
What is the best strategy to optimize the table if query always filter by pickup_datetime and order by affiliated_base_number?
- Cluster on pickup_datetime Cluster on affiliated_base_number
* [X] Partition by pickup_datetime Cluster on affiliated_base_number
- Partition by pickup_datetime Partition by affiliated_base_number
- Partition by affiliated_base_number Cluster on pickup_datetime

## Question 5:
Implement the optimized solution you chose for question 4. Write a query to retrieve the distinct affiliated_base_number between pickup_datetime 2019/03/01 and 2019/03/31 (inclusive).</br> 
Use the BQ table you created earlier in your from clause and note the estimated bytes. Now change the table in the from clause to the partitioned table you created for question 4 and note the estimated bytes processed. What are these values? Choose the answer which most closely matches.
- 12.82 MB for non-partitioned table and 647.87 MB for the partitioned table
* [X] 647.87 MB for non-partitioned table and 23.06 MB for the partitioned table
- 582.63 MB for non-partitioned table and 0 MB for the partitioned table
- 646.25 MB for non-partitioned table and 646.25 MB for the partitioned table

```sql
--Creating Partitioned and Clustered Table
CREATE OR REPLACE TABLE `flv.fhvpartitioned`
PARTITION BY
 DATE(pickup_datetime)
CLUSTER BY
 affiliated_base_number AS
SELECT
 *
FROM
 `flv.fhv_native`
WHERE
pickup_datetime BETWEEN '2019-03-01' AND '2019-03-31';

--Querying Partitioned table
SELECT DISTINCT COUNT(affiliated_base_number) FROM flv.fhvpartitioned;

--Querying un-partitioned table
SELECT DISTINCT COUNT(affiliated_base_number) FROM flv.fhvnative
WHERE
pickup_datetime BETWEEN '2019-03-01' AND '2019-03-31';

```



## Question 6: 
Where is the data stored in the External Table you created?

- Big Query
* [x] GCP Bucket
- Container Registry
- Big Table


## Question 7:
It is best practice in Big Query to always cluster your data:
* [x] True
- False


## (Not required) Question 8:
A better format to store these files may be parquet. Create a data pipeline to download the gzip files and convert them into parquet. Upload the files to your GCP Bucket and create an External and BQ Table. 

```python
import pandas as pd
from pathlib import Path
from prefect_gcp.cloud_storage import GcsBucket
from prefect import flow, task
from prefect.tasks import task_input_hash
from datetime import timedelta


@task(log_prints=True, tags=["extract"], cache_key_fn=task_input_hash, cache_expiration=timedelta(days=1))
def extract(url) -> pd.DataFrame:
    """Read data from web to Dataframe"""
    data=pd.read_csv(url)

    #changed all column headers to lowercase
    data.columns=[i.lower() for i in data.columns]

    data['pickup_datetime']=pd.to_datetime(data['pickup_datetime'])
    data['dropoff_datetime']=pd.to_datetime(data['dropoff_datetime'])
    data['pulocationid']=pd.to_numeric(data['pulocationid'], downcast='float')
    data['dolocationid']=pd.to_numeric(data['dolocationid'], downcast='float')

    return data


@task()
def write_local(data: pd.DataFrame,  dataset_file: str) -> Path:
    """Write dataframe to local"""
    new_path=f"./data/fhv/{dataset_file}.parquet"
    data.to_parquet(Path(new_path), compression="gzip")
    return new_path


@task()
def write_gcs(path) -> None:
    """"Uploading local file to GCS"""
    gcp_block = GcsBucket.load("de-zoom")
    gcp_block.upload_from_path(
        from_path=Path(path), to_path=path)
    return


@flow()
def etl_to_gcs(year:int, month: int) -> None:
    data_set=f"fhv_tripdata_{year}-{month:02}"
    url=f"https://github.com/DataTalksClub/nyc-tlc-data/releases/download/fhv/{data_set}.csv.gz"
    data= extract(url)
    new_path=write_local(data, data_set)
    write_gcs(new_path)

@flow()
def main_flow(year: int=2019, months:list =[1,2]):
    for month in months:
        etl_to_gcs(year,month)
    
if __name__=='__main__':
    year= 2019
    months= list(range(1,13))
    main_flow(year, months)
    
  ```


Note: Column types for all files used in an External Table must have the same datatype. While an External Table may be created and shown in the side panel in Big Query, this will need to be validated by running a count query on the External Table to check if any errors occur. 
 
## Submitting the solutions

* Form for submitting: https://forms.gle/rLdvQW2igsAT73HTA
* You can submit your homework multiple times. In this case, only the last submission will be used. 

Deadline: 13 February (Monday), 22:00 CET


## Solution

We will publish the solution here

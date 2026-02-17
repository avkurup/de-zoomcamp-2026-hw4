# Module 4 Homework: Analytics Engineering with dbt

**Data Engineering Zoomcamp 2026 Cohort**

In this homework, I used the dbt project in `04-analytics-engineering/taxi_rides_ny/` to transform NYC taxi data and answer questions by querying the models.

## Setup

- **Platform:** dbt Cloud + Google BigQuery
- **GCP Project:** `de-zoomcamp-2026-487003`
- **BigQuery Dataset:** `nytaxi` (source data), `nytaxi_dev` (dbt dev models)
- **GCS Bucket:** `de-zoomcamp-2026-yellow-taxi-ashok`
- **Data Source:** [DataTalksClub NYC TLC Data](https://github.com/DataTalksClub/nyc-tlc-data)

### Data Loaded

| Dataset | Records | Period |
______________________________
| Green Taxi | 7,778,101 | 2019-2020 |
| Yellow Taxi | 109,047,518 | 2019-2020 |
| FHV | 43,244,696 | 2019 |
______________________________

### External Tables Created

```sql
-- Green taxi data
CREATE OR REPLACE EXTERNAL TABLE `nytaxi.green_tripdata`
OPTIONS (
  format = 'CSV',
  uris = ['gs://de-zoomcamp-2026-yellow-taxi-ashok/taxi_data/green/green_tripdata_*.csv.gz']
);

-- Yellow taxi data
CREATE OR REPLACE EXTERNAL TABLE `nytaxi.yellow_tripdata`
OPTIONS (
  format = 'CSV',
  uris = ['gs://de-zoomcamp-2026-yellow-taxi-ashok/taxi_data/yellow/yellow_tripdata_*.csv.gz']
);

-- FHV data
CREATE OR REPLACE EXTERNAL TABLE `nytaxi.fhv_tripdata`
OPTIONS (
  format = 'CSV',
  uris = ['gs://de-zoomcamp-2026-yellow-taxi-ashok/taxi_data/fhv/fhv_tripdata_*.csv.gz']
);
```

---

## Questions & Answers

### Question 1. dbt Lineage and Execution
Given a dbt project with the following structure:

models/
├── staging/
│   ├── stg_green_tripdata.sql
│   └── stg_yellow_tripdata.sql
└── intermediate/
    └── int_trips_unioned.sql (depends on stg_green_tripdata & stg_yellow_tripdata)
    
If you run dbt run --select int_trips_unioned, what models will be built?

stg_green_tripdata, stg_yellow_tripdata, and int_trips_unioned (upstream dependencies)
Any model with upstream and downstream dependencies to int_trips_unioned
int_trips_unioned only
int_trips_unioned, int_trips, and fct_trips (downstream dependencies)

Answer: `int_trips_unioned` only**

The `--select` flag without the `+` prefix builds only the specified model. To include upstream dependencies, would need `dbt run --select +int_trips_unioned`.

---

### Question 2. dbt Tests
You've configured a generic test like this in your schema.yml:
```yaml
columns:
  - name: payment_type
    data_tests:
      - accepted_values:
          arguments:
            values: [1, 2, 3, 4, 5]
            quote: false
```
Your model fct_trips has been running successfully for months. A new value 6 now appears in the source data.

What happens when you run dbt test --select fct_trips?

Answer: dbt fails the test with non-zero exit code**

The `accepted_values` test checks that all values are within the specified list. Value `6` is not in `[1, 2, 3, 4, 5]`, so the test fails. By default, dbt tests have severity `error`, which returns a non-zero exit code.

---

### Question 3: Count of Records in `fct_monthly_zone_revenue`

```sql
SELECT COUNT(*) FROM `nytaxi_dev.fct_monthly_zone_revenue`;
```

**Answer: 12,184**

---

### Question 4: Zone with Highest Revenue for Green Taxis in 2020

```sql
SELECT pickup_zone, SUM(revenue_monthly_total_amount) as total_revenue
FROM `nytaxi_dev.fct_monthly_zone_revenue`
WHERE service_type = 'Green'
  AND revenue_month >= '2020-01-01'
  AND revenue_month < '2021-01-01'
GROUP BY pickup_zone
ORDER BY total_revenue DESC
LIMIT 5;
```

**Answer: East Harlem North**

---

### Question 5: Total Trips for Green Taxis in October 2019

```sql
SELECT SUM(total_monthly_trips) as trip_count
FROM `nytaxi_dev.fct_monthly_zone_revenue`
WHERE service_type = 'Green'
  AND revenue_month = '2019-10-01';
```

**Answer: 384,624**

---

### Question 6: Count of Records in `stg_fhv_tripdata`

Created staging model `models/staging/stg_fhv_tripdata.sql`:

```sql
with source as (
    select * from {{ source('raw', 'fhv_tripdata') }}
),

renamed as (
    select
        cast(dispatching_base_num as string) as dispatching_base_num,
        cast(PUlocationID as integer) as pickup_location_id,
        cast(DOlocationID as integer) as dropoff_location_id,
        cast(pickup_datetime as timestamp) as pickup_datetime,
        cast(dropOff_datetime as timestamp) as dropoff_datetime,
        cast(SR_Flag as string) as sr_flag,
        cast(Affiliated_base_number as string) as affiliated_base_number
    from source
    where dispatching_base_num is not null
)

select * from renamed
```

Verification query:

```sql
SELECT COUNT(*) FROM `nytaxi_dev.stg_fhv_tripdata`;
```

**Answer: 43,244,693**

---


- **Homework URL:** https://courses.datatalks.club/de-zoomcamp-2026/homework/hw4
- **GitHub Repo:** https://github.com/avkurup/de-zoomcamp-2026-hw4

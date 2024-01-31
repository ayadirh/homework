## Week 1 Homework


## Question 1. Knowing docker tags

Do the same for "docker run".

Which tag has the following text? - *Automatically remove the container when it exits* 

```docker run --help | grep "Automatically remove"```


## Question 2. Understanding docker first run 

Run docker with the python:3.9 image in an interactive mode and the entrypoint of bash.
Now check the python modules that are installed ( use ```pip list``` ). 

```docker run -it --entrypoint /bin/bash python:3.9```
```pip list```


### Running Postgres with Docker

#### Windows

Running Postgres on Windows (note the full path)

```yaml
services:
  pgdatabasehw:
    image: postgres:13
    restart: always
    environment:
      POSTGRES_USER: root
      POSTGRES_PASSWORD: root
      POSTGRES_DB: ny_green_taxi
    volumes:
      - "./green-taxi-volume:/var/lib/postgresql/data:rw"
    ports:
      - "5432:5432"
  pgadmin:
    image: dpage/pgadmin4
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: root
    ports:
      - "8080:80"
```

```docker-compose up```


#### Insert Data Using Python

```URL="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/green/green_tripdata_2019-09.csv.gz"```

```python
python3 ingest_data.py \
  --user=root \
  --password=root \
  --host=localhost \
  --port=5432 \
  --db=ny_green_taxi \
  --table_name=green_taxi_trips \
  --url=${URL}
```

#### Ingest_data.py 
Same as ingest_data.py but with name changes for date and timestamp columns

```python
    df.lpep_pickup_datetime = pd.to_datetime(df.lpep_pickup_datetime)
    df.lpep_dropoff_datetime = pd.to_datetime(df.lpep_dropoff_datetime)
```

Using `pgcli` to connect to Postgres

```bash
pgcli -h localhost -p 5432 -u root -d ny_green_taxi
```

## Question 3. Count records 

How many taxi trips were totally made on September 18th 2019?

Tip: started and finished on 2019-09-18. 

Remember that `lpep_pickup_datetime` and `lpep_dropoff_datetime` columns are in the format timestamp (date and hour+min+sec) and not in date.

```sql
 select count(1) from green_taxi_trips where date(lpep_pickup_datetime) = '2019-09-18' AND date(lpep_drop off_datetime) = '2019-09-18';
```

## Question 4. Longest trip for each day

Which was the pick up day with the longest trip distance?
Use the pick up time for your calculations.

Tip: For every trip on a single day, we only care about the trip with the longest distance. 

```sql
select DATE_TRUNC('day', lpep_pickup_datetime) AS pickup_date, MAX(trip_distance) AS largest_trip_distance FRO M green_taxi_trips GROUP BY DATE_TRUNC('day', lpep_pickup_datetime) ORDER BY largest_trip_distance desc limit 1;
```

### Creating Table

```sql
CREATE TABLE taxi_locations (LocationID INT PRIMARY KEY,Borough VARCHAR(255), Zone VARCHAR(255), service_zone VARCHAR(255));
```

### Insertion
```sql
 INSERT INTO taxi_locations (LocationID, Borough, Zone, service_zone) VALUES (1, 'EWR', 'Newark Airport', 'EWR');
```


## Question 5. Three biggest pick up Boroughs

Consider lpep_pickup_datetime in '2019-09-18' and ignoring Borough has Unknown

Which were the 3 pick up Boroughs that had a sum of total_amount superior to 50000?

```sql
SELECT borough, SUM(total_amount) AS total_amount_sum FROM green_taxi_trips , taxi_locations WHERE lpep_pickup_datetime >= '2019-09-18'::date AND lpep_pickup_datetime < '2019-09-19'::date AND borough != 'Unknown' GROU P BY borough HAVING SUM(total_amount) > 50000 ORDER BY total_amount_sum DESC LIMIT 3;
```


## Question 6. Largest tip

For the passengers picked up in September 2019 in the zone name Astoria which was the drop off zone that had the largest tip?
We want the name of the zone, not the id.

Note: it's not a typo, it's `tip` , not `trip`

```sql
 SELECT t2.zone = 'Astoria' as pickupzone, tl.zone AS dropzone, MAX(gtt.tip_amount) FROM green_taxi_trips gtt JOIN taxi_locations tl ON gtt."DOLocationID" = tl.locationid JOIN taxi_locations t2 ON gtt."PULocationID" =
 t2.locationid WHERE lpep_pickup_datetime >= '2019-09-01'::date AND lpep_pickup_datetime < '2019-10-01'::date AND t2.zone = 'Astoria' GROUP BY tl.zone, gtt."DOLocationID", t2.zone, gtt.tip_amount ORDER BY gtt.tip_amount DESC LIMIT 5;
```



## Terraform
### main.tf file
```terraform
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "5.6.0"
    }
  }
}

provider "google" {
  #credentials = file(var.credentials)
  project     = var.project
  region      = var.region
}


resource "google_storage_bucket" "demo-bucket" {
  name          = var.gcs_bucket_name
  location      = var.location
  force_destroy = true


  lifecycle_rule {
    condition {
      age = 1
    }
    action {
      type = "AbortIncompleteMultipartUpload"
    }
  }
}



resource "google_bigquery_dataset" "demo_dataset" {
  dataset_id = var.bq_dataset_name
  location   = var.location
}
```

### variables.tf
```terraform
variable "credentials" {
  description = "My Credentials"
  default     = "<Path to your Service Account json file>"
  #ex: if you have a directory where this file is called keys with your service account json file
  #saved there as my-creds.json you could use default = "./keys/my-creds.json"
}


variable "project" {
  description = "Project"
  default     = "dtc-de-course-2024-411721"
}

variable "region" {
  description = "Region"
  #Update the below to your desired region
  default     = "us-central1"
}

variable "location" {
  description = "Project Location"
  #Update the below to your desired location
  default     = "US"
}

variable "bq_dataset_name" {
  description = "My BigQuery Dataset Name"
  #Update the below to what you want your dataset to be called
  default     = "zoomcamp_demo_1_dataset"
}

variable "gcs_bucket_name" {
  description = "My Storage Bucket Name"
  #Update the below to a unique bucket name
  default     = "zoomcamp-demo-bucketname"
}

variable "gcs_storage_class" {
  description = "Bucket Storage Class"
  default     = "STANDARD"
}
```
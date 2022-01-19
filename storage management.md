# Storage
### Best Practices

## AWS RedShift
Amazon Redshift uses SQL to analyze structured and semi-structured data across data warehouses, operational databases, and data lakes, using AWS-designed hardware and machine learning to deliver the best price performance at any scale. 

When loading data into AWS Redshift, using COPY command is recommended for large data sets.

The COPY command appends the new input data to any existing rows in the table.

The maximum size of a single input row from any source is 4 MB.


---

### 0. Pre-step

Install `AWS` cli V2 following official instructions [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
```python
# If unable to connect to AWS through CLI. You may access AWS website and use AWS CLI online (console.aws.amazon.com)
aws s3 ls

# to create a bucket
aws s3api create-bucket --bucket BUCKET_NAME --region us-east-2 --create-bucket-configuration  LocationConstraint=us-east-2

# to upload into a bucket use
# aws s3 cp origin destination
aws s3 cp segments.csv s3://BUCKET_NAME
```

OR

Install Cloud SDK fro `GCP` which contains `gcloud`, `gsutil` and `bq` command-line tools following official instructions [here](https://cloud.google.com/sdk/docs/install)

#### IMPORTANT make sure that the account you are setting up has owner permissions in the project
#### If Unsure how if your account hass admin access, refer to AWS IAM for users and polcies management (https://console.aws.amazon.com/iamv2/home#/home)
---

#### For the following steps, please notice that you need to replace BUCKET_NAME with an unique name

### 1. Upload a file to your bucket (S3/GCS)

1. Download [this](https://drive.google.com/file/d/1tSnINCBDWziorV9PO67M6eYbl7maOlXs/view?usp=sharing) file, which is only ~70MB
2. Upload it to your bucket
- For AWS use: </br>

```python
# validate that you can see the buckets of your project
aws s3 ls

# to create a bucket
aws s3api create-bucket --bucket BUCKET_NAME --region us-east-2 --create-bucket-configuration  LocationConstraint=us-east-2

# to upload into a bucket use
# aws s3 cp origin destination
aws s3 cp segments.csv s3://BUCKET_NAME
```

- For GCS use: </br>

```python
# maybe you need to reauthenticate
gcloud auth login --update-adc

# validate project info with
gcloud config list

# if is not the project you want use
gcloud config set project {PROJECT_ID}

# to create a bucket
gsutil mb gs://BUCKET_NAME

# to upload into a bucket use
# gsutil cp origin destination
gsutil cp segments.csv gs://BUCKET_NAME
```

---
### 2.  Copy a file from a bucket to your bucket (S3/GCS)

In this section you are going to copy a ~20GB

- For AWS use: </br>

```python
# to copy from a bucket use
# aws s3  cp origin destination
aws s3 cp s3://wl-de-training-data/challenge/profeco/all_data.csv s3://BUCKET_NAME
```

- For GCP use: </br>

```python
# to copy from a bucket use
# gsutil cp origin destination
gsutil cp gs://de-bootcamp-2021/all_data.csv gs://BUCKET_NAME
```
---

### 3.  Prepare your data warehouse for ingestion

- For AWS create you redshift cluster

```python
aws redshift create-cluster --node-type dc2.large --number-of-nodes 2 --master-username adminuser --master-user-password TopSecret1 --cluster-identifier mycluster --region us-east-2 --db-name my_database
```

More information about instance types [here](https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-clusters.html)
- For GCP create a `dataset`
```python
bq --location=US mk -d \
--description "This is my test dataset." \
mydataset
```
---
### 4. Copy data from a bucket to a data warehouse

- For AWS:
  ```python
    # create a table
    aws redshift-data execute-statement \
    --database dev \
    --db-user adminuser \
    --cluster-identifier debootcampfmncluster \
    --sql "CREATE TABLE profeco (producto varchar (255),presentacion varchar (255), marca varchar (255), categoria varchar (255), catalogo varchar (255), precio varchar (255),fechaRegistro varchar (255), cadenaComercial varchar (255),giro varchar (255),nombreComercial varchar (255), direccion varchar (255), estado varchar (255), municipio varchar (255), latitud varchar (255), longitud varchar (255));"

    # copy s3 file into my_table
    # please take a moment to review the options that are being passed
        aws redshift-data execute-statement \
        --database my_database \
        --db-user adminuser \
        --cluster-identifier mycluster \
        --sql "COPY my_table FROM 's3://BUCKET_NAME/all_data.csv' CREDENTIALS 'aws_access_key_id=ACCESS_KEY;aws_secret_access_key=SECRET_ACCESS_KEY' DELIMITER ',' IGNOREHEADER 1 MAXERROR as 250 CSV;"

        # you can look for errors executing this SQL command on Redshift UI
        "select * from stl_load_errors;"

        # If you do not have your AWS access key look for 
        ~/.aws/credentials 

        # If data was ingested without errors go to Redshift UI and play around with SQL queries on your table
  ```

- For GCP use
```python
bq load \
--autodetect \
--source_format=CSV \
mydataset.mytable \
gs://BUCKET_NAME/all_data.csv

# if schema is not right maybe explicitly declare it
bq load \
--skip_leading_rows=1 \
--source_format=CSV \
mydataset.mytable \
gs://BUCKET_NAME/all_data.csv \
producto:STRING,presentacion:STRING,marca:STRING,categoria:STRING,catalogo:STRING,precio:STRING,fechaRegistro:STRING,cadenaComercial:STRING,giro:STRING,nombreComercial:STRING,direccion:STRING,estado:STRING,municipio:STRING,latitud:STRING,longitud:STRING

# if there is a schema mismatch error try deleting the current table
bq rm -t mydataset.mytable 

# If data was ingested without errors go to BigQUery UI and play around with SQL queries on your table
```
---

#### Useful commands for redshift
- aws redshift-data describe-statement --id #
- aws redshift-data get-statement-result --id#

#### Please try to execute the following Querys:
## Find the top 3 expensive products by estate
```
select 
  rank,
    estado,
    marca, 
    producto, 
    precio
from 
(
  select distinct
    RANK() OVER (PARTITION BY estado order by precio desc) as rank,
    estado,
    marca, 
    producto, 
    precio
  from `data-castle-bravo.mydataset_jbeas.testtable_partitioned`
  where estado <> ''
) as top_marcas
where (rank <= 3)
order by estado
```

## Find the top 10 marcas having estado_id = 14 as a input parameter
```
SELECT 
  a.marca, a.estado, sum(1) as available_products
  FROM `data-castle-bravo.mydataset_jbeas.testtable_partitioned` as a
  inner join `data-castle-bravo.mydataset_jbeas.estados` as b on 
  a.estado = b.estado
  where b.estado_id = 14
  group by a.marca, a.estado 
  order by available_products desc
  limit 10;
```

#### Can you share any improvement updates to the queries?

---

### 5. Clean up

1. Delete files in the buckets
- For AWS use
```python
# Delete a single file
aws s3 rm s3://BUCKET_NAME/all_data.csv
# Delete all files inside a bucket
aws s3 rm s3://BUCKET_NAME --recursive
```
- For GCP use
```python
# you can remove specific object 
gsutil rm -r gs://BUCKET_NAME/all_data.csv
# or recursively delete
gsutil rm -r gs://BUCKET_NAME
```
2. Delete buckets
- For AWS use
```python
aws s3 rb s3://BUCKET_NAME 
```
- For GCP use
```python
gsutil rb gs://BUCKET_NAME
```
3. Delete datawarehouse
 - For AWS use
 ```python
 # on AWS you can directly delete the whole cluster
 aws redshift delete-cluster --cluster-identifier mycluster --skip-final-cluster-snapshot --region us-east-2
 ```
 - For GCP use
```python
# remove the table
bq rm -t mydataset.mytable 

# remove the dataset
bq rm -d mydataset

# if you want to remove dataset and all its tables
bq rm -r -d mydataset
```
### General Tips

- Never perform a query without a where clause, its expensive on resources and money
- Try to avoid select *, select only the fields that are required for the task
- Use table preview as frecuent as possible instead of performing a query to check data types
- check table schema from schema tab instead of performing a query (show create table)
- while querying, try to put partitions first then other fields on where clause
- try to have at least one partition on where clause, when partitios avaiable
- if a query is being executed frecuently, try adding them into a view or temp table

## Querying from Redshift

While querying on Redshift, amount of data processed in unknown till the query is being executed.

In order to check real amount of processed data, you need to check into query details, from redshift main page. 

Runtime may vary depending on cluster size, table size and available worker nodes and memory avaiable.

```
select * from my_table limit 1;
```

in this example, even if this is a simple query, took 7 seconds to get results back, and 37 rows where scanned  making a total of 5.85 KB of scanned data.
same amount of time could be taken to get biger amount of data processed, it depends directly on cluster size and how table is structored.

Like in Bigquery, almost all standart SQL commands are available on redshift too plus another extra unique functions.

you can check translation between BigQuery and Redshit supported SQL here:

https://cloud.google.com/architecture/dw2bq/redshift/redshift-bq-sql-translation-reference.pdf

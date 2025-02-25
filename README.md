## What is this?

This is a guide for creating data products for the OS-Climate Data Mesh.

## Platform Architecture

![The GitOps lifecycle of a data product](/images/PlatformArchitecture.png)

The Data Mesh offers a GitOps-based Data-as-Code approach to creating and deploying data products. 

A Data Product is defined in Git Repository. Its structure is described below. 

At runtime, a Data Product can be deployed into the platform either:

* manually
* automatically, using ArgoCD

Once the Data Product is uploaded into the platform, an Airflow pipeline is executed automatically, triggering the ETL pipeline.


## How are data product definitions structured  

OS-Climate Data Products are defined via code, stored in Git. An example of a data product can be found [here](sample_data_products/pcaf).  

In principle, each data product is structured as a collection of Apache Airflow DAGs, performing one or more of the following:

- Downloading data from an external source and loading into an S3-compatible object store or a local database
- Transforming data into a Parquet-compatible format for Apache Hive or Apache Iceberg
- Defining a table structure on top of the Parquet formatted data and registering it in Trino
- Performing transformations on top of Trino tables, either directly in code or using DBT

The product definitions are deployed in Airflow via a central bucket and can be manually executed. The expected result in each case is:

- the data product itself is deployed as one or more Trino tables, which can be queried using the Trino APIs, including Python and Java clients
- the metadata associated with the data product is deployed in the metadata store (OpenMetadata)
- the lineage of the data product is automatically registered in the metadata store (OpenMetadata)

## A data product in detail 

Here's a complete data product [samples](sample_data_products/pcaf) folder:

```

pcaf
├── dbt
│   ├── pcaf
│   │   ├── analyses
│   │   ├── dbt_project.yml
│   │   ├── macros
│   │   ├── models
│   │   │   └── pcaf
│   │   │       ├── countries.py
│   │   │       ├── pcaf_oecd_agg.sql
│   │   │       ├── pcaf_oecd_staging.sql
│   │   │       ├── pcaf_primap.sql
│   │   │       ├── pcaf_primap_staging.sql
│   │   │       ├── pcaf_unfccc_annexi_staging.sql
│   │   │       ├── pcaf_unfccc_nonannexi_staging.sql
│   │   │       ├── pcaf_unfccc_sovereign_emissions.sql
│   │   │       ├── pcaf_unfccc_with_lulucf.sql
│   │   │       ├── pcaf_unfccc_without_lulucf.sql
│   │   │       ├── pcaf_wdi.sql
│   │   │       ├── pcaf_wdi_staging.sql
│   │   │       └── schema.yml
│   │   ├── package-lock.yml
│   │   ├── packages.yml
│   │   ├── seeds
│   │   ├── snapshots
│  └── tests
│   └── profiles.yml
├── generate docs.py
├── pcaf_ingestion-oecd.py
├── pcaf_ingestion-primap.py
├── pcaf_ingestion-unfccc.py
├── pcaf_ingestion-worldbank.py
└── pcaf_unfccc_dbt_transformation.py

```

This data product consists of two sets of files:
* Data pipelines represented as Apache Airflow DAGs (all `pcaf_*.py` files)
* `dbt` transformation definitions; the `dbt` transformations are called by Apache Airflow data pipelines                   

## What can product definitions contain

Product definitions are written as Apache Airflow DAGs in Python. There are several ways to define tasks:

- Very simple tasks that do not require additional dependencies and are not compute intensive can be defined inline as part of the DAG. Possible examples include: downloading small files from external sources, basic conversions, etc
- Complex tasks that do require additional dependencies or are compute intensive can be defined in separate container images and executed using either the DockerOperator (for local deployment) or the KubernetesOperator (when deployed on the platform)
- (Work in progress) In the future, the project will provide predefined containerized images for generic tasks, such as DBT transformations.

## A basic end to end example 

Here is a simple example for a data product definition from the PCAF repository. It describes a process to download and register an intermediate data product as a Trino table. The whole source code is available [here]().

First, we create a task to download the content of the CSV file, transform it into Parquet, and upload it into an S3 bucket that is accessible to Trino.

    @task(
        task_id="load_data_to_s3_bucket"
    )
    def load_data_to_s3_bucket():
        import pandas as pd
        import zipfile
        import urllib.request
        from airflow.providers.amazon.aws.hooks.s3 import S3Hook
        url = "https://api.worldbank.org/v2/country/all/indicator/NY.GDP.MKTP.CD;NY.GDP.MKTP.PP.CD?source=2&downloadformat=csv"
        local_file = "worldbank.zip"
        if os.path.isfile(local_file):
             os.remove(local_file)
        if not os.path.isfile(local_file):
            with urllib.request.urlopen(url) as file:
                with open(local_file, "wb") as new_file:
                    new_file.write(file.read())
                new_file.close()

        zipfile = zipfile.ZipFile(open(local_file, "rb"))

        s3_hook = S3Hook(aws_conn_id='s3')
        for parquet_file_name in zipfile.namelist():
            print(parquet_file_name)
            if "Metadata" not in parquet_file_name:
                with zipfile.open(parquet_file_name, "r") as file_descriptor:
                    df = pd.read_csv(file_descriptor, skiprows=4, quotechar= '"')
                    parquet_bytes = df.to_parquet(compression='gzip')
                    s3_hook.load_bytes(parquet_bytes, bucket_name= "pcaf", key=f"raw/worldbank/worldbank.parquet", replace=True)

Next, we define a task for creating a schema to store our intermediate data product:

    trino_create_schema = TrinoOperator(
        task_id="trino_create_schema",
        trino_conn_id="trino_connection",
        sql=f"CREATE SCHEMA IF NOT EXISTS hive.pcaf WITH (location = 's3a://pcaf/data')",
        handler=list,
    )

Then, we define a task to register this table into Trino:

    trino_create_worldbank_table = TrinoOperator(
        task_id="trino_create_worldbank_table",
        trino_conn_id="trino_connection",
        sql=f"""create table if not exists hive.pcaf.worldbank (
                        "Country Name" varchar,"Country Code" varchar,"Indicator Name" varchar,"Indicator Code" varchar,"1960" double,"1961" double,"1962" double,"1963" double,"1964" double,"1965" double,"1966" double,"1967" double,"1968" double,"1969" double,"1970" double,"1971" double,"1972" double,"1973" double,"1974" double,"1975" double,"1976" double,"1977" double,"1978" double,"1979" double,"1980" double,"1981" double,"1982" double,"1983" double,"1984" double,"1985" double,"1986" double,"1987" double,"1988" double,"1989" double,"1990" double,"1991" double,"1992" double,"1993" double,"1994" double,"1995" double,"1996" double,"1997" double,"1998" double,"1999" double,"2000" double,"2001" double,"2002" double,"2003" double,"2004" double,"2005" double,"2006" double,"2007" double,"2008" double,"2009" double,"2010" double,"2011" double,"2012" double,"2013" double,"2014" double,"2015" double,"2016" double,"2017" double,"2018" double,"2019" double,"2020" double,"2021" double,"2022" double
                        )
                        with (
                         external_location = 's3a://pcaf/raw/worldbank/',
                         format = 'PARQUET'
                        )""",
        handler=list,
        outlets=['hive.pcaf.worldbank']
    )


Finally, we define the end-to-end flow as follows:

    load_data_to_s3_bucket()  >> trino_create_schema >> trino_create_worldbank_table

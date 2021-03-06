# Redshift Spectrum PoC Environment

The following are instructions to rebuild the Redshift Spectrum PoC environment that was presented in the AWS Big Data Blog: ["Leveraging Redshift Spectrum to Enchance Customer 360: Insights from Data Lake to Data Warehouse."](https://aws.amazon.com/blogs/big-data/from-data-lake-to-data-warehouse-enhancing-customer-360-with-amazon-redshift-spectrum/)

## Prerequisites

  - Access to an AWS Account
  - Your own EC2 key pair
  - Sufficient resources to support the AWS resources illustrated and discussed below.
  - Permissions to:
      - Execute the provided [Cloud Formation template](cf-templates/redshift-spectrum-poc-env.template). The required role and policies with [minimal privileges are provided](cf-templates/template-exec-policy.json)
      - Permissions to create your own S3 buckets, sync the datasets from the provided S3 sources to your buckets, and rights to load the data into Redshift.

## Prepare S3 Dataset

The datasets for the PoC range from 150-700GB, so I advise starting this step early and work on setting the other parts of the environment in parallel. 5 datasets are made available, but only two are presented in the blog post: clickstream-csv10 and clickstream-parquet1. All datasets contain the same data, but they vary in terms of data format, and number of files. As described in the article, these characteristics effect performance and are better optimized for different scenarios. 

Furthermore, as described in the article:
-	The data is a modified version of the uservisits data-set from [AMPLab’s Big Data Benchmark](https://amplab.cs.berkeley.edu/benchmark/), which was generated by [Intel’s Hadoop benchmark tools](https://github.com/intel-hadoop/HiBench). 
-	Changes were minimal, so that existing test harnesses for this test can be adapted:
-	Increased the 751,754,869-row dataset 5X to 3,758,774,345 rows.
-	Added surrogate keys to support joins with customer and time dimensions. These keys were distributed evenly across the entire dataset to represents user visits from 6 customers over 7 years. 
-	Values for the “visitDate” column were replaced to align with the 7-year timeframe, and align with the time surrogate key. 

### Data sources:

The datasets reside in US-East-1, and you must be an authenticated AWS user to access these data sets.

 1. clickstream-csv10:
    - location: s3://redshift-spectrum-bigdata-blog-datasets/clickstream-csv10
    - format: csv
    - file partitioning scheme: 10 files for each customer and year/month
    - dataset size: 615.9 GB
    - file size: ~90-130 MB
 2. clickstream-parquet10:
    - location: s3://redshift-spectrum-bigdata-blog-datasets/clickstream-parquet10
    - format: parquet
    - file partitioning scheme: 10 files for each customer and year/month
    - dataset size: 115.4 GB
    - file size: ~15-30 MB
 3. clickstream-parquet1:
    - location: s3://redshift-spectrum-bigdata-blog-datasets/clickstream-parquet10
    - format: parquet
    - file partitioning scheme: 1 file for each customer and year/month
    - dataset size: 116.4 GB
    - file size: 200-250 MB
 4. clickstream-csv20
    - location: s3://redshift-spectrum-bigdata-blog-datasets/clickstream-csv20
    - format: csv
    - file partitioning scheme: 20 files for each customer and year/month
    - dataset size: 615.9 GB
    - file size: ~60 MB
 5. clickstream-parquet20
    - location: s3://redshift-spectrum-bigdata-blog-datasets/clickstream-parquet20
    - format: parquet
    - file partitioning scheme: 20 files for each customer and year/month
    - dataset size: 288.5 GB
    - file size: ~20-30 MB

### Steps:

1. Create a bucket for each scenario that you want to test. For instance, if you decide to test the two scenarios described in the article, create two buckets like <my-unique_id>-redshift-spectrum-datastore-csv10 and <my-unique_id>-redshift-spectrum-datastore-parquet1 for housing the clickstream-csv10 and clickstream-parquet1 datasets respectively.
2. Copy the public datasets over to your bucket. You must be an authenticated user to download this data set. In the example scenario, run these two commands using the AWS CLI with the appropriate permissions:

     - *aws s3 sync --source-region us-east-1 s3://redshift-spectrum-bigdata-blog-datasets/clickstream-csv10 s3://`<my-unique-id>`-redshift-spectrum-datastore-csv10
  
     - *aws s3 sync --source-region us-east-1 s3://redshift-spectrum-bigdata-blog-datasets/clickstream-parquet1 s3://`<my-unique-id>`-redshift-spectrum-datastore-parquet1
     
This copy process may take hours, so you can let the sync run while you proceed with the other steps to build out your environment.     

## Provision Infrastructure

A Cloud Formation template, [redshift-spectrum-poc-env.template](cf-templates/redshift-spectrum-poc-env.template), has been provided under the cf-templates directory to build the following environment:

![poc environment diagram](/images/redshift-spectrum-poc-env-diagram.png)

The template is designed to create an isolated environment, so a new VPC is carved out with the typical networking elements diagram above such as public and private subnets, IGW and routing constructs. The primary resources deployed are a bastion host and a Redshift cluster deployed in a private subnet. The instance types and size of the cluster is spcified by you when you run the template. The default values in the template are the primary configurations used in most of the experiments presented in the article.

Note that since this is a PoC environment wthout HA requirements, minimal resources are deployed in a single Availability Zone.

### Steps:

1. [Download the Cloud Formation template provided](cf-templates/redshift-spectrum-poc-env.template).
2. Run the template with the necessary permissions to create the resources illustrated above. If your account doesn't have sufficient permissions, [create a role from this linked document to provide least privileges to execute the template](cf-templates/template-exec-policy.json). The template requires you to provide an AMI ID. Please provide the latest Microsoft Windows Server 2016 Base AMI. You can find this AMI from the console by searching on "AMI-Name: Windows_Server-2016-English-Full-Base-." The AWS documentation provides other options as well: http://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/finding-an-ami.html

## Setup Database Client

You're free to use any client to interface with Redshift. If you don't have a preference or know of any options, follow the instructions in our [documentation to install SQL Workbench](http://docs.aws.amazon.com/redshift/latest/mgmt/connecting-using-workbench.html). 

### Steps:

1. Validate that your template was successfully executed, and the bastion host is deployed. This instance should be labeled "Redshift Spectrum POC Bastion Host."
2. Use Microsoft Remote Desktop to log on to the Windows bastion host using your private key from the EC2 key pair that you specified when initiating the Cloud Formation template.
3. Install your client or SQL Workbench [(via instructions referenced above)](http://docs.aws.amazon.com/redshift/latest/mgmt/connecting-using-workbench.html) with the latest Redshift JDBC drivers: 

## Create the Redshift Datawarehouse

The Redshift cluster has been provisioned by Cloud Formation, but additional steps have to be taken to build the dimensional tables and loading the dataset that was described in the article.

This PoC leverages the benchmarking environment documented on AWS's website. You can follow the whole process described [here](http://docs.aws.amazon.com/redshift/latest/dg/tutorial-tuning-tables.html), or run the minimal scripts provided under the /sql-scripts folder.

### Steps:
1. Create the dimension tables. Log into your client and run the create table commands provided in the [create-dimensions.sql](sql-scripts/create-dimensions.sql) script.
2. Load the star schema benchmark data set into your cluster via the copy command. You can run the commands provided in the [load-dimension-data.sql](sql-scripts/load-dimension-data.sql) script, which will require you to provide the appropriate access and secret keys. Specifically, you need to replace the text <Your-Access-Key-ID> and <Your-Secret-Access-Key> in the script with the appropriate access and secret keys.

## Define External Redshift Tables

Redshift Spectrum tables are created differently than native Redshift tables, and are defined as "External" tables. Schema information is stored externally in either a Hive metastore, or in Athena. 

### Steps:

1. Define a schema by running the following command:

create external schema clickstream from data catalog 
database 'rs_spectrum_clickstreams' 
iam_role '`<Your Redshift IAM Role ARN Generated by the CF Template>`'
create external database if not exists;

You will need replace `<Your Redshift IAM Role ARN Generated by the CF Template>` above with the ARN of the IAM role created for your cluster. You can find this value in the "Output" tab of the active Cloud Formation stack under the variable "RedshiftClusterIAMRole." The value should look something like:

            arn:aws:iam::`<Your Account #>`:role/redshift-spectrum-poc2-rRedshiftSpectrumRole-RD4GHLO9Z9EN

This command will create a database in Athena where schema and table metadata will be stored.

2. Create your external tables:

Scripts have been provided under the [/sql-scripts directory](sql-scripts/) to define the external tables for each of the datasources as described in the table below: 

| Dataset                                                            | DML Script                       |
|--------------------------------------------------------------------|----------------------------------|
| s3://redshift-spectrum-bigdata-blog-datasets/clickstream-csv10     | create-clickstream-csv10.sql     | 
| s3://redshift-spectrum-bigdata-blog-datasets/clickstream-csv20     | create-clickstream-csv20.sql     |
| s3://redshift-spectrum-bigdata-blog-datasets/clickstream-parquet1  | create-clickstream-parquet1.sql  |
| s3://redshift-spectrum-bigdata-blog-datasets/clickstream-parquet10 | create-clickstream-parquet10.sql |
| s3://redshift-spectrum-bigdata-blog-datasets/clickstream-parquet20 | create-clickstream-parquet20.sql |

You will need to **modify these scripts to reference your unique bucket and table names**. For instance, in the [create-clickstream-parquet1.sql](sql-scripts/create-clickstream-parquet1.sql) file, the script consists of commands to define the table and a large number of commands to add partitions to it. This is the general composition of all these scripts. The modifications that have to be made are as follows:

1. Modfiy the S3 bucket location in the create table command:

  **'s3://redshift-spectrum-datastore-parquet1/'** below has to be replaced with the bucket that is holding your copy of the   parquet-1 dataset.

    create external table clickstream.uservisits_parquet1(
    custKey int4,
    yearmonthKey int4,
    visitDate int4,
    adRevenue float,
    countryCode char(3),
    destURL varchar(100),
    duration int4,
    languageCode char(6),
    searchWord varchar(32),
    sourceIP varchar(116),
    userAgent varchar(256))
    partitioned by(customer int4, visitYearMonth int4)
    stored as parquet
    location 's3://redshift-spectrum-datastore-parquet1/'
    table properties ('numRows'='3758774345');

2. Modify the "add partition" commands:

There are 504 commands in each script that add partitions to these external tables.

Here is an example of one of the commmands:

    alter table clickstream.uservisits_parquet1 add partition(customer=1, visitYearMonth=199201) 
    location 's3://redshift-spectrum-datastore-parquet1/52a17f02aa5675c8399b182d9351da5a79b0522ca1080270c15b1767031babf4/customer=1/visitYearMonth=199201/';
  
As before, you will need to do a find and replace all on the bucket names to match yours. In this case, **s3://redshift-  spectrum-datastore-parquet1** has to be replaced accordingly.  

## Conclusion

Have fun and profit from Redshift Spectrum. If you hit any snags or wish for additional guidance, reach out to your AWS Solutions Architect. They will reach out to me as needed to assist.

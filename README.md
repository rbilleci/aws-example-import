
# Example AWS Data Pipeline

### Notice

* **This project is an example and not intended for production deployments.**
* This project has not been stress tested. You will need to optimize this based on your workload.
* This project has not gone through any QA. Only basic error cases have been tested.
* This project assumes a certain table schema. It will need to be customized to support multiple table schemas.

### Overview

This project demonstrates a serverless AWS data pipeline,
based on a scenario where the source system is providing
full table exports to S3 on a regular schedule.

This project uses Amazon Athena to compare two export files in S3, 
compute the delta, 
then publishes operations to an Amazon Kinesis stream, 
where an AWS Lambda function will apply the operations to a target system. 

### When this method is appropriate
This method for computing deltas may be advantageous given the following conditions:
1. When dealing with large export files. Amazon Athena can calculate the delta between two 1 million-row files within
seconds.
2. When the source system can only provide full exports

### Requirements

This method only works if each table has a primary key. The example assume a column named 'PK', but it can
be changed to use any column or even a composite primary key.

### Limitations of example

This example demonstrates the end-to-end flow from Amazon S3 to an Amazon Kinesis stream.
Each record written to the stream has an operation (INSERT, UPDATE, DELETE).  
To write the changes to a database, you must add an AWS Lambda function to read the operations from the stream.

The **publish-updates** function in this example must be optimized to avoid potential timeouts when working with
large numbers of rows.


# Data Flow

1. The source system performs an export and pushes the export file to an S3 bucket, with a path of
s3://<BUCKET>/<TENANT>/<TABLE>/<EXPORT_VERSION>/<FILENAME.gz>. This example uses CSV files compressed with GZIP.
2. A CloudWatch event triggers, executing an AWS Step Function, passing in the S3 key of the uploaded file.
3. The AWS Step Function registers the new file in an Amazon DynamoDB table named 'registry'
4. The AWS Step Function attempts to acquire a lock for the tenant/table pair, to ensure that multiple import jobs 
do not run concurrently for the same tenant/table. Note: concurrent imports will run for different tables. 
5. The AWS Step Function executes an Amazon Athena query to compute the delta. When there is no previous import job, 
query returns all rows.
6. An AWS Lambda function reads the results from the Amazon Athena query, and writes those to an Amazon Kinesis stream
7. This step is up to you. You need to read the data from the stream, and write the operations to your data source.

# Testing it out

1. Run the cfn.yaml to deploy the infrastructure. Provide an S3 bucket name where import files will be uploaded
2. Copy files from the **data** directory to S3 one at a time
   After copying each file, examine the history of the AWS Step Function and review the
   status, inputs, outputs, and logs.
   Notice below, when you copy the files, keep the directory structure intact, but omit the **data** directory. the
   top-level directory in your S3 bucket should be the name/id of a tenant, in our case the tenant name/id is just 'tenant'.

    a. Add **data/tenant/test_table/20201230_V0001/export1.csv.gz** to **s3://<BUCKET>/tenant/test_table/20201230_V0001/export1.csv.gz** 
    
    b. Add **data/tenant/test_table/20201230_V0002/export2.csv.gz** to **s3://<BUCKET>/tenant/test_table/20201230_V0002/export2.csv.gz**
     
    c. Add **data/tenant/test_table/20201231_V0001/export3.csv.gz** to **s3://<BUCKET>/tenant/test_table/20201231_V0001/export3.csv.gz**
    
3. Examine the CloudWatch logs for the publish-updates function to see statistics on records written to the Amazon Kinesis stream   
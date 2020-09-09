# Cost and Usage report standard queries in AWS Control Tower enviornment - Set up

As you might know, AWS Cost and Usage Reports tracks your AWS usage and provides estimated charges associated with your account. Each report contains line items for each unique combination of AWS products, usage type, and operation that you use in your AWS account. You can customize the AWS Cost and Usage Reports to aggregate the information either by the hour or by the day.

AWS Cost and Usage Reports can do the following:

- Deliver report files to your Amazon S3 bucket

- Update the report up to three times a day

- Create, retrieve, and delete your reports using the AWS CUR API Reference

## How Cost and Usage Reports work
Each update is cumulative, so each version of the Cost and Usage Reports includes all of the line items and information from the previous version. The reports generated throughout the month are estimated, and subject to change during the rest of the month as you continue to use your AWS services. AWS finalizes the report at the end of each month. Finalized reports have the calculations for your blended and unblended costs, and cover all of your usage for the month.

AWS might update reports after they have been finalized if AWS applies refunds, credits, or support fees to your usage for the month. You can set this as a preference when creating or editing your report. The report is available within 24 hours of the date that you create a report on the Cost & Usage Reports page of the Billing and Cost Management console.

## How can take real insights from this in a "readable" way?

What if we could write, well-known SQL queries to know how much do we spent in a particular AWS Account, for a particular service in a particular month? Or even better, what is the estimated consumnption through all the AWS accounts in my organization year to date? Or to know which Amazon EC2 instance has been consuming the most in my workload? We could continue asking this type of questions forever, and everyone could have a different to propose.

Amazon Athena is an interactive query service that makes it easy to analyze data in Amazon S3 using standard SQL. Athena is serverless, so there is no infrastructure to manage, and you pay only for the queries that you run.

So, it seems that we could potentially run SQL queries easily for those report files in S3, right?

Athena is easy to use. Simply point to your data in Amazon S3, define the schema, and start querying using standard SQL. Most results are delivered within seconds. 

Problem is, what happens if the schema changes? And thinking on the CUR files, they will change most likely. Why we say this? Basically if there is a new resource launched using an AWS service that we having been using before we created the schema in Amazon Athena, the columns in the report for that particular service won't be available in Athena to query.

As mentioned, CUR files can be delivered to Amazon S3, this means that there is an event every time a report file is created or updated in that bucket. Thinking on event driven architectures, we can trigger from that event a crawler from AWS Glue, so that tables in Amazon Athena can be updated.

Problem solved.

## Ok. How can I set up all this?

You might be asking this question yourself... You probably know SQL, but no clue about Amazon Athena, or AWS Glue... there are some good news: what you need to do is pretty straight forward and doesn't need you to learn everything about those services.

First things first. You'll need to create an Amazon S3 bucket and then the CUR report. 
As we are configuring this for an AWS Control Tower environment, you will need to create the AWS CUR set up in the management account of your AWS Organization (previously named "master" account").
I won't go into the steps as it is pretty detailed here:

[Creating a new Amazon S3 bucket for your reports](https://docs.aws.amazon.com/cur/latest/userguide/create-athena-bucket.html "Creating a new Amazon S3 bucket for your reports")

[Creating new Cost and Usage Reports](https://docs.aws.amazon.com/cur/latest/userguide/create-athena-cur.html "Creating new Cost and Usage Reports")

> Be sure you select: "Include resource IDs" in **Additional report details**  
> and "Athena" in **Enable report data integration for**

Now that you have set up the reports, what?

Well, the Billing and Cost Management team has made things easy for us, but creating an AWS CloudFormation template (infrastructure as code), so we can automatically launch all resources needed to create the AWS Glue Crawler, Amazon Glue Database and the AWS Lambda event (which will trigger the Crawler every time a new report is created/updated).

> ### Important
> Before moving on to this procedure, you must wait for the first AWS CUR to be delivered to your Amazon S3 bucket. It might take up to 8 hours for AWS to deliver your first report.  

Be sure to align the Region when using the template.

**To use the Athena AWS CloudFormation template**

1. Open the [Amazon S3 console](https://console.aws.amazon.com/s3/ "Amazon S3 console")

2. From the list of buckets, choose your bucket name.

3. Navigate your folders until you find the .yml template file.

    The .yml file is usually found in the prefix name folder, or report name folder. The location varies depending on how your report is formatted.

4. Select the .yml template file.

5. Select **download**.

6. Open the [AWS CloudFormation console](https://console.aws.amazon.com/cloudformation "AWS CloudFormation console")

7. Choose **Create New Stack** if you have never used AWS CloudFormation before. Otherwise, choose **Create Stack**.

8. Under **Prepare template**, choose **Template is ready**.

9. Under **Template source**, choose **Upload a template file**.

10. Select *Choose file**.

11. Choose the downloaded .yml template, and then choose **Open**.

12. Choose **Next**.

13. For **Stack name**, enter a name for your template and choose *Next**.

14. Choose **Next**.

15. At the bottom of the page, select **I acknowledge that AWS CloudFormation might create IAM resources**. This template creates the following resources (names will vary as they use random identifiers or the name of the CUR report):

    - Three IAM roles  
        a. AWSCURCrawlerComponentFunction  
        b. AWSCURCrawlerLambdaExecutor  
        c. AWSS3CURLambdaExecutor  

    - An AWS Glue database  
        a. athenacurcfn

    - An AWS Glue crawler  
        a. AWSCURCrawler

    - Two Lambda functions  
        a. AWSCURInitializer (this AWS Lambda Function is triggered by Amazon S3 when a new CUR report file is uploaded (or updated) and starts the AWS Glue Crawler, so it can update the Amazon Athena data catalog).  
        b. AWSS3CURNotification.  

    - An Amazon S3 notification  
        a. *random*  

16. Choose **Create**.

Fantastic, now what?

## Creating and launching queries to the CUR files

Now is time to get those SQL skills and start building some queries... let's go to the [Amazon Athena console](https://console.aws.amazon.com/athena/ "Amazon Athena console") and make sure we selected the right AWS region and also the Database created by the AWS CloudFormation template:

![alt text](https://github.com/denegrij/CUR/blob/master/CUR_Database.png "CUR Database")

You can find here an [Amazon Athena user guide](https://docs.aws.amazon.com/athena/latest/ug/getting-started.html "Amazon Athena user guide"), but I think that *"learn by looking others"* it's better :)

Here are some examples:

### Monthly Spend by Service

```sql
/* TITLE: Monthly Spend by Service */
/* DESCRIPTION: Monthly spend by service in the AWS Organization */

SELECT line_item_product_code AS Service,
         round(sum(line_item_blended_cost),
         3) AS Cost,
         month AS Month,
         line_item_usage_account_id AS AWS_Account
FROM ct_monthly_spend
WHERE year='2020'
        AND month='9'
GROUP BY  line_item_product_code, month,line_item_usage_account_id
HAVING sum(line_item_blended_cost) > 0.001
ORDER BY  line_item_usage_account_id,line_item_product_code
````

### Monthly Spend by Service in Account

```sql
/* TITLE: Monthly Spend by Service in Account */
/* DESCRIPTION: Monthly spend by service in a particular AWS Account per region */

SELECT line_item_product_code AS Service,
         round(sum(line_item_blended_cost),
         3) AS Cost,
         month AS Month,
         line_item_usage_account_id AS AWS_Account
FROM ct_monthly_spend
WHERE year='2020'
        AND month='8'
        AND line_item_usage_account_id='130395768702'
GROUP BY  line_item_product_code, month,line_item_usage_account_id
HAVING sum(line_item_blended_cost) > 0.001
ORDER BY Cost desc,line_item_product_code
````

### Monthly Service Spend in Account

```sql
/* TITLE: Monthly Service Spend in Account */ 
/* DESCRIPTION: Monthly spend by region for a particular service IN a particular AWS account */

SELECT line_item_product_code AS Service,
         round(sum(line_item_blended_cost),
         3) AS Cost,
         month AS Month,
         product_region AS Region,
         line_item_line_item_description AS Description
FROM ct_monthly_spend
WHERE year='2020'
        AND month='9'
        AND line_item_usage_account_id='123456789123'
        AND line_item_product_code = 'AWSCloudTrail'
GROUP BY  line_item_product_code, month,product_region,line_item_line_item_description
HAVING sum(line_item_blended_cost) > 0.001
ORDER BY  line_item_product_code;
```

### Monthly S3 Spend

```sql
/* TITLE: Monthly S3 Spend */
/* DESCRIPTION: Monthly spend by S3 bucket IN the AWS Organization */

SELECT line_item_resource_id AS S3_Bucket_Name,
         round(sum(line_item_blended_cost),
         3) AS Cost,
         line_item_usage_account_id AS AWS_Account,
         product_region AS Region
FROM ct_monthly_spend
WHERE year='2020'
        AND month='9'
        AND line_item_product_code='AmazonS3'
GROUP BY  line_item_resource_id, line_item_usage_account_id, product_region
HAVING sum(line_item_blended_cost) > 0.001
ORDER BY  line_item_usage_account_id, cost DESC
````

---

Now, for the next one, I was a little bit more creative. As I needed to know the AWS Accounts by their name, not the AWS Account number. Problem is that CUR files doesn't export that information. So, I created in Amazon Athena a new table with two columns: the AWS Account number and the AWS Account name (wow, that's creative?)

Pretty easy, first I uploaded a csv file with couple of AWS Accounts to the Amazon S3 buckets (in this example in the "Accounts" directory):

| Account name        | Account ID
| ------------- |:-------------:|
| Master      | 123456789123
| Network      | 123456789123

![alt text](https://github.com/denegrij/CUR/blob/master/accounts.png "Accounts mapping")

```sql
CREATE EXTERNAL TABLE `account_name`(
  `account_name` string, 
  `account_id` string)
ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.mapred.TextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://denegrij-ct-cur/Accounts'
TBLPROPERTIES (
  'has_encrypted_data'='false', 
  'transient_lastDdlTime'='1597156500')
  ````

Then I can insert rows into that table whenever I create a new AWS Account in my AWS Organization (or remove when I deleted):

```sql
INSERT INTO account_name 
VALUES ('Any_Oter_Account','123456789123')
````

You might be asking... how that could scale?... well... not really. Remember we talked about *event driven* architectures? Well, AWS Control Tower has what we call [Lifecycle Events](https://docs.aws.amazon.com/controltower/latest/userguide/lifecycle-events.html "Lifecycle Events"). 

So, if we take a closer look, we can find an event called [CreateManagedAccount](https://docs.aws.amazon.com/controltower/latest/userguide/lifecycle-events.html#create-managed-account "CreateManagedAccount") which, as its name points out is an event that gets generated every time AWS Control Tower successfully completed an action to create and provision a new account using account factory.

That's great! so now, we can update our Amazon Athena *account_name* table with the newly created account. But how?
You could trigger an AWS Lambda function to *append* a line in the file that contains your accounts and voilá!

[Here](https://controltower.aws-management.tools/automation/lifecycle/) you can find more examples of using AWS Control Tower Lifecycle events to trigger other actions.

An AWS Lambda function example to append the newly created AWS Account info to the Amazon S3 file could be (still needs the newly created account number and name as parameters):

```python

import boto3
import csv 

# call s3 bucket
s3 = boto3.resource('s3')
bucket = s3.Bucket(BUCKET_NAME) # Enter your bucket name, e.g 'Data'

# key path, e.g.'customer_profile/Reddit_Historical_Data.csv'
key = 'KEY_PATH'

def lambda_handler(event,context):

    # download s3 csv file to lambda tmp folder
    local_file_name = '/tmp/test.csv' #
    s3.Bucket(BUCKET_NAME).download_file(key,local_file_name)
    
    # list you want to append
    lists = ['NewAccount','123456789123'] 
    
    # write the data into '/tmp' folder
    with open('/tmp/test.csv','r') as infile:
        reader = list(csv.reader(infile))
        reader = reader[::-1] # the date is ascending order in file
        reader.insert(0,lists)
    
    with open('/tmp/test.csv', 'w', newline='') as outfile:
        writer = csv.writer(outfile)
        for line in reversed(reader): # reverse order
            writer.writerow(line)
    
    # upload file from tmp to s3 key
    bucket.upload_file('/tmp/test.csv', key)
    
    return {
        'message': 'success!!'
    }
````
    
---

### Monthly Account Spend

```sql
/* TITLE: Monthly Account Spend */
/* DESCRIPTION: Monthly spend by account IN the AWS Organization */

SELECT ct_monthly_spend.line_item_usage_account_id AS AWS_Account,
        account_name.account_name AS Account_Name,
         round(sum(ct_monthly_spend.line_item_blended_cost),
         3) AS Cost,
         ct_monthly_spend.month AS Month
FROM ct_monthly_spend
INNER JOIN account_name
    ON ct_monthly_spend.line_item_usage_account_id = account_name.account_id
WHERE year='2020'
        AND month='9'

GROUP BY  line_item_usage_account_id, Month,Account_Name
HAVING sum(line_item_blended_cost) > 0.001
ORDER BY  Cost DESC
````

### YTD Account Spend

```sql
/* TITLE: YTD Account Spend */
/* DESCRIPTION: YTD spend by account IN the AWS Organization */

SELECT ct_monthly_spend.line_item_usage_account_id AS AWS_Account,
        account_name.account_name AS Account_Name,
         round(sum(ct_monthly_spend.line_item_blended_cost),3) AS Cost
FROM ct_monthly_spend
INNER JOIN account_name
    ON ct_monthly_spend.line_item_usage_account_id = account_name.account_id
WHERE year='2020'
GROUP BY  Account_Name,line_item_usage_account_id
HAVING sum(line_item_blended_cost) > 0.001
ORDER BY  Cost DESC

````

### Monthly EC2 Instance Spend

```sql
/* TITLE: Monthly EC2 Instance Spend */ 
/* DESCRIPTION: Monthly spend in EC2 instances in the AWS Organization */

SELECT line_item_resource_id AS Instance_ID,
         round(sum(line_item_blended_cost),
         3) AS Cost,
         line_item_availability_zone AS AZ,
         line_item_usage_account_id AS AWS_Account
FROM ct_monthly_spend
WHERE year='2020'
        AND month='9'
        AND line_item_product_code='AmazonEC2'
        AND line_item_resource_id LIKE 'i-%'
GROUP BY  line_item_resource_id, line_item_usage_account_id, line_item_availability_zone
HAVING sum(line_item_blended_cost) > 0.0001
ORDER BY  line_item_usage_account_id,line_item_resource_id, cost DESC

````

### Monthly EBS Volume Spend

```sql
/* TITLE: Monthly EBS Volume Spend */
/* DESCRIPTION: Monthly spend in EBS Volumes IN the AWS Organization by EBS Volume ID */

SELECT line_item_resource_id AS EBS_Volume_ID,
         round(sum(line_item_blended_cost),
         3) AS Cost,
         product_region AS Region,
         line_item_line_item_description AS Description,
         line_item_usage_account_id AS AWS_Account
FROM ct_monthly_spend
WHERE year='2020'
        AND month='7'
        AND line_item_product_code='AmazonEC2'
        AND line_item_resource_id LIKE 'vol-%'
GROUP BY  line_item_line_item_description,line_item_resource_id, line_item_usage_account_id, product_region
HAVING sum(line_item_blended_cost) > 0.0001
ORDER BY  line_item_usage_account_id, cost DESC
````

---

**Did you like Amazon Athena and want to learn more?**  
We have published a [full workshop dedicated to Amazon Athena](https://athena-in-action.workshop.aws/ "Amazon Athena Workshop"), covering from the basics to run machine learning queries.

---

But... there is always a *but*... you might be thinking how to view this information in a more ~~less-geek~~user friendly way

Well, that's not hard to do with [Amazon QuickSight](https://aws.amazon.com/quicksight/)

I won't go this time to deep on how to do it, but you can basically chose Amazon Athena as data source and start building your visuals:

1. [Creating a Dataset Using Amazon Athena Data](https://docs.aws.amazon.com/quicksight/latest/user/create-a-data-set-athena.html)
2. [Creating an Amazon QuickSight Visual](https://docs.aws.amazon.com/quicksight/latest/user/creating-a-visual.html)

Examples:

![alt text](https://github.com/denegrij/CUR/blob/master/Dashboard1.png "Dashboard example 1")
![alt text](https://github.com/denegrij/CUR/blob/master/Dashboard2.png "Dashboard example 1")

---

Once again, if you want to learn more, in the Amazon Athena workshop pointed out before, there is a very detailed [dedicated lab about Amazon QuickSight!](https://athena-in-action.workshop.aws/30-basics/307-quicksight.html "Amazon QuickSight Workshop").

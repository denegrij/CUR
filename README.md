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

5. Select download.

6. Open the AWS CloudFormation console at https://console.aws.amazon.com/cloudformation.

7. Choose Create New Stack if you have never used AWS CloudFormation before. Otherwise, choose Create Stack.

8. Under Prepare template, choose Template is ready.

9. Under Template source, choose Upload a template file.

10. Select Choose file.

11. Choose the downloaded .yml template, and then choose Open.

12. Choose Next.

13. For Stack name, enter a name for your template and choose Next.

14. Choose Next.

15. At the bottom of the page, select I acknowledge that AWS CloudFormation might create IAM resources. This template creates the following resources:

    - Three IAM roles

    - An AWS Glue database

    - An AWS Glue crawler

    - Two Lambda functions

    - An Amazon S3 notification

16. Choose Create.

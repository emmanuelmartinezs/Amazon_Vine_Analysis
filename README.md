# Furniture Products - Amazon Vine Analysis

## Overview of Project
Since your work with Jennifer on the SellBy project was so successful, you’ve been tasked with another, larger project: analyzing Amazon reviews written by members of the paid Amazon Vine program. The Amazon Vine program is a service that allows manufacturers and publishers to receive reviews for their products. Companies like SellBy pay a small fee to Amazon and provide products to Amazon Vine members, who are then required to publish a review.

In this project, you’ll have access to approximately 50 datasets. Each one contains reviews of a specific product, from clothing apparel to wireless products. You’ll need to pick one of these datasets and use PySpark to perform the ETL process to extract the dataset, transform the data, connect to an AWS RDS instance, and load the transformed data into pgAdmin. Next, you’ll use PySpark, Pandas, or SQL to determine if there is any bias toward favorable reviews from Vine members in your dataset. Then, you’ll write a summary of the analysis for Jennifer to submit to the SellBy stakeholders.

## Deliverables:
This new assignment consists of two technical analysis deliverables and a written report.

1. ***Deliverable 1:*** Perform ETL on Amazon Product Reviews
2. ***Deliverable 2:*** Determine Bias of Vine Reviews
3. ***Deliverable 3:*** A Written Report on the Analysis [README.md](https://github.com/emmanuelmartinezs/Amazon_Vine_Analysis)


## Deliverables:
This new assignment consists of three technical analysis deliverables and a proposal for further statistical study. You’ll submit the following:

* Data Source: `SQL table schema.csv` and `Amazon ETL starter code.csv`
* Data Tools:  `Amazon_Reviews_ETL.ipynb` and `Vine_Review_Analysis.ipynb`.
* Software: `Python 3.9`, `Visual Studio Code 1.50.0`, `Anaconda 4.8.5`, `Jupyter Notebook 6.1.4` and `Pandas`


## Resources and Before Start Notes:

![logo](https://github.com/emmanuelmartinezs/Amazon_Vine_Analysis/blob/main/Resources/Images/Vine-Header.png?raw=true)


### Cloud Storage with S3 on AWS
#### Database Versus Data Storage

Data storage holds raw data such as CSVs, Excel files, and JavaScript Object Notation (JSON) files. Think of your own computer file system where you keep a ton of files as data storage. This data doesn't need to be queried and analyzed for business decisions. The files still have structure and can be reviewed, but not nearly as efficiently as a database.

A database contains cleaned, related information in tabular form. This database has been carefully planned and structured so that data can be analyzed efficiently through queries. Doing so comes at a cost of processing data to fit all the rules and structures.

Data storage is a place where large amounts of raw data can be kept without any munging or curating. Data storage allows us to keep data of different types or data we might want to parse in the future.

The benefit of having dedicated data storage is that nothing limits the intake of data. Data can flow in constantly and be saved without having to worry if it fits the criteria of the database. We have seen this with our extract, transform, and load (ETL) process—the data storage can hold raw files, such as CSV or JSON, for different needs.


#### AWS's Simple Storage Service

S3 is Amazon's cloud file storage service that uses key-value pairs. Files are stored on multiple servers and have a high rate of availability of more than 99.9%. To store files, S3 uses buckets, which are similar to folders or directories on your computer. Buckets can contain additional folders and files. Each bucket must have a unique name across all of AWS.

One of S3's perks is its fine-grained control over files. Each file or bucket can have different read and write permissions, which helps regulate what can be done with each file.

S3 is also very scalable—you are not limited to the memory of one computer. As data flows in, more and more can be stored, as opposed to a local computer that is limited by available memory. Additionally, it offers availability—several team members can access massive amounts of data from one central location.


#### PySpark and S3 Stored Data

Since PySpark is a big data tool, it has many ways of reading in files from data storage so that we can manipulate them. We have decided to use S3 as our data storage, so we'll use PySpark for all data processing.

Using PySpark is how we've been reading in our data into Google Colab so far. The format for reading in from S3 is the S3 link, followed by your bucket name, folder by each folder, and then the filename, as follows:

For US East (default region)

template_url = "https://<bucket-name>.s3.amazonaws.com/<folder-name>/<file-name>"

example_url = "https://dataviz-curriculum.s3.amazonaws.com/data-folder/data.csv"
For other regions

template_url = "https://<bucket-name.s3-<region>.amazonaws.com/<folder-name>/<file-name>"

example_url =" https://dataviz-curriculum.s3-us-west-1.amazonaws.com/data-folder/data.csv"


#### PySpark ETL (Extract, Transform and Load)

Let's run through a mock scenario using two different types of raw data stored in S3. Our goal is to get this raw data from S3 into an RDS database.

Assume your company already has three tables set up in the RDS database and would like to get the raw data from S3 into the database. Create a new database in pgAdmin called "my_data_class_db." We'll have it represent the company database by first running the following schema in pgAdmin for our RDS:

````sql
-- Create Active User Table
CREATE TABLE active_user (
 id INT PRIMARY KEY NOT NULL,
 first_name TEXT,
 last_name TEXT,
 username TEXT
);

CREATE TABLE billing_info (
 billing_id INT PRIMARY KEY NOT NULL,
 street_address TEXT,
 state TEXT,
 username TEXT
);

CREATE TABLE payment_info (
 billing_id INT PRIMARY KEY NOT NULL,
 cc_encrypted TEXT
);
````

**NOTE**
Table creation is not part of the ETL process. We're creating the tables to represent a pre-established database you need for the raw data. In a real-life situation, databases will already have a well-defined schema and tables for you, as the engineer, to process data into.

Start with creating a new notebook, installing Spark:

````python
import os
# Find the latest version of spark 3.0  from http://www-us.apache.org/dist/spark/ and enter as the spark version
# For example:
# spark_version = 'spark-3.0.2'
spark_version = 'spark-3.<enter version>'
os.environ['SPARK_VERSION']=spark_version

# Install Spark and Java
!apt-get update
!apt-get install openjdk-11-jdk-headless -qq > /dev/null
!wget -q http://www-us.apache.org/dist/spark/$SPARK_VERSION/$SPARK_VERSION-bin-hadoop2.7.tgz
!tar xf $SPARK_VERSION-bin-hadoop2.7.tgz
!pip install -q findspark

# Set Environment Variables
import os
os.environ["JAVA_HOME"] = "/usr/lib/jvm/java-11-openjdk-amd64"
os.environ["SPARK_HOME"] = f"/content/{spark_version}-bin-hadoop2.7"

# Start a SparkSession
import findspark
findspark.init()
````

We'll use Spark to write directly to our Postgres database. But in order to do so, there are few more lines of code we need.

First, enter the following code to download a Postgres driver that will allow Spark to interact with Postgres:

````python
!wget https://jdbc.postgresql.org/download/postgresql-42.2.16.jar
````
You should get a message containing the words "HTTP request sent, awaiting response… 200 OK," indicating that your request was processed without a problem.
Then, start a Spark session with an additional option that adds the driver to Spark:

````python
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("CloudETL").config("spark.driver.extraClassPath","/content/postgresql-42.2.16.jar").getOrCreate()
````

We have performed the first two steps of the ETL process before with PySpark, so let's quickly review those.


**Extract**
We can connect to data storage, then extract that data into a DataFrame. We'll do this on two datasets, and be sure to replace the bucket name with one of your own.

We'll start by importing SparkFiles from PySpark into our notebook. This will allow Spark to add a file to our Spark project.

Next, the file is read in with the read method and combined with the csv() method, which pulls in our CSV stored in SparkFiles and infers the schema. SparkFiles.get() will have Spark retrieve the specified file, since we are dealing with a CSV.  The "," is the chosen separator, and we will have Spark determine the head for us. Enter the following code:

````python
# Read in data from S3 Buckets
from pyspark import SparkFiles
url ="https://YOUR-BUCKET-NAME.s3.amazonaws.com/user_data.csv"
spark.sparkContext.addFile(url)
user_data_df = spark.read.csv(SparkFiles.get("user_data.csv"), sep=",", header=True, inferSchema=True)
````

Finally, an action is called to show the first 10 runs and confirm our data extraction by entering the following code:

````python
# Show DataFrame
user_data_df.show()
````

Repeat a similar process to load in the other data. Enter the code:

````python
url ="https://YOUR-BUCKET-NAME.s3.amazonaws.com/user_payment.csv"
spark.sparkContext.addFile(url)
user_payment_df = spark.read.csv(SparkFiles.get("user_payment.csv"), sep=",", header=True, inferSchema=True)

# Show DataFrame
user_payment_df.show()
````

**Transform**
Now that the raw data stored in S3 is available in a PySpark DataFrame, we can perform our transformations.

First, join the two tables:

![d1](https://github.com/emmanuelmartinezs/Amazon_Vine_Analysis/blob/main/Resources/Images/data-16-9-1-1-Join-Two-DataFrames.png)


Next, drop any rows with null or "not a number" (NaN) values:

![d1](https://github.com/emmanuelmartinezs/Amazon_Vine_Analysis/blob/main/Resources/Images/data-16-9-1-2-Drop-Null-Values.png)

Filter for active users:

![d1](https://github.com/emmanuelmartinezs/Amazon_Vine_Analysis/blob/main/Resources/Images/data-16-9-1-3-Load-Sql-Function.png)

Next, select columns to create three different DataFrames that match what is in the AWS RDS database. Create a DataFrame to match the active_user table:

![d1](https://github.com/emmanuelmartinezs/Amazon_Vine_Analysis/blob/main/Resources/Images/data-16-9-1-4-Create-User-Dataframe-Active-User-Table.png)

Next, create a DataFrame to match the billing_info table:

![d1](https://github.com/emmanuelmartinezs/Amazon_Vine_Analysis/blob/main/Resources/Images/data-16-9-1-5-Create-User-DataFrame-Match-Billing-Info-Table.png)

Finally, create a DataFrame to match the payment_info table:

![d1](https://github.com/emmanuelmartinezs/Amazon_Vine_Analysis/blob/main/Resources/Images/data-16-9-1-6-Create-User-DataFrame-Match-Payment.png)

Once our data has been transformed to fit the tables in our database, we're ready to move on to the "Load" step.

**Load**
The final step is to get our transformed raw data into our database. PySpark can easily connect to a database to load the DataFrames into the table. First, we'll do some configuration to allow the connection with the following code:

````python
# Configure settings for RDS
mode = "append"
jdbc_url="jdbc:postgresql://<connection string>:5432/<database-name>"
config = {"user":"postgres",
          "password": "<password>",
          "driver":"org.postgresql.Driver"}
````

You'll need to provide your username and password, and also supply the AWS server name where `<connection string>` is located in the code above. To find it in PgAdmin, right-click AWS in the Server directory listing on the left side of PgAmin, and then select Properties in the drop-down menu. Select the Connection tab in the window that opens, and then select the address in the Host name/address field. Copy that address and paste it in place of `<connection string>`.


Let's further break down what's happening here:

* `mode` is what we want to do with the DataFrame to the table, such as `overwrite` or `append`. We'll append to the current table because every time we run this ETL process, we'll want more data added to our database without removing any.
* The `jdbc_url` is the connection string to our database.
- Replace `<connection string>` with the endpoint connection url found from your AWS RDS console.
- Replace `<database name>` with the name of your database you wish to connect to.
* A dictionary of configuration that includes the `user`, `password`, and `driver` to what type of database is being used.
- The `user` field is the username for your database, which should be `postgres` if you followed with the creation of the RDS instance. Otherwise, enter the one you created.
- The `password` would be the password you created when making the RDS instance.

**NOTE**
> If you forget anything like the name of the database or user name you can check on pgAdmin for these values. Be sure that you are entering the name of the database and not the name of your server in the connection string.

The cleaned DataFrames can then be written directly to our database by using the `.write.jdbc` method that takes in the parameters we set:

The connection string stored in `jdbc_url` is passed to the URL argument.
The corresponding name of the table we are writing the DataFrame to.
The mode we're using, which is "append."
The connection configuration we set up passed to the properties.
The code is as follows:

````python
# Write DataFrame to active_user table in RDS
clean_user_df.write.jdbc(url=jdbc_url, table='active_user', mode=mode, properties=config)

# Write dataframe to billing_info table in RDS
clean_billing_df.write.jdbc(url=jdbc_url, table='billing_info', mode=mode, properties=config)

# Write dataframe to payment_info table in RDS
clean_payment_df.write.jdbc(url=jdbc_url, table='payment_info', mode=mode, properties=config)

````
Let's wrap up by double-checking our work and running queries in pgAdmin on our database to confirm that the load did exactly what we wanted:

````sql
-- Query database to check successful upload
SELECT * FROM active_user;
SELECT * FROM billing_info;
SELECT * FROM payment_info;
````

Nice work! You now have enough knowledge and practice with PySpark and AWS to begin your client project.


> Let's move on!

# Deliverable 1:  
## Linear Regression to Predict MPG
### Deliverable Requirements:

The `MechaCar_mpg.csv` dataset contains mpg test results for 50 prototype MechaCars. The MechaCar prototypes were produced using multiple design specifications to identify ideal vehicle performance. Multiple metrics, such as vehicle length, vehicle weight, spoiler angle, drivetrain, and ground clearance, were collected for each vehicle. Using your knowledge of R, you’ll design a linear model that predicts the mpg of MechaCar prototypes using several variables from the `MechaCar_mpg.csv file`. 

> To Deliver. 

- The `MechaCar_mpg.csv` file is imported and read into a dataframe
- An RScript is written for a linear regression model to be performed on all six variables
- An RScript is written to create the statistical summary of the linear regression model with the intended p-values
- There is a summary that addresses all three questions


#### Results on Deliverable:
**Resulting Model:** 

### mpg =  (6.267)**vehicle_length** + (0.0012)**vehicle_weight** + (0.0688)**spoiler_angle** + (3.546)**ground_clearance** + (-3.411)**AWD** + (-104.0)
				

**Statistical Summary:** 
![d1](https://github.com/emmanuelmartinezs/MechaCar_Statistical_Analysis/blob/main/Resources/Images/linear_regression_d1.png)

From the above output we can see that:

1. The **vehicle length**, and **vehicle ground clearance** are statistically likely to provide non-random amounts of variance to the model. That is to say, the vehicle length and vehicle ground clearance have a significant impact on miles per gallon on the MechaCar prototype. Conversely,
the **vehicle weight**, **spoiler angle**, and **All Wheel Drive** (AWD) have p-Values that indicate a random amount of variance with the dataset.  

2. The p-Value for this model, ```p-Value: 5.35e-11```, is much smaller than the assumed significance level of 0.05%. This indicates there is sufficient evidence to **reject our null hypothesis**, which further indcates that the slope of this linear model is **not zero**.


3.  This linear model has an r-squared value of 0.7149, which means that approximately 71% of all mpg predictions will be determined by this model. Relatively speaking, his multiple regression model **does predict mpg of MechaCar prototypes effectively**. 

If we remove the less impactful independent variables (vehicle weight, spoiler angle, and All Wheel Drive), the predictability does decrease, but not drastically: the r-squared value falls from 0.7149 to 0.674. 

![d1](https://github.com/emmanuelmartinezs/MechaCar_Statistical_Analysis/blob/main/Resources/Images/new_linear_regression_d1.png)


# Deliverable 2:  
## Summary Statistics on Suspension Coils
### Deliverable Requirements:

The MechaCar Suspension_Coil.csv dataset contains the results from multiple production lots. In this dataset, the weight capacities of multiple suspension coils were tested to determine if the manufacturing process is consistent across production lots. Using your knowledge of R, you’ll create a summary statistics table to show:

- The suspension coil’s PSI continuous variable across all manufacturing lots
- The following PSI metrics for each lot: mean, median, variance, and standard deviation.

#### Technical Analysis
1. Download the `Suspension_Coil.csv` file, and place it in the active directory for your R session.
2. In your `MechaCarChallenge.RScript`, import and read in the `Suspension_Coil.csv` file as a table.
3. Write an RScript that creates a `total_summary` dataframe using the `summarize()` function to get the mean, median, variance, and standard deviation of the suspension coil’s PSI column.

Your `total_summary` dataframe should look like this:

![d1](https://github.com/emmanuelmartinezs/MechaCar_Statistical_Analysis/blob/main/Resources/Images/data-15-total-summary-data-mean-median-variance-sd.png)

4. Write an RScript that creates a `lot_summary` dataframe using the `group_by()` and the `summarize()` functions to group each manufacturing lot by the mean, median, variance, and standard deviation of the suspension coil’s PSI column.
Your lot_summary dataframe should look like this:

![d1](https://github.com/emmanuelmartinezs/MechaCar_Statistical_Analysis/blob/main/Resources/Images/data-15-manufacturing-lot.png)

5. Save your `MechaCarChallenge.RScript` file to your GitHub repository.

> To Deliver. 

You will earn a perfect score for Deliverable 2 by completing all requirements below:

- The Suspension_Coil.csv file is imported and read into a dataframe
- An RScript is written to create a total summary dataframe that has the mean, median, variance, and standard deviation of the PSI for all manufacturing lots
- An RScript is written to create a lot summary dataframe that has the mean, median, variance, and standard deviation for each manufacturing lot
- There is a summary that addresses the design specification requirement for all the manufacturing lots and each lot individually

The Suspension Coil dataset provided for the MechaCar contains the results of testing the weight capacities of multiple suspension coils from multiple production lots to determine consistency. 

First looking at all manufacturing lots:

![d2](https://github.com/emmanuelmartinezs/MechaCar_Statistical_Analysis/blob/main/Resources/Images/total_lot_summary.png)

Diving a little deeper into each of the 3 lots:

![d2](https://github.com/emmanuelmartinezs/MechaCar_Statistical_Analysis/blob/main/Resources/Images/manufactoring_lot_summary.png)

With the understanding that the design specifications for the MechaCar suspension coils mandate that <mark style="background-color: Yellow">**the variance of the suspension coils cannot exceed 100 pounds per square inch (PSI)**</mark> . 

Does the current manufacturing data meet this design specification for all manufacturing lots in total and each lot individually? Why or why not?

When looking at the entire population of the production lot, the variance of the coils is 62.29 PSI, which is well within the 100 PSI variance requirement.  

Similarly, but significantly more consistent, Lot 1 and Lot 2 are well within the 100 PSI variance requirement; with variances of 0.98 and 7.47 respectively.  However, it is Lot 3 that is showing much larger variance in performance and consistency, with a variance of 170.29.  It is Lot 3 that is disproportionately causing the variance at the full lot level.  

This very simple boxplot illustrates the differences between the lots:

![d2](https://github.com/emmanuelmartinezs/MechaCar_Statistical_Analysis/blob/main/Resources/Images/boxplot2.png)

# Deliverable 3:  
## t-Tests on Suspension Coils
### Deliverable Requirements:

Using your knowledge of R, perform t-tests to determine if all manufacturing lots and each lot individually are statistically different from the population mean of 1,500 pounds per square inch.

#### Technical Analysis
1. In your `MechaCarChallenge.RScript`, write an RScript using the `t.test()` function to determine if the PSI across all manufacturing lots is statistically different from the population mean of 1,500 pounds per square inch.
2. Next, write three more RScripts in your `MechaCarChallenge.RScript` using the `t.test()` function and its `subset()` argument to determine if the PSI for each manufacturing lot is statistically different from the population mean of 1,500 pounds per square inch.

- An RScript is written for t-test that compares all manufacturing lots against mean PSI of the population
- An RScript is written for three t-tests that compare each manufacturing lot against mean PSI of the population
- There is a summary of the t-test results across all manufacturing lots and for each lot

The next step is to conduct a t-test on the suspension coil data to determine whether there is a statistical difference between the mean of this provided sample dataset and a hypothesized, potential population dataset. Using the presumed **population mean of 1500**, we find the following:

There is a summary of the t-test results across **all manufacturing lots**
![d3](https://github.com/emmanuelmartinezs/MechaCar_Statistical_Analysis/blob/main/Resources/Images/t_test_all.png)

From here we can see the **true mean of the sample is 1498.78**, which we also saw in the summary statistics above.  With a **p-Value of 0.06**, which is higher than the common significance level of 0.05, there is **NOT enough evidence to support rejecting the null hypothesis**.  That is to say, the mean of all three of these manufacturing lots is statistically similar to the presumed population mean of 1500. 

**Next looking at each individual lots:**

1. Lot 1 sample actually has the **true sample mean of 1500**, again as we saw in the summary statistics above. With a **p-Value of 1**, clearly we cannot reject (i.e. accept) the null hypothesis that there is no statistical difference between the observed sample mean and the presumed population mean (1500).
2. Lot 2 has essentially the same outcome with a **sample mean of 1500.02**, a **p-Value of 0.61**; the null hypothesis cannot be rejected, and the sample mean and the population mean of 1500 are statistically similar.
3. However, Lot 3, not surprisingly is a different scenario. Here **the sample mean is 1496.14** and the **p-Value is 0.04**, which is lower than the common significance level of 0.05.  All indicating to **reject the null hypothesis** that this sample mean and the presumed population mean are not statistically different.

 ![d3](https://github.com/emmanuelmartinezs/MechaCar_Statistical_Analysis/blob/main/Resources/Images/t_test_lot.png)

How does this information help?  Clearly, something went awry in Lot 3's production cycle. The process needs to be checked for system fails and the suspension coils from this lot need to be inspected to remove those not meeting quality criteria.

# Deliverable 4:  
## Study Design: MechaCar vs Competition
### Deliverable Requirements:

Using your knowledge of R, design a statistical study to compare performance of the MechaCar vehicles against performance of vehicles from other manufacturers.

The statistical study design has the following:
- A metric to be tested is mentioned
- A null hypothesis or an alternative hypothesis is described
- A statistical test is described to test the hypothesis


This study would involve collecting data on MechaCar and its comparable models across several different manufacturers over the last 3 years.

* What are the competitions' comparable models, 
* Which cars will MechaCar be competing with head-to-head? which cars will be included in the study?
* Which factors will look at the study to determine the relevant to selling price?
 

#### Metrics
Collecting data for comparable models across all major manufacturers for past 3 years for the following metrics:

*  Safety Feature Rating: **Independent Variable**
*  Current Price (Selling): **Dependent Variable**
*  Drive Package : **Independent Variable**
*  Engine (Electric, Hybrid, Gasoline / Conventional): **Independent Variable**
*  Resale Value: **Independent Variable**
*  Average Annual Cost of ownership (Maintenance): **Independent Variable**
*  MPG (Gasoline Efficiency): **Independent Variable**


#### Hypothesis: Null and Alternative
After determining which factors are key for the MechaCar's genre:

 * Null Hypothesis (Ho): MechaCar is priced correctly based on its performance of key factors for its genre.
 * Alternative Hypothesis (Ha): MechaCar is NOT priced correctly based on performance of key factors for its genre.
 
#### Statistical Tests
A **multiple linear regression** would be used to determine the factors that have the highest correlation/predictability with the list selling price (dependent variable); which combination has the greatest impact on price (it may be all of them!)



##### Furniture Products - Amazon Vine Analysis Completed by Emmanuel Martinez

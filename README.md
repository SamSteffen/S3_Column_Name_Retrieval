# AWS Athena: How To Retrieve Column Names From a CSV Dataset Stored in AWS S3 That's Too Large to Download or Open
### The Problem
There may be instances when you're working in AWS S3 and Athena when your data file, having been successfully uploaded to a folder within an S3 bucket, is too large to open using S3 Select and may even be too large to safely download and open in any text editor. What to do? How are you supposed to retrieve column names to create an accurate table in Athena if you can't see the data you're working with?

### A Solution
> Note: The following solution will only work for data whose file contains column headers.

### Step 1: Create the Table in AWS Athena Using Placeholder Column Names and Nullable String Datatypes 
A somewhat tedious (but nonetheless viable) workaround to this problem involves creating a table using the method described in [AWS Athena CREATE TABLE documentation](https://docs.aws.amazon.com/athena/latest/ug/create-table.html) but with the following modifications:

    1. Use placeholder names for the column headers.
    2. Call in all the data as 'string' datatypes.
    3. Call in all the columns as nullable.
    4. Take a blind guess at how many columns your data is likely to contain. 
    5. Omit from the ```TBLPROPERTIES``` definition the parameter ```'skip.header.line.count' = '1'```

With these modifications implemented, your initial ```CREATE TABLE``` script might resemble the following:

```
CREATE TABLE IF NOT EXISTS my_unviewable_data_table (
    column1 string comment 'NULL'
    , column2 string comment 'NULL'	
    , column3 string comment 'NULL'	
    , column4 string comment 'NULL'	
    , column5 string comment 'NULL'	
    , column6 string comment 'NULL'	
    , column7 string comment 'NULL'	
    , column8 string comment 'NULL'	
    , column9 string comment 'NULL'	
    , column10 string comment 'NULL'		
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES ('field.delim' = ',')
STORE AS TEXTFILE
LOCATION 's3://MyUniqueS3BucketLocation/MyData/'
TBLPROPERTIES ('classification' = 'csv'
-- , 'skip.header.line.count' = '1'                  -- this line should be deleted from this script
);
```

Note that the above script just uses 'columnX' for the name of the columns, and may contain more or fewer columns than our known dataset. That's okay! In fact, if it's your first time creating a table from a dataset you've never seen and you don't know how many columns there are, it's better to try creating and recreating the table until you can confirm that you've reached the 'end' of the dataset&mdash;that is, until your data starts showing columns that are without column headers or are otherwise entirely null. 

### Step 2: View and Order the Data in AWS Athena 
Once you've created the table, you can view the first ten rows of data by running the following:

```select * from my_unviewable_data_table limit 10```

My recommendation at this point would be to identify a column that appears to contain mostly *numeric* data. Because you've called in all the columns as strings (synonymous with varchars in Athena) you can then query the data by that column <ins>in reverse order</ins>. Because Athena processes numbers before letters when given string data, a query sorting a numeric column in descending order should reveal the column headers contained within the dataset (if there are any). 

Say, for example, ```column3``` in the ```my_unviewable_data_table``` contained what appeared to be only numbers. In this case, you could run the following query.

```SELECT * FROM my_unviewable_data_table ORDER BY column3 DESC limit 10```

### Step 3: Locate the Number of Columns in Your Dataset
If the ```CREATE TABLE``` script from Step 1 were to yield a dataset that appears to have data in all the columns (column1 through column10) you may want to drop the table using the following query:

```DROP TABLE IF EXISTS my_unviewable_data_table```

Once the table is dropped, add ten more columns to your ```CREATE TABLE``` script, and try creating the table again, like this:

```
CREATE TABLE IF NOT EXISTS my_unviewable_data_table (
    column1 string comment 'NULL'
    , column2 string comment 'NULL'	
    , column3 string comment 'NULL'	
    , column4 string comment 'NULL'	
    , column5 string comment 'NULL'	
    , column6 string comment 'NULL'	
    , column7 string comment 'NULL'	
    , column8 string comment 'NULL'	
    , column9 string comment 'NULL'	
    , column10 string comment 'NULL'		
    , column11 string comment 'NULL'
    , column12 string comment 'NULL'	
    , column13 string comment 'NULL'	
    , column14 string comment 'NULL'	
    , column15 string comment 'NULL'	
    , column16 string comment 'NULL'	
    , column17 string comment 'NULL'	
    , column18 string comment 'NULL'	
    , column19 string comment 'NULL'	
    , column20 string comment 'NULL'		
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES ('field.delim' = ',')
STORE AS TEXTFILE
LOCATION 's3://MyUniqueS3BucketLocation/MyData/'
TBLPROPERTIES ('classification' = 'csv'
-- , 'skip.header.line.count' = '1'                  -- this line should be deleted from this script
);
```
Following this, rerun your query to view the column headers:

```SELECT * FROM my_unviewable_data_table ORDER BY column3 DESC limit 10```

You should repeat this process of creating and dropping the table until you have arrived at the 'end' of your dataset, which should be indicated by the datarow that contains the file's column headers showing NULL values.

### Step 4: Modify Your CREATE TABLE Script to Include Appropriate Column Names, Number of Columns, and Datatypes

Once you can view the column headers and confirm that you've arrived at the end of your data (because you are seeing null or unnamed columns), you can modify your ```CREATE TABLE IF NOT EXISTS``` script using the appropriate column names, number of columns, and datatypes.

### Step 5: Remove Column Headers from the Data Being Queried

With your column names are updated, you can also add the ```'skip.header.line.count' = '1'``` parameter back to the ```TBLPROPERTIES``` definition. 

### Step 6: Recreate Your Table Using Your Updated Query
With your ```CREATE TABLE``` script fully updated to suit your dataset, you can then drop the 'placeholder' version of your table by running the following:

```DROP TABLE IF EXISTS my_unviewable_data_table```

As a final step, you can recreate your table using your updated script.
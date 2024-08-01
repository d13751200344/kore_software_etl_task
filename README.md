SSIS ETL Process for CSV to SQL Server


Overview
This project demonstrates how to use SSIS (SQL Server Integration Services) to migrate data from a CSV file to a production SQL Server table. The ETL (Extract, Transform, Load) process is divided into three main steps: data extraction and type standardization, data cleaning, and incremental load to the production table. The objective is to simulate a real-world scenario where data from third-party integrations may contain errors, and showcase techniques for handling such data.


Package Descriptions

There are four packages, each of them represents different attempt and different methods that were utilized to solve problems.
Package V4 contains a complete solution of the pipeline.

Package V1: Initial data conversion and separation using Conditional Split
Package V2: Processing data using Script Component and Conditional Split.
Package V3: Using SQL task to set IsNullFlag and attempting to separate data.
Package V4: Contains a complete solution of the pipeline. Importing data, processing data from stg.Users_noConversion, performing type conversion and cleaning, and inserting results into prod.Users.

I want to use other tables to record the data rows that contain unacceptable values like null or incompatible datatypes, so I created 2 tables: stg.Users_Problem1 and stg.Users_Problem2.
All rows containing null are stored in stg.Users_Problem1 or stg.Users_Problem2.

Package V1:
 I followed the task description: data conversion first and transformation later.
 Tried to use "Conditional Split" to separate Null data rows from the others, but all 33 rows all went to "Good values" condition. Failed.
 And there was an error occurs during data conversion because some values are not acceptable for the converted data type.


Package V2:
 I followed the task description: data conversion first and transformation later.
 Tried to separate the Null rows from the rest of the rows with Script Component and Conditional Split, but it failed as it did the first time, all 33 rows all went to "Good values" condition.
 And there was an error occurs during data conversion because some values are not acceptable for the converted data type.


**I have created another table called "stg.Users_noConversion" (All nvarchar) which will store the data that has been singled out from the null rows, but may contain incompatible data types because the data types for all fields in this table will be strings, and leave the data type transformation for later. If I don't do this, I will get an error when converting the data because some of the values will be inappropriate for the converted data type.**

All rows containing null are stored in stg.Users_Problem1;
stg.Users_Problem1 (
        StgID INT IDENTITY(1,1) PRIMARY KEY,
        UserID NVARCHAR(255),
        FullName NVARCHAR(255),
        Age NVARCHAR(255),
        Email NVARCHAR(255),
        RegistrationDate NVARCHAR(255),
        LastLoginDate NVARCHAR(255),
        PurchaseTotal NVARCHAR(255)
    );

stg.Users_noConversion (
        StgID INT IDENTITY(1,1) PRIMARY KEY,
        UserID  NVARCHAR(255),
        FullName NVARCHAR(255),
        Age NVARCHAR(255),
        Email NVARCHAR(255),
        RegistrationDate NVARCHAR(255),
        LastLoginDate NVARCHAR(255),
        PurchaseTotal NVARCHAR(255),
    );


Package V3:
I tried to use 'Execute SQL Task' to separate the Null row from the rest of the rows using Script Component and Conditional Split, and it failed just like before. (I created "stg.Users_rawData" table to process the data.)

steps: get raw data -> 
successfully store the data in stg.Users_rawData table (33 rows) -> 
Exeute SQL Task to set IsNullFlag value based on the values of each data row -> 
tried to use "Conditional Split" to separate Null data rows from the others, but failed.
** 33 rows has been processed in stg.Users_Problem1 table**
stg.Users_rawData (
        UserID  NVARCHAR(255),
        FullName NVARCHAR(255),
        Age NVARCHAR(255),
        Email NVARCHAR(255),
        RegistrationDate NVARCHAR(255),
        LastLoginDate NVARCHAR(255),
        PurchaseTotal NVARCHAR(255),
	IsNullFlag BIT
    );


** It seems the problem is on the source file data, but I couldn't find it.**


Package V4:
I utilized V2 Package, and then designed the pipeline from stg.Users_noCoversion (33 rows), which contains Unicode string data, to prod.Users ->
I used "Script Component" to create a column "isValid", using script to separate data rows that contain inappropriate data, and led them to different routes.(all data went to the wrong route, stored in stg.Users_Problem2, so failed) ->
If it's going to valid data types, then I will do Data Conversion to convert all columns data type, and then I will do data cleaning. ->
I used Script Component & Derived Column to possess Email and PurchaseTotal, and Derived column for checking Age, RegistrationDate and LastLoginDate. ->
Sorted data, and used Aggregate tools to do data cleaning. Group by UserID, Age, FullName, and Email, keeping earliest RegistrationDate and latest LastLoginDate, and summed up the PurchaseTotal. ->
Stored data into stg.Users_Transformation table ->
Update or Insert the data of stg.Users_Transformation table into prod.Users table with the "Execute SQL Task" tool to Load data. 
** 33 rows has been processed in stg.Users_Problem2 table **
The query is:
`MERGE INTO prod.Users AS target
USING stg.Users AS source
ON target.UserID = source.UserID
WHEN MATCHED THEN
    UPDATE SET 
        target.FullName = source.FullName,
        target.Age = source.Age,
        target.Email = source.Email,
        target.RegistrationDate = source.RegistrationDate,
        target.LastLoginDate = source.LastLoginDate,
        target.PurchaseTotal = source.PurchaseTotal
WHEN NOT MATCHED BY TARGET THEN
    INSERT (UserID, FullName, Age, Email, RegistrationDate, LastLoginDate, PurchaseTotal)
    VALUES (source.UserID, source.FullName, source.Age, source.Email, source.RegistrationDate, source.LastLoginDate, source.PurchaseTotal);`


stg.Users_Transformation (
        ID INT IDENTITY(1,1) PRIMARY KEY,
        UserID INT,
        FullName NVARCHAR(255),
        Age INT,
        Email NVARCHAR(255),
        RegistrationDate DATE,
        LastLoginDate DATE,
        PurchaseTotal FLOAT,
    );
stg.Users_Problem2 (
        StgID INT IDENTITY(1,1) PRIMARY KEY,
        UserID NVARCHAR(255),
        FullName NVARCHAR(255),
        Age NVARCHAR(255),
        Email NVARCHAR(255),
        RegistrationDate NVARCHAR(255),
        LastLoginDate NVARCHAR(255),
        PurchaseTotal NVARCHAR(255)
    );








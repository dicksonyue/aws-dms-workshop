---
# AWS DMS migration runbook with RDS, Redshift, S3 as target
    AWS Database Migration Service

    AWS Schema Conversion Tool

    Oracle to PostgreSQL, Redshift, S3, Athena -- Lab Guide
---

![mslogo](.//media/image1.png)

Dickson Yue, Solutions Architect

dyue@amazon.com

RUN THIS WORKSHOP IN AP-SOUTHEAST-1 (SINGAPORE)

# Objective

In this lab, you will be performing a migration from Oracle to
PostgreSQL using SCT and DMS

High Level Steps

-   Create a AWS CloudFormation stack

    -   Source: RDS Oracle - snapshot

    -   Targets: RDS PostgreSQL, Redshift, S3

    -   IAM Role for DMS S3 access

-   Create AWS Database Migration Instances

-   Connect to your environment

-   Setup AWS Schema Conversion Tool

-   Convert the Oracle schema to PostgreSQL

-   Create Source Endpoint in AWS DMS

-   Create Target Endpoint in AWS DMS -- RDS PostgreSQL

-   Create Migration Task in AWS DMS -- RDS PostgreSQL target

-   Start the migration

-   Generate transactions on Oracle and see the data being migrated to
    PostgreSQL -- CDC

![](.//media/dms.png)

Optional

-   Create 2 Target Endpoints in AWS DMS -- Redshift and S3

-   Create 2 Migration Tasks in AWS DMS -- Redshift and S3 target

-   Create database and table in Athena data catalog

-   Query data in Redshift and Athena

# Prerequisites

### Install a SQL Client

-   Install a SQL client of your choice; in this lab -- we will be using
    SQLWorkbenchJ screenshots

    -   SQL WorkbenchJ: <http://www.sql-workbench.net/downloads.html>

    -   DBeaver: <http://dbeaver.jkiss.org/>

    -   SQuirrel: <http://squirrel-sql.sourceforge.net/>

### Install SCT

-   Download the latest version of AWS Schema Conversion Tool (SCT) from
    this link
    <https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Installing.html>

### Download JDBC Drivers

-   Download & keep this file locally

-   You will need these to connect to source & target databases using
    SQL client & AWS Schema Conversation Tool
    
    - <https://s3-ap-southeast-1.amazonaws.com/aws-apac-dms-workshop/content/labs/jdbc-drivers/ojdbc7.jar> 
    - <https://s3-ap-southeast-1.amazonaws.com/aws-apac-dms-workshop/content/labs/jdbc-drivers/postgresql-42.1.1.jar> 
    -  <https://s3.amazonaws.com/redshift-downloads/drivers/RedshiftJDBC42-1.2.7.1003.jar> (if you will query data in redshift, you will need this driver )                                                   


### Or launch an EC2 Windows Instance in the Singapore Region


Please check on page 7 for the procedure

Create AWS Cloudformation Stack

In this step, you will launch a AWS Cloudformation template they will
setup the following resources needed for this lab.

-   Source Database: Amazon RDS Oracle (this database will be
    pre-populated with sample database installed from
    <https://github.com/awslabs/aws-database-migration-samples>)

-   Target Database: Amazon RDS PostgreSQL

# Instructions

-   Open the link provided below
<https://ap-southeast-1.console.aws.amazon.com/cloudformation/home?region=ap-southeast-1#/stacks/new?stackName=dmsworkshop&templateURL=https://s3-ap-southeast-1.amazonaws.com/aws-share-dyue/dmsworkshop/dmsworkshop.template>
  

-   Click Next

    ![](.//media/image3.png)
-   Enter the following parameters in the 'Specify Details' page --
    *[this should be populated by default]{.underline}*

    -   Stack name: dms-lab

    -   Source Oracle Database Configuration

    

  |OracleDBName         |ORCL
  |-------------------- |--------------
  |OracleDBPassword     |oraadmin123
  |OracleDBStorage      |100
  |OracleInstanceType   |db.t2.medium

-   Target RDS PostgreSQL Database Configuration

  |RDSDBName         |postgres
  |----------------- |--------------
  |RDSDBUsername     |postadmin
  |RDSDBPassword     |postadmin123
  |RDSInstanceType   |db.t2.medium
  |RDSDBStorage      |100

-   Target Redshift Database Configuration

  |RsDBName         |dw
  |---------------- |------------
  |RsDBUsername     |rsadmin
  |RsDBPassword     |rsAdmin123
  |RsInstanceType   |dc1.large

-   Click Next

    ![](.//media/image4.png)


-   Tags

  |Key     |purpose
  |------- |---------
  |Value   |dms-lab

-   Click '**Next**'

![](.//media/image5.png)

Checked "I acknowledge that AWS CloudFormation might create IAM
resources."

![](.//media/image6.png)

-   Review and Click Create

![](.//media/image7.png)

-Optional-

Using an EC2 instance for Workshop tools

## Launch a Windows EC2 instance from a pre-installed AMI


-   Connect on the AWS Console and go to Singapore Region

![](.//media/image8.png)

-   Go to the EC2 Services

![](.//media/image9.png)

![](.//media/image10.png)

-   Create and Launch an Instance

![](.//media/image11.png)

-   Go to "**Community AMIs**" to select the AMI with all tools
    pre-installed

    ![](.//media/image12.png)

-   **Search and Select** the AMI "DMS\_Workshop\_Win\_Tools"
    ![](.//media/image13.png)

-   **Select an Instance**: t2.micro would be good for this workshop

-   **Next**: Configure Instance Details

    ![](.//media/image14.png)

-   **Number of instances: 1**

-   **Network**: Select the VPC created by the CloudFormation template
    (ie. vpc-XXXXXX \| dmsworkshop)

-   **Auto-assign Public IP**: Enable

-   **No change to other options**

    ![](.//media/image15.png)

-   **Next**: Add Storage

-   **Next**: Add Tags

-   **Next**: Configure Security Group

-   **Allow RDP** (port: 3389)**, from source**: "My IP"

    ![](.//media/image16.png)

-   **Review and Launch**

-   **Launch** if no change required.

-   ![](.//media/image17.png)Proceed without a key pair and Launch
    Instances

    ![](.//media/image18.png)

-   The instance is being provisioned and can take few minutes

-   Go to "**View Instances**"

-   Once the instance is in running state with Status Checks 2/2

-   Retrieve the **public IP address**

![](.//media/image19.png)

-   Connect using Microsoft Remote Desktop

-   **Hostname**: IP or DNS name retrieved from the console

-   **Username**: Administrator

-   **Password**: Qo;u@-4Tea!b.\*wvnvughW-ll!8sy\*vl


## AWS Schema Conversion Tool

### Install AWS Schema Conversion Tool (on your laptop)


-   Download the latest version of AWS Schema Conversion Tool (SCT) from
    this link
    <https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Installing.html>

-   If you already have SCT installed, download the latest version &
    install it

-   JDBC Drivers

    -   For connecting to our source (Oracle) vs target (PostgreSQL) you
        will need the respective JDBC drivers

        -   Oracle JDBC driver:
            [[http://bit.ly/2phVpPk]{.underline}](http://bit.ly/2phVpPk)
            -\>

        -   PostgreSQL JDBC driver:
            [[http://bit.ly/2pt04ZT]{.underline}](http://bit.ly/2pt04ZT)
            -\>

-   Configure SCT with drivers

    -   Click on **Settings** \> **Global Settings**

![](.//media/image20.png)

-   In the Global Settings window,

    -   Goto **Drivers** on left panel

    -   Oracle Driver Path: Select the downloaded ojdbc jar file

    -   PostgreSQL Driver Path: Select the downloaded postgresql jar
        file

    -   Click **OK** to Proceed

![](.//media/image21.png)

Open Security Groups from Source / Target Database Instances

For you to access Source and Target databases, you will have to add your
laptop to Oracle & Postgres Security Groups

  
https://ap-southeast-1.console.aws.amazon.com/ec2/v2/home?region=ap-southeast-1\#SecurityGroups:sort=groupId
  

Add following inbound rule to respective security groups

-   Oracle Port -- 1521 \> Open to 'My IP'

-   Postgres Port -- 5432 \> Open to 'My IP'

-   Redshift Port -- 5439 \> Open to 'My IP'

Create a new SCT project

-   In AWS SCT, select File \> New Project Wizard

-   Step 1 -- Select Source

  |Project Name             |DMS-Workshop-Oracle2PostgreSQL
  |------------------------ |--------------------------------
  |Location                 |Leave Default - Transactional Database (OLTP)
  |Source Database Engine   |Oracle
  |Target Database Engine   |Amazon RDS for PostgreSQL

-   Click OK to proceed

![](.//media/image22.png)

-   Step 2 -- Connect to Source Database

    -   In the top of icon, select "Connect to Oracle"

|Type 			|SID
|-------------|--------------------------------------|
| Server Name | DNS name of your Oracle RDS instance |
|             |                                      |
|             | (follow next screen)                 |
| Server Port | 1521                                 |
| Oracle SID  | ORCL                                 |
| User name   | dbmaster                             |
| Password    | oraadmin123                          |

RDS endpoints

  ---------------------------------------------------------------------------------------------
  <https://ap-southeast-1.console.aws.amazon.com/rds/home?region=ap-southeast-1#dbinstances>:
  ---------------------------------------------------------------------------------------------

![](.//media/image23.png)

-   Click '**Test Connection**' -- Make sure you get a '**Connection
    Successful**' message

-   Click '**Next'** to proceed

![](.//media/image24.png)

-   Step 3 -- Select Target

    -   On the top icon, select "Connect to Amazon RDS for PostgreSQL"

  |Server Name   |DNS name of your RDS PostgreSQL instance
  |------------- |------------------------------------------
  |Database      |postgres
  |User name     |postadmin
  |Password      |postadmin123

-   Click 'Test Connection' -- Make sure you get a 'Connection
    Successful' message

-   Click 'Finish' to proceed

![](.//media/image25.png)

## Run Schema Conversation In SCT

Review the project screen and familiarize yourself

-   Uncheck all schemas on the left except for the **DMS\_SAMPLE**
    schema.

-   Select **DMS\_SAMPLE**

-   Click '**Actions'** \> '**Create Report**'

-   Go to the '**Summary\'** tab on the top and review the generated
    report

![](.//media/image26.png)

-   Look through what Oracle objects could be automatically converted
    and what could not be. Now, right click and click "**Convert
    schema**". The schema will be converted and shown on the PostgreSQL
    instance (it has not been applied yet).

-   Take a few minutes to review the objects being converted.

-   Since the majority of the objects which could not be converted are
    secondary objects like functions or procedures, right click on the
    created schema on the **Right Panel** and click "**Apply to
    database**". This will apply all those converted objects in the
    PostgreSQL target.

-   The above steps will convert all your Oracle objects into PostgreSQL
    objects. Objects which could not be converted automatically must be
    taken care of manually after migration at a later time.

-   At this point, most of the objects from your source Oracle databased
    has been converted to PostgreSQL target

# Database migration Service

## Create Database Migration Instance

-   Navigate to:
    [https://ap-southeast-1.console.aws.amazon.com/dms/home?region=ap-southeast-1\#replication-instances](https://ap-northeast-1.console.aws.amazon.com/dms/home?region=ap-southeast-1#replication-instances):

-   Click on '**Create Replication Instance**'

-   Populate the following values on this page

  |Name                  |dms-workshop-instance
  |--------------------- |---------------------------
  |Description           |dms instance for workshop
  |Instance Class        |dms.t2.medium
  |VPC                   |\[cf-stack-name\]
  |Multi-AZ              |No
  |Publicly accessible   |Checked

-   Click '**Create Replication Instance**' to proceed

![](.//media/image27.png)

Wait for a couple of minutes for the migration instance to start and
change the status to '**available'**

![](.//media/image28.png)

## Create Source / Target Endpoints

### Create Source Endpoint

-   Navigate to:
    <https://ap-southeast-1.console.aws.amazon.com/dms/home?region=ap-southeast-1#endpoints>:

-   Click on '**Create Endpoint**'

-   Enter these Details

| Endpoint Type                     | Source                            |
|-----------------------------------|-----------------------------------|
| Endpoint identifier               | dms-workshop-oracle               |
| Source engine:                    | oracle                            |
| Server name                       | \<oralce-rds-dns-endpoint\>       |
|                                   |                                   |
|                                   | get this from here                |
|                                   |                                   |
|                                   | https://ap-southeast-1.console.aw |
|                                   | s.amazon.com/rds/home?region=ap-s |
|                                   | outheast-1\#dbinstances           |
| Port                              | 1521                              |
| SSL Mode                          | none                              |
| User name                         | dbmaster                          |
| Password                          | oraadmin123                       |
| SID                               | ORCL                              |
| VPC                               | \[cf-stack-name\]                 |
| Replication instance              | dms-workshop-instance             |
| Refresh schemas after successful  | Checked                           |
| connection test                   |                                   |

![](.//media/image29.png)

-   Click \'Run Test\' -\> ensure you get the 'Connection tested
    successfully' message

-   Click on 'Save' to proceed

![](.//media/image30.png)

### Create Target Endpoint

-   Navigate to:
    https://ap-southeast-1.console.aws.amazon.com/dms/home?region=ap-southeast-1\#endpoints

-   Click on '**Create Endpoint**'

-   Enter these Details

| Endpoint Type                     | Target                            |
| Endpoint identifier               | dms-workshop-postgres             |
| Source engine:                    | postgres                          |
| Server name                       | \< postgres-rds-dns-endpoint\>    |
|                                   |                                   |
|                                   | get this from here                |
|                                   | https://ap-southeast-1.console.aw |
|                                   | s.amazon.com/rds/home?region=ap-s |
|                                   | outheast-1\#dbinstances           |
| Port                              | 5432                              |
| SSL Mode                          | none                              |
| User name                         | postadmin                         |
| Password                          | postadmin123                      |
| Database Name                     | postgres                          |
| VPC                               | \[cf-stack-name\]                 |
| Replication instance              | dms-workshop-instance             |
| Refresh schemas after successful  | Checked                           |
| connection test                   |                                   |

![](.//media/image31.png)

-   Click \'Run Test\' -\> ensure you get the 'Connection tested
    successfully' message

-   Click on 'Save' to proceed

![](.//media/image30.png)

-   Once all both source and target database endpoints have been created
    and successfully tested, you can proceed to the next step.

### Create DMS Migration Task

-   Navigate to
    https://ap-southeast-1.console.aws.amazon.com/dms/home?region=ap-southeast-1\#tasks:

-   Click on '**Create Task**'

Enter these Details

-   Basic Info

Make sure your configuration looks like the image below

  |Replication instance   |dms-workshop-instance
  |---------------------- |-----------------------------------------------------
  |Source endpoint        |dms-workshop-oracle
  |Target endpoint        |dms-workshop-postgres
  |Migration type         |Migrate existing data and replicate ongoing changes
  |Start task on create   |Checked

![](.//media/image32.png)

-   Task Settings

  |Target table preparation mode        |Do nothing
  |------------------------------------ |--------------------------------
  |CDC stop mode                        |Don't use custom CDC stop mode
  |Include LOB columns in replication   |Limited LOB Mode
  |Max LOB size (kb)                    |32KB
  |Enable logging                       |Checked

Make sure your configuration looks like the image below

![](.//media/image33.png)

-   Table Mappings

  |Schema Name is       |DMS\_SAMPLE
  |-------------------- |-------------
  |Table name is like   |\%
  |Action               |Include

-   Click \'**Add Selection Rule**\'

![](.//media/image34.png)

-   Under \'Transformation rules\' section click on \'add transformation
    rule\' (we will be creating 3 rules here)

![](.//media/image35.png)

-   Rule 1:

    -   Target: \'Schema\'

    -   Schema name is: \'DMS\_SAMPLE\'

    -   Action: \'make lower case\'

  Target           Schema
  ---------------- ----------------
  Schema Name is   DMS\_SAMPLE
  Action           Make lowercase

-   Click \'**Add transformation rule**\'.

![](.//media/image36.png)

-   Rule 2:

  |Target               |Table
  |-------------------- |----------------
  |Schema Name is       |DMS\_SAMPLE
  |Table Name is like   |\%
  |Action               |Make lowercase

-   Click \'Add transformation rule\'.

![](.//media/image37.png)

-   Rule 3:

  |Target                |Column
  |--------------------- |----------------
  |Schema Name is        |DMS\_SAMPLE
  |Table Name is like    |\%
  |Column name is like   |\%
  |Action                |Make lowercase

-   Click \'Add transformation rule\'.

![](.//media/image38.png)
-   Make sure your configuration looks like the image below.

-   Take few minutes to review the JSON text generated

![](.//media/image39.png)

-   Click on \'Create Task\'

Wait for the task to get created and start running.

At this stage, the database migration task should load 100% of data from
Oracle to Postgres. (This will usually take few 10s of minutes)

-   Monitoring the progress for your database migration task

    -   Select your newly create database migration task

    -   Click on 'Task monitoring' tab & review the cloud watch metrics
        for your task

    -   Click on 'Table statistics' tab & review table level stats for
        your migration

![](.//media/image40.png)

[Executing transactions on the source to test CDC ]{.underline}

Once your task's initial load is completed. You might want to execute a
few transactions on the source. So, connect to your source database as
dbmaster using your favorite tool: SQLDeveloper, DBeaver or even
SQL\*Plus!

-   Execute the following to sell some tickets:

    -   This is a stored procedure in Oracle, it will take \~ 3mins to
        perform 1000 transactions

  ----------------------------------------------------------
  exec ticketManagement.generateTicketActivity(0.01,1000);
  ----------------------------------------------------------

Once the transactions are committed on source, you should see them on
the target.

Check the status on your console \> Task \> Table Statistics

![](.//media/image41.png)
Once you've "sold" some tickets, you can execute the following to
"transfer" some tickets

  -----------------------------------------------------------
  exec ticketManagement.generateTransferActivity(0.1,1000);
  -----------------------------------------------------------

[Optional: Create DMS endpoints and task for Redshift]{.underline}

### Create Target Endpoint 

-   Navigate to:
    https://ap-southeast-1.console.aws.amazon.com/dms/home?region=ap-southeast-1\#endpoints

-   Click on '**Create Endpoint**'

-   Enter these Details

  |Endpoint Type         |Target
  |---------------------|--------------------  
  |Endpoint identifier   |dms-workshop-redshift
  |Target engine:        |redshift
  |Server name\*         |get this from here <https://ap-southeast-1.console.aws.amazon.com/redshift/home?region=ap-southeast-1#cluster-list>:
  |Port                  |5439
  |SSL mode              |none
  |User name             |rsadmin
  |Password              |rsAdmin123
  |Database name         |dw
                        

-   Click \'Run Test\' -\> ensure you get the 'Connection tested
    successfully' message

-   Click on 'Save' to proceed

-   Once all both source and target database endpoints have been created
    and successfully tested, you can proceed to the next step.

### Create DMS Migration Task

-   Navigate to
    https://ap-southeast-1.console.aws.amazon.com/dms/home?region=ap-southeast-1\#tasks:

-   Click on '**Create Task**'

Enter these Details

-   Basic Info

Make sure your configuration looks like the image below

  |Task name              |dms-workshop-redshift
  |---------------------- |-----------------------------------------------------
  |Replication instance   |dms-workshop-instance
  |Source endpoint        |dms-workshop-oracle
  |Target endpoint        |dms-workshop-redshift
  |Migration type         |Migrate existing data and replicate ongoing changes
  |Start task on create   |Checked

-   Task Settings

  |Target table preparation mode        |Do nothing
  |------------------------------------ |--------------------------------
  |CDC stop mode                        |Don't use custom CDC stop mode
  |Include LOB columns in replication   |Limited LOB Mode
  |Max LOB size (kb)                    |32KB
  |Enable logging                       |Checked

Let sync only one table for simplicity

-   Table Mappings

  |Schema Name is       |DMS\_SAMPLE
  |-------------------- |-------------------------
  |Table name is like   |SPORTING\_EVENT\_TICKET
  |Action               |Include

-   Click on \'Create Task\'

Wait for the task to get created and start running.

At this stage, the database migration task should load 100% of data from
Oracle to Postgres. (This will usually take few 10s of minutes)

-   Monitoring the progress for your database migration task

    -   Select your newly create database migration task

    -   Click on 'Task monitoring' tab & review the cloud watch metrics
        for your task

    -   Click on 'Table statistics' tab & review table level stats for
        your migration

# Query data in Redshift

## Configure your SQL bench with new redshift

![](.//media/image42.png)

|-----------------------------------|-----------------------------------|
| Driver                            | Redshift                          |
|                                   | (com.amazon.redshift.jdbc.Driver) |
|                                   |                                   |
|                                   | If you don\'t have the driver,    |
|                                   | add one and download from here    |
|                                   |                                   |
|                                   | <https://s3.amazonaws.com/redshif |
|                                   | t-downloads/drivers/RedshiftJDBC4 |
|                                   | 2-1.2.7.1003.jar>                 |
| Url                               | jdbc:redshift://{your redshift    |
|                                   | endpoint}.ap-southeast-1.redshift |
|                                   | .amazonaws.com:5439/dw            |
|                                   |                                   |
|                                   | https://ap-southeast-1.console.aw |
|                                   | s.amazon.com/redshift/home?region |
|                                   | =ap-southeast-1\#cluster-list:    |
| User name                         | Rsadmin                           |
| Password                          | rsAdmin123                        |

Run the script below to verify

  ------------------------------------------------------------
  select count(\*) from dms\_sample.sporting\_event\_ticket;
  ------------------------------------------------------------

![](.//media/image43.png)

### Remark

Do you aware of we didn\'t do schema creation for redshift. DMS will
create the schema if it doesn\'t exists. It is not prefer in most of the
case as the data type might not be optimal. In addition, the
distribution key and sort key design are very critical to the Redshift
query performance. The suggestion is

-   Run schema conversion tool to extract the schema as reference .

-   Carefully design distribution key, work key, compression , follow
    the best practices
    <http://docs.aws.amazon.com/redshift/latest/dg/c_designing-tables-best-practices.html>

[Optional: Create DMS endpoints and task for S3 ]{.underline}

### Create Target Endpoint

-   Navigate to:
    https://ap-southeast-1.console.aws.amazon.com/dms/home?region=ap-southeast-1\#endpoints

-   Click on '**Create Endpoint**'

-   Enter these Details

| Endpoint Type                     | Target                            |
|-----------------------------------|-----------------------------------|
| Endpoint identifier               | dms-workshop-s3                   |
| Target engine:                    | S3                                |
| Service Access Role ARN\*         | arn:aws:iam::{accounted}:role/{ro |
|                                   | le-name}                          |
|                                   |                                   |
|                                   | get this from here                |
|                                   | <https://console.aws.amazon.com/i |
|                                   | am/home?region=ap-southeast-1#/ro |
|                                   | les>                              |
| Target bucket name\*              | {bucket-name}                     |
| Target bucket folder\*            | dms                               |
| VPC                               | \[cf-stack-name\]                 |
| Replication instance              | dms-workshop-instance             |
| Refresh schemas after successful  | Checked                           |
| connection test                   |                                   |

#### IAM DMS Role 

AWS console -\> IAM -\> Role

<https://console.aws.amazon.com/iam/home?region=ap-southeast-1#/roles>

Filter: {cfn-stack-name}

![](.//media/image44.png)

![](.//media/image45.png)

#### S3 

AWS console -\> S3

Filter: {cfn-stack-name}

<https://s3.console.aws.amazon.com/s3/home?region=ap-southeast-1>

![](.//media/image46.png)

### Create DMS Migration Task

-   Navigate to
    https://ap-southeast-1.console.aws.amazon.com/dms/home?region=ap-southeast-1\#tasks:

-   Click on '**Create Task**'

Enter these Details

-   Basic Info

Make sure your configuration looks like the image below

  |Task name              |dms-workshop-s3
  |---------------------- |-----------------------------------------------------
  |Replication instance   |dms-workshop-instance
  |Source endpoint        |dms-workshop-oracle
  |Target endpoint        |dms-workshop-s3
  |Migration type         |Migrate existing data and replicate ongoing changes
  |Start task on create   |Checked

-   Task Settings

  |Target table preparation mode        |Do nothing
  |------------------------------------ |--------------------------------
  |CDC stop mode                        |Don't use custom CDC stop mode
  |Include LOB columns in replication   |Limited LOB Mode
  |Max LOB size (kb)                    |32KB
  |Enable logging                       |Checked

Let sync only one table for simplicity

-   Table Mappings

  |Schema Name is       |DMS\_SAMPLE
  |-------------------- |-------------------------
  |Table name is like   |SPORTING\_EVENT\_TICKET
  |Action               |Include

-   Click on \'Create Task\'

Wait for the task to get created and start running.

At this stage, the database migration task should load 100% of data from
Oracle to Postgres. (This will usually take few 10s of minutes)

-   Monitoring the progress for your database migration task

    -   Select your newly create database migration task

    -   Click on 'Task monitoring' tab & review the cloud watch metrics
        for your task

    -   Click on 'Table statistics' tab & review table level stats for
        your migration

Once the S3 DMS task is completed, you could verify the files in S3

![](.//media/image47.png)

In the Amazon S3 bucket , you will find the csv files.

![](.//media/image48.png)

## Configure Athena
----------------

### Go to Athena console

<https://ap-southeast-1.console.aws.amazon.com/athena/home?force&region=ap-southeast-1#query>

Create database

![](.//media/image49.png)

Run the script below

  Create database if not exists dmsworkshop;
  --------------------------------------------

Create table, change the s3 location to you bucket

 CREATE EXTERNAL TABLE IF NOT EXISTS                                   
 dmsworkshop.sporting\_event\_ticket (                                 
                                                                       
 \`ID\` int,                                                           
                                                                       
 \`SPORTING\_EVENT\_ID\` int,                                          
                                                                       
 \`SPORT\_LOCATION\_ID\` int,                                          
                                                                       
 \`SEAT\_LEVEL\` double,                                               
                                                                       
 \`SEAT\_SECTION\` string,                                             
                                                                       
 \`SEAT\_ROW\` string,                                                 
                                                                       
 \`SEAT\` string,                                                      
                                                                       
 \`TICKETHOLDER\_ID\` int,                                             
                                                                       
 \`TICKET\_PRICE\` double                                              
                                                                       
 )                                                                     
                                                                       
 ROW FORMAT SERDE                                                      
 \'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe\'                
                                                                       
 WITH SERDEPROPERTIES (                                                
                                                                       
 \'serialization.format\' = \',\',                                     
                                                                       
 \'field.delim\' = \',\'                                               
                                                                       
 ) LOCATION                                                            
 \'s3://{bucket\_name}/dms/DMS\_SAMPLE/SPORTING\_EVENT\_TICKET/\'      
                                                                       
 TBLPROPERTIES (\'has\_encrypted\_data\'=\'false\');                   

Run the script below to verify

  -----------------------------------------------------------
  Select count(\*) from dmsworkshop.sporting\_event\_ticket
  -----------------------------------------------------------

Clean Up Your Lab Environment:

DO NOT FORGET TAKE DOWN YOUR ENVIRONMENT

1.  Stop and delete your database migration tasks in DMS

2.  Delete the source/target endpoints in DMS

3.  Delete your DMS replication instance

4.  Delete the cloud formation template

Delete CloudFormation stack from the CloudFormation console.

Appendix



| Oracle - Get row count for all tables  
|-----------------------------------
| SELECT table\_name, num\_rows                            
                                                          
 FROM dba\_tables                                         
                                                          
 WHERE owner = \'DMS\_SAMPLE\'                            
                                                          
 ORDER BY table\_name;                                    
 Postgres                                                 
 SELECT relname AS table\_name, n\_live\_tup AS num\_rows 
                                                          
 FROM pg\_stat\_user\_tables                              
                                                          
 WHERE schemaname = \'dms\_sample\'                       
                                                          
 ORDER BY table\_name                                     

| Oracle - Command to get database size on disk   
|-----------------------------------                      
| SELECT owner, SUM(bytes) / 1024 / 1024 Size\_MB                       
                                                                       
 FROM dba\_segments                                                    
                                                                       
 WHERE owner = \'DMS\_SAMPLE\'                                         
                                                                       
 group by owner;                                                       
 Postgres - Command to get database size on disk                       
 SELECT pg\_size\_pretty(CAST((SELECT SUM(pg\_total\_relation\_size    
 (table\_schema \|\| \'.\' \|\| table\_name)) FROM                     
 information\_schema.tables WHERE table\_schema = \'dms\_sample\') AS  
 BIGINT)) AS tables\_schema\_size                                      

  Drop dms\_sample and restart dms task
  ---------------------------------------
  drop schema dms\_sample cascade;

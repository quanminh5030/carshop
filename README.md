# case2020s-backend Database Documentation
Information about the database design, creation, security, and tuning can be found here.  
Documentation written By: Arttu, Geoffrey, and Simon
Updated By: Anton, Alexander, Raymond, and Sanya

## Design

### Data Types
| Column        | Datatype       | NULL     | Constraints          | Explanation                                                                                                                 |
|---------------|----------------|----------|----------------------|-----------------------------------------------------------------------------------------------------------------------------|
| id            | int            | NOT NULL | PK\(auto generated\) | Primary key for the table\. This is a surrogate key and it is auto generated\.                                              |
| startTime     | timestamp      | NOT NULL |                      | Timestamp that is retrieved from the XML\-file test tags which describes at what time the test started\.                    |
| endTime       | timestamp      | NOT NULL |                      | Timestamp that is retrieved from the XML\-file test tags which describes at what time the test ended\.                      |
| testTrigger   | varchar\(50\)  | NOT NULL |                      | String that describes what invoked the test execution\.                                                                     |
| testType      | varchar\(50\)  | NOT NULL |                      | String that describes what type of a test is in question \(integration, system, unit test\)                                 |
| componentName | varchar\(50\)  | NOT NULL |                      | String that describes what type of a test is in question \(integration, system, unit test\)                                 |
| totalFail     | int            | NOT NULL |                      | Number describing the number of smaller tests inside the XML\-file’s test tag that failed\.                                 |
| totalPass     | int            | NOT NULL |                      | Number describing the number of smaller tests inside the XML\-file’s test tag that passed\.                                 |
| documentation | varchar\(1000\) | Nullable |                      | Some test tags have documentation tags that describe the test\. This is nullable since it is not found from all test tags\. |
| hasPassed     | Boolean        | NOT NULL |                      | Boolean value that shows if the test passed or not\.                                                                        |

*\*(trigger is a keyword in MariaDB and cannot be used by itself)*  
## installation and Initial Setup
MariaDB implementation on remote debian host.  
For the Back-End to communicate with the Database a MySQL driver is needed.  

### Installation for Linux
The following steps have been taken to create the DBMS instance by terminal commands.  

1. `sudo apt update` - Make sure the system is up to date.  
2. `sudo apt install mariadb-server` - Installation of mariaDB, includes MySQL dependencies.  
3. `sudo mysql_secure_installation` - Built in shell script to improve security.  
> - You can set a password for root accounts.  
> - You can remove root accounts that are accessible from outside the local host.  
> - You can remove anonymous-user accounts.  
> - You can remove the test database, which by default can be accessed by anonymous users.  
  
    "Y" yes to all for the best initial implementation security.  
4. `sudo systemctl status mariadb` - MariaDB service will start automatically, verify.  
5. `sudo mariadb` - Accessing mariaDB instance.  
6. `GRANT ALL ON *.* to '<username>'@'localhhost' INDENTIFIED BY '<password>' WITH GRANT OPTION;` - Creating super user account with full permissions.  
*\*Replace `<username>` and `<password>` with valid information.*  
7. `FLUSH PRIVILEGES;` - Flush the cache to enforce changes we have made.  
8. `exit;` - Exit mariadb cli.  
9. `mysqladmin -u <username> -p` - Test the user account created and login, you will be prompted for a password.  

### Allowing Remote Connections to the DB
Users have been created to access the dbms locally, however we are utilizing SSH connection to the server and can allow these type of remote connections.  

1. Navigate to /etc/mysql/mariadb.conf.d/
2. Edit 50-server.cnf file
3. Find bind-address lower in the file
4. Change 127.0.0.1 to 0.0.0.0
5. Save and exit,
6. restart mariadb (sudo service mariadb restart)

### Creating The Table
The table 'tablename' can be created with the following SQL Query.  

```SQL
CREATE TABLE 'tablename' (id INT NOT NULL AUTO_INCREMENT, startTime TIMESTAMP
NOT NULL, endTime TIMESTAMP NOT NULL, testTrigger VARCHAR(50) NOT NULL,
testType VARCHAR(50) NOT NULL, componentName VARCHAR(50) NOT NULL, totalFail
INT NOT NULL, totalPass INT NOT NULL, documentation VARCHAR(1000), hasPassed
BOOLEAN NOT NULL, PRIMARY KEY(id));
```

### Creating Low Level User
For our Back-End implementation we created a user account with the lowest level of access for better security.  

1. `CREATE USER '<username>'@'localhost' IDENTIFIED BY '<password>';` - Like before create a user, except here no privileges are granted.
2. `GRANT SELECT ON 'tablename' to '<username>'@'localhost';` - Granting right to select data from the table.
3. `GRANT INSERT ON 'tablename' to '<username>'@'localhost';` - Granting right to insert data from the table.
> - Can not delete rows
> - Can not update rows
> - Can not change/create tables
> - Can only view the table 'tablename' and it's data
> - Can only view the database in which 'tablename' resides in.

## Remote Connection
Connections to the data base are made through SSH tunnels, port 22.  
MariaDB is listening on default port 3306.  
For the Back-end to be able to communicate with the DBMS the tunnel must be created.  

### Command Line
A SSH tunnel needs to be created before mariaDB can be accessed remotely.  
  
1. `ssh -f <Host-username>@<DB-host-IP-address> -L 3308:localhost:3306 -N` - SSH tunnel including portfowarding from local machine 3308 to dbms host 3306.  
*\*Do note that SSH requires credentials for the host on which the DBMS recides.*  
2. `mysql -u <DB-username> -p -h 127.0.0.1 -P 3308` - Opening DB prompt with DB credentials.  


### MySQL Workbench
For modifying the DBMS or easier work environment MySQL Workbench can be used with it's built in SSH module.  
  
Connection Method: `Standard TCP/IP over SSH`  
SSH Hostname: `<Host-IP-address>`  
SSH Username: `<Host-username>`  
MySQL Hostname: `127.0.0.1`  
MySQL Server port: `3306`  
Username: `<DB-username>`  

*\*You will be prompted for passwords upon opening a connection.*  

### Environmental Variables From Back-End
We utilize knex to connect to the Database from the Back-End Server, this is done through the means of environmental parameters.  
The .env file is not present on this github in order to hide project details. (.env file in node.js root folder)  
The following variables are needed:  
PORT: `<listening-port>` - The listening port for the node.js app.  
DB_URI: `<ip-address-db-server>` - The IP address for the DB server (our case localhost because of port forwarding).  
DB_PORT: `<db-server-port>`  - Default 3306 without SSH Tunnel,  we port forward local 3308 to remote 3306.  
DB_SCHEMA: `<db-database-name>` - Use the schema/datbase name  
DB_USER: `<db-username>`  - The db user account  
DB_PASSWORD: `<db-user-password`>  - The password for the user account

## Tuning

### Slow Query
To help with measuring future performance, we have introduced a slow query log that logs
all queries that surpass a specified time limit to the slow query log. This can help us pinpoint
areas of improvement in our database.  

1. Log in to the database server
2. Log in to MariaDB using the admin log in details
3. To enable the slow query log, type the following command:  
`SET GLOBAL slow_query_log=1;`
4. To specify the file name of the slow query log, use the following command:  
`SET GLOBAL slow_query_log_file='mariadb-slow.log';`
5. By default the output is saved to a file. However, you can also specify that the output is
saved to a table. You can set either with one of the following commands:  
`SET GLOBAL log_output='FILE';`  
OR  
`SET GLOBAL log_output='TABLE';`  
*\*NOTE: As the default is that the output is saved to a file, we have not used this command but it could be useful moving forward.*
6. To set the time limit that is considered slow, we can use the following command:  
`SET GLOBAL long_query_time=1;`  
*\*NOTE: As we are in development phase, we have defined a long query as a query taking 1 second, as above. This will need to be changed when large amounts of data are added to the db as performance will slow naturally. To change the slow query time, simply run the command again and replace 1 with the number of seconds you consider to be a long query.*
7. We can also log queries that don’t use indexes as these queries are highly likely to be able to
be optimized with an index or a rewrite of the query. We do this with the following
command:  
`SET GLOBAL log_queries_not_using_indexes=ON;`  
  
These commands are the most relevant to our project and therefore are the commands we
have used. However, there are more options available with slow query log in MariaDB.
Please see https://mariadb.com/kb/en/slow-query-log-overview/ for more details.

### Indexes
Indexes are used to increase the performance of querries within the dbms.  
  
Case:  
> - Customer wants data between specific date ranges, or after x date
> - Customer wants data specific to a given component
> - Customer wants data according to a given test type (e.g.: integration, system, …)
> - Customer wants data specific to trigger type (e.g.: push to GitHub, manual testing, …)

Solution:  
> - Non-clustered indexes on possible frequently used WHERE statements in the backend (componentName, testTrigger, startTime, testType)
  
| Index Name        | Index Type | Column        | Table      |
|-------------------|------------|---------------|------------|
| IX\_componentName | B\-Tree    | componentName | 'tablename' |
| IX\_startTime     | B\-Tree    | startTime     | 'tablename' |
| IX\_testType      | B\-Tree    | testType      | 'tablename' |
| IX\_testTrigger   | B\-Tree    | testTrigger   | 'tablename' |

```SQL
CREATE INDEX IX_componentName on 'tablename' (componentName) USING BTREE;
CREATE INDEX IX_startTime on 'tablename' (startTime) using BTREE;
CREATE INDEX IX_testType on 'tablename' (testType) using BTREE;
CREATE INDEX IX_testTrigger on 'tablename' (testTrigger) using BTREE;
```

### Indexes Testing
The test data consisted of 29,552 rows in total.  
 \* (Clear query cache after every SELECT statement, “RESET QUERY CACHE”)  
   
`SELECT * FROM 'tablename' WHERE componentName = "componentA";`  
Retrieved Rows: 163  
Performance without index: 0.022 seconds.  
Performance with index: 0.001 seconds.  
  
`SELECT * FROM 'tablename' WHERE (startTime Between "2020-09-13 00:00:00" AND "2020-12-30 00:00:00");`  
Retrieved Rows: 49  
Performance without index: 0.028 seconds.  
Performance with index: 0.002 seconds.  
  
`SELECT * FROM 'tablename' where testType = "int";`  
Retrieved Rows: 9  
Performance without index: 0.021 seconds.  
Performance with index: 0.001 seconds.  


`SELECT * FROm 'tablename' WHERE testTrigger = “testTriggerA”;`  
Retrieved Rows: 20  
Performance without index: 0.021 seconds.  
Performance with index: 0.003 seconds.  

## Extra features

### Views
We implemented views to present data in a useful for analytical review.  
> - MonthlyStatistics: Displaying test results per component per month.  
> - YearlyStatistics: Displaying test results per component per year.  
> - LastWeek: Displaying test results per component from the last week.  

```SQL
CREATE VIEW MonthlyStatistics AS
SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', year(startTime) as 'Year', month(startTime) as 'Month'
FROM 'tablename' GROUP BY componentName, month(startTime), year(startTime) ORDER BY ComponentName ASC, year(startTime) DESC, month(startTime) DESC;
```

```SQL
CREATE VIEW YearlyStatistics AS
SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', year(startTime) as 'Year'
FROM 'tablename' GROUP BY componentName, year(startTime) ORDER BY ComponentName ASC, year(startTime) DESC;
```

```SQL
CREATE VIEW LastWeek AS
SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', year(startTime)
FROM 'tablename' WHERE startTime > (DATE_SUB(curdate(), INTERVAL 7 DAY))  AND startTime <= curdate() GROUP BY componentName ORDER BY ComponentName;
```

### Stored Procedures
We utilized stored procedures to fufill possible popular requested data in analytical format.
The stored procedures accept parameters to allow dynamic results.    
  
#### getStatistics(startDate, endDate, component):  
> - Retrieve data between 2 dates  
> - Makes sure there are multiple data points  
> - Date range of 14 days or less will result in daily statistics for the given component (useful for sprints).
> - Date ranges bigger than 14 days and smaller than 8 weeks will result in weekly statistics for the given component.
  
Example Usage:
```SQL
CALL getStatistics('2020-09-14', '2020-09-27', 'componentB');
CALL getStatistics('2020-09-14', '2020-10-30', 'componentB');
CALL getStatistics('2020-09-14', '2021-04-30', 'componentB');
CALL getStatistics('2019-09-14', '2021-08-30', 'componentB');
CALL getStatistics('2019-09-14', '2022-08-30', 'componentB');
```
  
Date formats used in the procedures can be both timestamp or date format ((YYYY-MM-DD) HH:MM:SS)
  
Procedure:  
```SQL
DELIMITER //
CREATE PROCEDURE getStatistics(
	IN startDate timestamp,
	IN endDate timestamp,
    IN compName VARCHAR(250)
	)
BEGIN
	DECLARE amountDays INT DEFAULT (DATEDIFF(endDate,startDate));
    /* If less or equals 14 days  */
    IF (amountDays <= 14) THEN
	SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', DATE(startTime) as 'Date'
    FROM 'tablename'
    WHERE startTime >= startDate AND endTime <= endDate AND componentName = compName
    GROUP BY ComponentName,  DAY(startTime);
    END IF;
    /* If less or equals 8 Weeks */
    IF (amountDays <= 56 AND amountDays > 14) THEN
    SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', YEARWEEK(startTime) as 'YearWeek'
    FROM 'tablename'
    WHERE startTime >= startDate AND endTime <= endDate AND componentName = compName
    GROUP BY ComponentName,  YEARWEEK(startTime);    
    END IF;
    /* If less than 8 months */
    IF (amountDays > 56 AND amountDays< 240) THEN
	SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', MONTH(startTime) as 'Month', YEAR(startTime) as 'Year'
    FROM 'tablename'
    WHERE startTime >= startDate AND endTime <= endDate AND componentName = compName
    GROUP BY ComponentName,  MONTH(startTime);   
    END IF;
     /*If less or equals than 2 years*/
    IF (amountDays <= 730 AND amountDays > 240) THEN
	SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', QUARTER(startTime) as 'Quarter', YEAR(startTime) as 'Year'
    FROM 'tablename'
    WHERE startTime >= startDate AND endTime <= endDate AND componentName = compName
    GROUP BY ComponentName,  QUARTER(startTime);
    END IF;
     /*If more than 2 years*/
    IF (amountDays > 730) THEN
	SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', YEAR(startTime) as 'Year'
    FROM 'tablename'
    WHERE startTime >= startDate AND endTime <= endDate AND componentName = compName
    GROUP BY ComponentName,  Year(startTime);
    END IF;
END //
DELIMITER ; 
``` 

#### getStatisticsByTimeFrame(startDate, endDate, component, period):  
Same as getStatistics except that you can decide if you want datapoints to be grouped daily, weekly, monthly, quarterly, yearly for a given component.
  
Example Usage:  
```SQL
CALL getStatisticsByTimeFrame('2020-09-14', '2022-08-30', 'componentB', 'Day');
CALL getStatisticsByTimeFrame('2020-09-14', '2022-08-30', 'componentB', 'Week');
CALL getStatisticsByTimeFrame('2020-09-14', '2022-08-30', 'componentB', 'Month');
CALL getStatisticsByTimeFrame('2020-09-14', '2022-08-30', 'componentB', 'Quarter');
CALL getStatisticsByTimeFrame('2020-09-14', '2022-08-30', 'componentB', 'Year');

```
  
Date formats used in the procedures can be both timestamp or date format ((YYYY-MM-DD) HH:MM:SS)  
  
Procedure:  
```SQL
DELIMITER //
CREATE PROCEDURE getStatisticsByTimeFrame(
	IN startDate timestamp,
	IN endDate timestamp,
    IN compName VARCHAR(250),
    IN period VARCHAR(7)
	)
BEGIN
/* Day */
IF (period = 'Day') THEN
	SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', DATE(startTime) as 'Date'
    FROM 'tablename'
    WHERE startTime >= startDate AND endTime <= endDate AND componentName = compName
    GROUP BY ComponentName,  DATE(startTime);
END IF;
/* Week */
IF (period = 'Week') THEN
	SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', YEARWEEK(startTime) as 'YearWeek'
    FROM 'tablename'
    WHERE startTime >= startDate AND endTime <= endDate AND componentName = compName
    GROUP BY ComponentName,  YEARWEEK(startTime);
END IF;
/* Month */
IF (period = 'Month') THEN
	SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', Month(startTime) as 'Month', Year(startTime) as 'Year'
    FROM 'tablename'
    WHERE startTime >= startDate AND endTime <= endDate AND componentName = compName
    GROUP BY ComponentName,  Month(startTime), Year(startTime);
END IF;
/* Quarter */
IF (period = 'Quarter') THEN
	SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', Quarter(startTime) as 'Quarter', Year(startTime) as 'Year'
    FROM 'tablename'
    WHERE startTime >= startDate AND endTime <= endDate AND componentName = compName
    GROUP BY ComponentName,  Quarter(startTime), Year(startTime);
END IF;
/* Year */
IF (period = 'Year') THEN
	SELECT componentName, SUM(totalFail) as '#FailedTests', SUM(totalPass)as '#PassedTests', SUM(totalFail + totalPass) as '#Tests', Year(startTime) as 'Year'
    FROM 'tablename'
    WHERE startTime >= startDate AND endTime <= endDate AND componentName = compName
    GROUP BY ComponentName,  Year(startTime);
END IF;

END //
DELIMITER ;
```

# Database in the middle of a restore Repro

ℹ️ This issue occurs when running inside or outside of docker, in Azure and outside of Azure and when one instance of SQL Server is outside of Azure and another inside of Azure.  The repro below uses Docker in order to simplify steps to reproduce but Docker is not required to reproduce the issue.

In the situation where an existing database has been dropped and a restored database renamed to the name of the previsouly existing database, if another database is created and in the "restoring" state, linked server queries fail to execute encountering the error "Database `<nameofrestoringdatabase>` cannot be opened.  It is in the middle of a restore.", however the query is targeting the database that is NOT in the middle of a restore.

For example, the following query is targeting the database `Northwind_live` but the error message indicates `Northwind_next_inprogress`:

**Query:**

```sql
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Employees];
```

**Error:**

```sql
Msg 927, Level 14, State 2, Line 1
Database 'Northwind_next_inprogress' cannot be opened. It is in the middle of a restore.
```

## Steps to Reproduce

1. Create `SQL_DB_HOST` - Will host the Northwind Database
```sh
docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=yourStrong(!)Password' -p 1433:1433 --name sql_db_host -d mcr.microsoft.com/mssql/server:2019-CU10-ubuntu-20.04 
```
2. Create `SQL_DB_CLIENT` - Will access `SQL_DB_HOST` via linked server
```sh
docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=yourStrong(!)Password' -p 1434:1433 --name sql_db_client -d mcr.microsoft.com/mssql/server:2019-CU10-ubuntu-20.04
```
3. Open SQL Server Management Studio (SSMS)
4. Connect to `SQL_DB_HOST` (localhost,1433)
5. [Create Northwind Database](https://raw.githubusercontent.com/microsoft/sql-server-samples/master/samples/databases/northwind-pubs/instnwnd.sql) on `SQL_DB_HOST`
6. Backup Northwind Database
```sql
USE [master]
GO
BACKUP DATABASE [Northwind] TO  DISK = N'/var/opt/mssql/data/Northwind_Full.bak' WITH NOFORMAT, NOINIT,  NAME = N'Northwind-Full Database Backup', SKIP, NOREWIND, NOUNLOAD,  STATS = 10
GO
```
7. Drop Northwind Database
```sql
USE [master]
GO
ALTER DATABASE [Northwind] SET SINGLE_USER WITH ROLLBACK IMMEDIATE
GO
DROP DATABASE [Northwind]
GO
```
8. Restore Northwind Backup to `Northwind_live` database
```sql
USE [master]
GO
RESTORE DATABASE [Northwind_live] FROM  DISK = N'/var/opt/mssql/data/Northwind_Full.bak' WITH  FILE = 1,  MOVE N'Northwind' TO N'/var/opt/mssql/data/Northwind_live.mdf',  MOVE N'Northwind_log' TO N'/var/opt/mssql/data/Northwind_live.ldf',  NOUNLOAD,  STATS = 5
GO
```
9. Obtain IP Address of `SQL_DB_HOST`
```sh
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' sql_db_host
```
10. Connect to `SQL_DB_CLIENT` (localhost, 1434)
11. Create Linked Server on `SQL_DB_CLIENT` to `SQL_DB_HOST` replacing `<SQL_DB_HOST_IP_ADDRESS>` with the IP address obtained in Step #9
```sql
USE [master]
GO
EXEC master.dbo.sp_addlinkedserver @server = N'sql_db_host', @srvproduct=N'', @provider=N'SQLNCLI', @provstr=N'Library=DMBSSOCN;Server=<SQL_DB_HOST_IP_ADDRESS>;Database=master;'
GO
EXEC master.dbo.sp_addlinkedsrvlogin @rmtsrvname=N'sql_db_host',@useself=N'False',@locallogin=NULL,@rmtuser=N'sa',@rmtpassword='yourStrong(!)Password'
GO
```
12. Execute linked server queries from `SQL_DB_CLIENT` to verify (can all be executed in one batch)
```sql
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Employees];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Employees];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Categories];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Categories];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Customers];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Customers];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Shippers];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Shippers];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Suppliers];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Suppliers];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Orders];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Orders];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Products];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Products];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Order Details];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Order Details];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[CustomerCustomerDemo];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[CustomerCustomerDemo];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[CustomerDemographics];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[CustomerDemographics];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Region];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Region];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Territories];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Territories];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[EmployeeTerritories];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[EmployeeTerritories];
```
13. Create `Northwind_next` database on `SQL_DB_HOST`
```sql
DECLARE @BackupFileName nVarchar(1000) = 'Northwind_Full.bak',
	@DatabaseName nVarchar(1000) = 'Northwind_next',
	@DataPath nVarchar(100) = '/var/opt/mssql/data/',
	@Command_Full_REC nVarchar(4000)
SET @Command_Full_REC = 'RESTORE DATABASE [' + @DatabaseName + '] FROM DISK = N''' + @DataPath + @BackupFileName + '''' + ' WITH  FILE = 1 ,' + 'MOVE N''Northwind''  TO N''' + @DataPath + @DatabaseName + '.mdf'',MOVE N''Northwind_log'' TO N''' + @DataPath + @DatabaseName + '.ldf'',RECOVERY'
EXEC (@Command_Full_REC);
```
14. Rename `Northwind_live` database to `Northwind_tmp` on `SQL_DB_HOST`
```sql
DECLARE @DatabaseNameLive NVARCHAR(100) = 'Northwind_live',
	@DatabaseNamePrevTemp NVARCHAR(100) = 'Northwind_tmp',
	@Command_full_live_rename nVarchar(4000)
SET @Command_full_live_rename = 'ALTER DATABASE [' + @DatabaseNameLive + '] MODIFY NAME = [' + @DatabaseNamePrevTemp + '];'
EXEC (@Command_full_live_rename);
```
15. Rename `Northwind_next` database to `Northwind_live` on `SQL_DB_HOST`
```sql
DECLARE @DatabaseNameNext nVarchar(1000) = 'Northwind_next', 
	@DatabaseNameLive NVARCHAR(100) = 'Northwind_live',
	@Command_full_Rec_rename nVarchar(4000)
SET @Command_full_Rec_rename = 'ALTER DATABASE [' + @DatabaseNameNext + '] MODIFY NAME = [' + @DatabaseNameLive + '];'
EXEC (@Command_full_Rec_rename);
```
16. Drop `Northwind_tmp` on `SQL_DB_HOST`
```sql
DECLARE @DatabaseNamePrevTemp NVARCHAR(100) = 'Northwind_tmp',
		@Command_Bkp_Drop nVarchar(4000)
SET @Command_Bkp_Drop = 'DROP DATABASE [' + @DatabaseNamePrevTemp + ']';
EXEC (@Command_Bkp_Drop);
```
17. Create `Northwind_next_inprogress` database on `SQL_DB_HOST`
```sql
DECLARE @BackupFileName nVarchar(1000) = 'Northwind_Full.bak',
	@DatabaseNameNext nVarchar(1000) = 'Northwind_next_inprogress', 
	@DataPath nVarchar(100) = '/var/opt/mssql/data/',
	@Command_Full_concurdb nVarchar(4000)
SET @Command_Full_concurdb = 'RESTORE DATABASE [' + @DatabaseNameNext + '] FROM DISK = N''' + @DataPath + @BackupFileName+'''' + ' WITH  FILE = 1 ,' + 'MOVE N''Northwind''  TO N''' + @DataPath + @DatabaseNameNext + '.mdf'',MOVE N''Northwind_log'' TO N''' + @DataPath + @DatabaseNameNext + '.ldf'',NORECOVERY, NOUNLOAD, STATS = 5'
EXEC (@Command_Full_concurdb);
```
18. Execute linked server queries from `SQL_DB_CLIENT` to verify (can all be executed in one batch)
```sql
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Employees];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Employees];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Categories];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Categories];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Customers];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Customers];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Shippers];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Shippers];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Suppliers];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Suppliers];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Orders];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Orders];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Products];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Products];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Order Details];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Order Details];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[CustomerCustomerDemo];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[CustomerCustomerDemo];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[CustomerDemographics];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[CustomerDemographics];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Region];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Region];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[Territories];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[Territories];
SELECT * FROM [sql_db_host].[Northwind_live].[dbo].[EmployeeTerritories];SELECT COUNT(*) FROM [sql_db_host].[Northwind_live].[dbo].[EmployeeTerritories];
```

## Expected Result
All queries execute successfully and have identical behavior to Step #12

## Actual Result
Queries fail with following error message indicating `Northwind_next_inprogress cannot be opened. It is in the middle of a restore.`, however `Northwind_live` is the database being accessed, not `Northwind_next_inprogress`.

# SQLTools
A collection of tools for SQL Server administration and development  
 
/*****************************************************************************
       Joe Kelly
       2015-09-29 13:00:54.183
 
       A collection of tools for database authoring and monitoring 
       
       Note: by default this script turns off execution for this spid. Run 
       
              SET NOEXEC OFF 
              
       to be able to execute any content 
       
       Note: uncomment 'EXEC's' where appropriate
       
       Note: sources cited where applicable, otherwise curated from BOL/MSDN 
	   or self created 
 
*****************************************************************************/

 
------------------------------------------------------------------------------
	-- Common commands,  settings 
	 
		   SET NOEXEC ON 
		   -- SET NOEXEC OFF 
	 
		   -- SET NOCOUNT ON 
		   -- SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED 
	 
		   -- (the 'recommended' way to look at sysobjects)
		   -- SELECT TOP 33 * FROM INFORMATION_SCHEMA.Tables 
		   -- SELECT TOP 33 * FROM INFORMATION_SCHEMA.Columns 
		   -- SELECT TOP 33 * FROM INFORMATION_SCHEMA.Views 
		   -- SELECT TOP 33 * FROM INFORMATION_SCHEMA.Routines  
	 
		   -- drop temp table 
		   -- IF OBJECT_ID('tempdb..#dbSPIDS') IS NOT NULL DROP TABLE #dbSPIDS
	 
------------------------------------------------------------------------------
-- Find tables with column named ... 
 
       EXEC Utility.dbo.findTWithC 'MyDB-tmar',   'product_code'
       
------------------------------------------------------------------------------
-- Find routine with text (wraps 4000 N) 
-- http://www.sqltact.com/2012/04/syssqlmodules-vs-informationschemarouti.html
 
       DECLARE @searchString NVARCHAR (4000) = '' 
       SET @searchString = 'product'
 
       SELECT s.name 
       +      '.'    
       +      o.name
       ,      o.type_desc
       ,      m.[definition]
       FROM   sys.sql_modules m
       JOIN   sys.objects o 
              ON     m.object_id = o.object_id
       JOIN   sys.schemas s 
              ON     s.schema_id = o.schema_id
       WHERE  m.[definition] like '%' + @searchString + '%'
       ORDER BY o.name
 
------------------------------------------------------------------------------
-- Try/catch syntax 
 
       BEGIN TRY 
              SELECT 2/0
       END TRY 
       BEGIN CATCH 
              SELECT Utility.dbo.fnGetErrorTryCatch () 
       END CATCH 
        
------------------------------------------------------------------------------
-- What users are active db x? 
USE master 
GO 
 
       IF OBJECT_ID('tempdb..#dbSPIDS') IS NOT NULL DROP TABLE #dbSPIDS
 
       CREATE TABLE #dbSPIDS (
              SPID INT      
       ,      [Status] VARCHAR (256)     
       ,      [Login] VARCHAR (256)      
       ,      HostName  VARCHAR (256)    
       ,      BlkBy VARCHAR (256) 
       ,      DBName VARCHAR (256)       
       ,      Command VARCHAR (MAX)      
       ,      CPUTime INT   
       ,      DiskIO INT    
       ,      LastBatch VARCHAR (256)    
       ,      ProgramName VARCHAR (256) 
       ,      SPID2 INT     
       ,      REQUESTID INT 
       )
       INSERT #dbSPIDS EXEC sp_who2
 
--     SELECT d.* 
       SELECT 'KILL ',      d.SPID,       d.* 
       FROM   #dbSPIDS d 
       where login not like '%sa%'
--     WHERE  d.dbname LIKE '%MyDB%dnn%TMAR'
      --WHERE       d.dbname = 'MyDB-d-TMAR'
      -- and (d.dbname LIKE 'MyDBTMAR' OR d.dbname LIKE 'MyDBTMAR_new' )
       and (d.dbname LIKE '%760%' )
 
-- kill 54
 
------------------------------------------------------------------------------
-- What is the spid doing? 
 
       DBCC INPUTBUFFER (86) 

------------------------------------------------------------------------------
-- What text is the sp executing? 
       
       DECLARE @sqltext VARBINARY(128)
       SELECT @sqltext = sql_handle
       FROM sys.sysprocesses
       WHERE spid = 339 
       SELECT TEXT
       FROM sys.dm_exec_sql_text(@sqltext)
       GO
 
------------------------------------------------------------------------------
-- Renaming Databases 
 
sp_who2 
 
-- Optional but should be done for MyDB-DEV-NEW
EXEC Utility.dbo.BACKUP_APP_INTERFACE_PARAMETER_TBL_sp
GO 
-- kill 88
 
SP_WHO2 
 
-- Optional but should be done for MyDB-DEV-NEW
ALTER DATABASE [MyDB-TMAR] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO 
ALTER DATABASE [MyDB-TMAR-NEW] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO 
 
USE master;
GO
ALTER DATABASE [MyDB-TMAR]
Modify Name = [MyDB-TMAR_OLD];
GO
ALTER DATABASE [MyDB-TMAR-NEW]
Modify Name = [MyDB-TMAR];
GO
 
EXEC Utility.dbo.RESTORE_APP_INTERFACE_PARAMETER_TBL_sp 'MyDB-TMAR'
GO 
 
-- ALTER DATABASE [MyDB-TMAR] SET MULTI_USER WITH ROLLBACK IMMEDIATE;
-- ALTER DATABASE [MyDB-TMAR_OLD] SET MULTI_USER WITH ROLLBACK IMMEDIATE;
 
--use utility 
--go 
--select * from sysobjects where type = 'p' 
 
-- EXEC Utility.dbo.TMARDBRestore_RemoveTRSJobs 'MyJob'
 
EXEC Utility.dbo.TMARDBRestore_RemoveTRSJobs 'MyJob'
GO 
 
USE [MyDB-TMAR]
GO
 
DROP FULLTEXT INDEX ON MyDB_product_search 
GO 
--DROP FULLTEXT CATALOG MyDB_product_search 
--GO 
 
ALTER DATABASE [MyDB-TMAR] SET MULTI_USER;
GO
ALTER DATABASE [MyDB-TMAR_OLD] SET MULTI_USER;
GO 
USE [MyDB-TMAR] 
GO
IF EXISTS (
       SELECT * 
       FROM sys.database_principals
       WHERE name = 'MyDB-'
       )
DROP USER [MyDB-]
CREATE USER [MyDB-] FOR LOGIN [MyDB-]
-- ALTER ROLE [db_owner] ADD MEMBER [MyDB-]
EXEC sp_addrolemember N'db_owner', N'MyDB-'
 
-- NOTE: remember to add in user "Myweb" to prod1 and prod2 
USE [MyDB-TMAR] 
GO
IF EXISTS (
       SELECT * 
       FROM sys.database_principals
       WHERE name = 'Myweb'
       )
DROP USER [Myweb]
CREATE USER [Myweb] FOR LOGIN [Myweb]
-- ALTER ROLE [db_owner] ADD MEMBER [MyDB-]
EXEC sp_addrolemember N'db_owner', N'Myweb'
 
GO 
 
-- NOTE: remember to add in user "MyAdmin" to prod1 and prod2  
USE [MyDB-TMAR] 
GO
IF EXISTS (
       SELECT * 
       FROM sys.database_principals
       WHERE name = 'MyAdmin'
       )
DROP USER [MyAdmin]
CREATE USER [MyAdmin] FOR LOGIN [MyAdmin]
-- ALTER ROLE [db_owner] ADD MEMBER [MyDB-]
EXEC sp_addrolemember N'db_owner', N'MyAdmin'
 
------------------------------------------------------------------------------
-- Kill the active database connections 
-- Note: Uncomment the 'exec' 
 
       DECLARE       @execSql VARCHAR (1000) = '' 
       ,      @databaseName VARCHAR (100) = ''  
       ,      @dbid INT = 0 
 
       SET @execSql = '' 
       SET @databaseName = 'MyDB-DNN-TMAR'  
       SET @dbid = db_id(@databaseName) 
 
       IF (ISNULL (@dbid,   '') = '') PRINT 'Bad DB Name' 
 
       SELECT @execSql = @execSql + 'kill ' + CONVERT (CHAR(10), sp.spid) + '; '  
       FROM   master.dbo.sysprocesses  sp 
       WHERE   dbid = @dbid 
              AND    sp.DBID <> 0  
              AND sp.spid <> @@spid  
       SELECT @execSql  
--     EXEC (@execSql)
 
       GO
 
-- kill 59
 
------------------------------------------------------------------------------
---------------------------- Restore Area ------------------------------------
------------------------------------------------------------------------------
-- What is the status of the backups and restores? 
       SELECT 
              r.session_id
       ,      r.command
       ,      CONVERT(NUMERIC(6,2),      r.percent_complete) [Percent Complete]
       ,      CONVERT(VARCHAR(20), DATEADD(ms,r.estimated_completion_time,GetDate()),20) [ETA Completion Time]
       ,      CONVERT(NUMERIC(10,2),r.total_elapsed_time/1000.0/60.0) [Elapsed Min]
       ,      CONVERT(NUMERIC(10,2),r.estimated_completion_time/1000.0/60.0) [ETA Min]
       ,      CONVERT(NUMERIC(10,2),r.estimated_completion_time/1000.0/60.0/60.0) [ETA Hours]
       ,      CONVERT(VARCHAR(1000),     
                     (
                           SELECT SUBSTRING(
                                  text,r.statement_start_offset/2,  
                                         CASE 
                                                WHEN r.statement_end_offset = -1 
                                                       THEN 1000 
                                                ELSE (r.statement_end_offset-r.statement_start_offset)/2 
                                         END
                                  )
                           FROM sys.dm_exec_sql_text(sql_handle)
                     )
              )
       FROM sys.dm_exec_requests r WHERE command IN ('RESTORE DATABASE','BACKUP DATABASE')
 
------------------------------------------------------------------------------
-- Most recent Myweb_user 
 
       SELECT TOP 33 wu.* 
       FROM   dbo.Myweb_user wu
       ORDER BY wu.MODDATE DESC 
 
------------------------------------------------------------------------------
-- Recover, NOT Restore, the db 
-- https://msdn.microsoft.com/en-us/library/ms188696.aspx
 
-- 		RESTORE DATABASE [MyDB-DEV-NEW] WITH RECOVERY
 
------------------------------------------------------------------------------
-- Restore a full database backup 
 
-- 		RESTORE DATABASE [MyDB-DEV] FROM DISK = 'C:\MyDB.bak'
 
------------------------------------------------------------------------------
-- The restore operation halted prematurely.
-- Repeat the original RESTORE statement specifying WITH RESTART.
 
-- 		RESTORE DATABASE [MyDB-DEV] FROM DISK = 'C:\MyDB.bak' WITH RESTART
 
------------------------------------------------------------------------------
-- Rollback any transactions and set db to single user 
 
-- 		ALTER DATABASE [MyDB-DEV-NEW] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
 
------------------------------------------------------------------------------
-- Set to read only 
 
-- 		ALTER DATABASE [MyDB-DEV-NEW] SET READ_ONLY;
 
------------------------------------------------------------------------------
-- Set to multi user 
 
--     ALTER DATABASE [MyDB-DEV-NEW] SET MULTI_USER;
 
-- Same but rollback pending transactions 
 
--     ALTER DATABASE [MyDB-DEV-NEW] SET MULTI_USER WITH ROLLBACK IMMEDIATE;
 
------------------------------------------------------------------------------
-- Force the db offline 
 
--     ALTER DATABASE [MyDB-DEV-NEW] SET OFFLINE 
 
-- Same but rollback pending transactions 
 
--     ALTER DATABASE [MyDB-DEV-NEW] SET OFFLINE WITH ROLLBACK IMMEDIATE
  
------------------------------------------------------------------------------
-- File tools 
 
       -- Find the BAK files:
       SELECT TOP 100 
              m.physical_device_name
       ,      b.backup_start_date
       ,      b.backup_finish_date
       ,      b.backup_size/1024.0 BackupSizeKB
       FROM   msdb.dbo.backupset b
       JOIN   msdb.dbo.backupmediafamily m 
              ON     b.media_set_id = m.media_set_id
       WHERE  /* database_name like '%Mysmco%' */ 
              m.physical_device_name LIKE '%bak%' 
       ORDER BY 
              b.backup_finish_date DESC
 
       -- Find the MDF's and LDF's:
       DECLARE @command varchar(1000)
       SELECT @command = 'SELECT * FROM sys.database_files'
       EXEC sp_MSforeachdb @command
 
       -- Or ... easier
       EXEC sp_MSforeachdb 'USE ? SELECT ''?'', SF.filename,  SF.size FROM sys.sysfiles SF'
 
       -- Or ... even easier
       SELECT 'USE [' + name + ']; SELECT SF.filename, SF.size FROM sys.sysfiles SF'
       FROM master.dbo.sysdatabases 
 
------------------------------------------------------------------------------
-- Delete old backups. 
 
--     EXECUTE master.dbo.xp_delete_file 0,N'\\Myapp4\SomeSQLBACKUP\MyCoSQL-TEST','N',N'2017-01-05T11:14:40',1
 
------------------------------------------------------------------------------
-- What are the sizes of my db's?
 
       GO
       
       WITH fs
       AS (
              SELECT 
                     m.database_id
              ,      m.[type]
              ,      m.size * 8.0 / 1024 size
              FROM   sys.master_files m 
       )
       SELECT
              name
       ,      (SELECT SUM (size) FROM fs WHERE [type] = 0 AND fs.database_id = db.database_id) DataFileSizeMB
       ,      (SELECT SUM (size) FROM fs WHERE [type] = 1 AND fs.database_id = db.database_id) LogFileSizeMB
       FROM   sys.databases db
 
------------------------------------------------------------------------------
-- Find tables with foreign keys 
-- Source: http://stackoverflow.com/questions/483193/how-can-i-list-all-foreign-keys-referencing-a-given-table-in-sql-server 
 
       SELECT 
              t.name TableWithForeignKey
       ,      fk.constraint_column_id FK_PartNo 
       ,      c.name ForeignKeyColumn 
       FROM   sys.foreign_key_columns fk
       JOIN   sys.tables t 
              ON     fk.parent_object_id = t.[object_id] 
       JOIN   sys.columns c 
              ON     fk.parent_object_id = c.[object_id] 
              AND fk.parent_column_id = c.column_id
       JOIN   sys.tables t1
              ON     fk.referenced_object_id = t1.[object_id]  
       ORDER BY
              TableWithForeignKey
       ,      FK_PartNo
 
------------------------------------------------------------------------------
-- SSIS/SQLAgent - Enable User to See and Start Jobs
 
       -- To see, the user must be in the SQLAgentReaderRole or SQLAgentUserRole,      
       -- to start them the user must be in the SQLAgentOperaterRole. As these 
       -- only exist in MSDB then the user must also be a member there.
       
------------------------------------------------------------------------------
-- Tool for Scripting View Generation
 
       -- Given two databases, source and dest, this script will generate a READ 
       -- ONLY view in dest for each table or view in source. 
 
       -- http://joeskb.blogspot.com/2013/08/tool-for-scripting-view-generation.html
 
------------------------------------------------------------------------------
--  Get RowCount from a Dynamically Executed Statement
 
       DECLARE 
		  @Period VARCHAR (8) = '' 
    ,     @DEDFYXXRecords BIGINT = 0 
    ,     @EXECFYXX NVARCHAR (MAX) = ''
    ,     @RowCount BIGINT = 0;  
  
    SELECT @EXECFYXX = ' INSERT Data( ' 
    +    ' Period ' 
    +    ',   CostCenter '     
    +    ') ' 
    +    'SELECT ' 
    +    ' FYXX.Period ' 
    +    ',   FYXX.CostCenter '     
    +    'FROM DataFY'
    +    LEFT (@Period,    2) 
    +    ' FYXX ' 
    +    ' JOIN DataB da ' 
    +    '     ON FYXX.Account = da.Acct ' 
    +    ' WHERE da.GroupID = 99 '
    +    '    AND    FYXX.Period = '''
    +    @Period 
    +    '''; SELECT @RowCount = @@ROWCOUNT; ';
 
    -- EXEC sp_executesql @EXECFYXX, N'@RowCount BIGINT OUTPUT', @RowCount OUTPUT
 
    SELECT @DEDFYXXRecords = @RowCount
    
    SELECT @DEDFYXXRecords 
    
------------------------------------------------------------------------------
-- Dropping temp tables 
IF OBJECT_ID('tempdb..##badge_updt') IS NOT NULL DROP TABLE ##badge_updt
 
------------------------------------------------------------------------------
-- Find all db's in single user mode 
 
SELECT name,  user_access_desc
FROM sys.databases 
WHERE user_access_desc = 'SINGLE_USER'
 
------------------------------------------------------------------------------
-- Find long running transactions 
SELECT * FROM sys.dm_tran_database_transactions
 
------------------------------------------------------------------------------
-- Advanced commands for the terminal 
--sqlcmd -S aaopsql-upgrade.sf.aao.org -d master -U MyAdmin -P SomePass
--alter login [SomeDomain\SomeUser] with default_database = master
--alter login MyAdmin with default_database = master
 
------------------------------------------------------------------------------
-- Transaction log size 
DBCC SQLPERF(logspace)
  
------------------------------------------------------------------------------
-- Transaction log backups 
-- BACKUP LOG [foo] TO  DISK = N'\\netapp4\SQLBACKUPS\AAOPSQL-UPGRADE\Logs\Temp\fooLogTest.trn' 
--     WITH  RETAINDAYS = 1, NOFORMAT, NOINIT, NAME = N'foo-Full Database Backup', SKIP, NOREWIND, NOUNLOAD, STATS = 10
GO
  
------------------------------------------------------------------------------
-- Recovery Models 
SELECT name AS [Database Name],
recovery_model_desc AS [Recovery Model]
FROM sys.databases
GO
 
------------------------------------------------------------------------------
-- Last Restore time 
-- http://dba.stackexchange.com/questions/33703/how-to-query-last-restore-date-in-sql-server/33705 
WITH LastRestores AS
(
SELECT
    DatabaseName = [d].[name] ,
    [d].[create_date] ,
    [d].[compatibility_level] ,
    [d].[collation_name] ,
    r.*,
    RowNum = ROW_NUMBER() OVER (PARTITION BY d.Name ORDER BY r.[restore_date] DESC)
FROM master.sys.databases d
LEFT OUTER JOIN msdb.dbo.[restorehistory] r 
ON r.[destination_database_name] = d.Name
)
SELECT *
FROM [LastRestores]
WHERE [RowNum] = 1
 
------------------------------------------------------------------------------
-- Space per db 
-- https://www.mssqltips.com/sqlservertip/2393/determine-sql-server-memory-use-by-database-and-object/
 
DECLARE @total_buffer INT;
 
SELECT @total_buffer = cntr_value
FROM sys.dm_os_performance_counters 
WHERE RTRIM([object_name]) LIKE '%Buffer Manager'
AND counter_name = 'Database Pages';
 
;WITH src AS
(
SELECT 
database_id, db_buffer_pages = COUNT_BIG(*)
FROM sys.dm_os_buffer_descriptors
--WHERE database_id BETWEEN 5 AND 32766
GROUP BY database_id
)
SELECT
[db_name] = CASE [database_id] WHEN 32767 
THEN 'Resource DB' 
ELSE DB_NAME([database_id]) END,
db_buffer_pages,
db_buffer_MB = db_buffer_pages / 128,
db_buffer_percent = CONVERT(DECIMAL(6,3), 
db_buffer_pages * 100.0 / @total_buffer)
FROM src
ORDER BY db_buffer_MB DESC; 
 
------------------------------------------------------------------------------
-- SQL Server Memory 
-- https://msdn.microsoft.com/en-us/library/ms176018.aspx
 
SELECT  
(physical_memory_in_use_kb/1024) AS Memory_usedby_Sqlserver_MB,  
(locked_page_allocations_kb/1024) AS Locked_pages_used_Sqlserver_MB,  
(total_virtual_address_space_kb/1024) AS Total_VAS_in_MB,  
process_physical_memory_low,  
process_virtual_memory_low  
FROM sys.dm_os_process_memory; 
 
-- For excessive details use 
-- DBCC MEMORYSTATUS
 
------------------------------------------------------------------------------
-- Truncate log file 
 
SELECT file_id, name  
FROM sys.database_files;  
GO  
-- DBCC SHRINKFILE (2, TRUNCATEONLY); 
 
------------------------------------------------------------------------------
-- Drop full text index RUN THIS FIRST or you will be caught in an endless blocking cycle 
USE [MyDB-DEV]
GO 
ALTER DATABASE [MyDB-DEV] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO 
DROP FULLTEXT INDEX ON MyDB_product_search 
GO 
DROP FULLTEXT CATALOG MyDB_product_search 
GO 
ALTER DATABASE [MyDB-DEV] SET MULTI_USER WITH ROLLBACK IMMEDIATE;
 
------------------------------------------------------------------------------
-- Find fulltext catalog and indices  
 
SELECT 
    t.name AS TableName, 
    c.name AS FTCatalogName ,
    i.name AS UniqueIdxName,
    cl.name AS ColumnName
FROM 
    sys.tables t 
JOIN 
    sys.fulltext_indexes fi 
ON 
    t.[object_id] = fi.[object_id] 
JOIN 
    sys.fulltext_index_columns ic
ON 
    ic.[object_id] = t.[object_id]
JOIN
    sys.columns cl
ON 
        ic.column_id = cl.column_id
    AND ic.[object_id] = cl.[object_id]
JOIN 
    sys.fulltext_catalogs c 
ON 
    fi.fulltext_catalog_id = c.fulltext_catalog_id
JOIN 
    sys.indexes i
ON 
        fi.unique_index_id = i.index_id
    AND fi.[object_id] = i.[object_id];
 
--select distinct
--    object_name(fic.[object_id])as table_name,
--    [name]
--from
--    sys.fulltext_index_columns fic
--    join sys.columns c
--        on c.[object_id] = fic.[object_id]
--        and c.[column_id] = fic.[column_id]
 
------------------------------------------------------------------------------
-- Find/drop offline databases 
 
SELECT DISTINCT 
'drop database [' 
+ db.name
+ ']'
, db.name
--'DB_NAME' = db.name,
--'FILE_NAME' = mf.name,
--'FILE_TYPE' = mf.type_desc,
--'FILE_PATH' = mf.physical_name
FROM
sys.databases db
JOIN sys.master_files mf
ON db.database_id = mf.database_id
WHERE db.state = 6 -- OFFLINE
order by db.name 

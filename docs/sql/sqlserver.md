# SQL 

## List database sizes

``` sql linenums="1"
SELECT
    D.name,
    F.Name AS FileType,
    F.physical_name AS PhysicalFile,
    F.state_desc AS OnlineStatus,
    CAST(F.size AS bigint) * 8*1024 AS SizeInBytes,
    CAST((F.size*8.0)/1024/1024 AS decimal(18,3)) AS SizeInGB
FROM
    sys.master_files F
    INNER JOIN sys.databases D ON D.database_id = F.database_id
where
    charindex('datag 2014', f.physical_name) > 0
ORDER BY SizeInBytes desc
```

## Blocking queries

!!! info 
    This query can help you finding the root cause of performance issues (for example: kpi's, lookups, data searches, ...)
    
    You run this query on the database via SQL Server Management Studio, and it returns the active queries that requires a long time to process. You can see the sql statement, the time running and the process which is blocking the current query.

``` sql linenums="1"
use master;
select distinct
                                      command,
            db_name(r.database_id) as dbname ,
            percent_complete       as pctcomp,
            r.reads                          ,
            r.writes                         ,
            r.logical_reads                  ,
            r.cpu_time           /1000.                                                      as cpusec       ,
            r.total_elapsed_time / 1000.                                                     as elapsSec     ,
            '*'                  +substring(s.text, statement_start_offset / 2+1, 8096) +'*' as statementTxt ,
            start_time                                                                                       ,
             /*sp.status,*/
            sp.hostname        ,
            blocking_session_id,
             /*r.open_transaction_count, '"'+p_blocking.text + '"' as blocking_StatementTxt, ses.login_name as blocking_login_name, ses.host_name as blocking_host_name, ses.login_time as blocking_login_time, ses.program_name blocking_program_name, */
            r.plan_handle
from
            sys.dm_exec_requests r
            left join dbo.sysprocesses sp on r.session_id = sp.spid
            left join sys.dm_exec_sessions ses on r.blocking_session_id = ses.session_id
            left join sys.dm_exec_connections con on r.blocking_session_id = con.most_recent_session_id
             /* cross apply sys.dm_exec_query_plan(con.most_recent_sql_handle) as qp */
            cross apply sys.dm_exec_sql_text(r.sql_handle) s
            outer apply sys.dm_exec_sql_text(con.most_recent_sql_handle) p_blocking
where
            1 = 1
            and db_name(r.database_id) <> 'master'
order by
            start_time asc
```

## Search entire database

```sql linenums="1"
use database;
DECLARE @SearchStr nvarchar(100) = 'STRING TO FIND'
DECLARE @Results TABLE (ColumnName nvarchar(370), ColumnValue nvarchar(3630))
  
SET NOCOUNT ON
  
DECLARE @TableName nvarchar(256), @ColumnName nvarchar(128), @SearchStr2 nvarchar(110)
SET  @TableName = ''
SET @SearchStr2 = QUOTENAME('%' + @SearchStr + '%','''')
  
WHILE @TableName IS NOT NULL
  
BEGIN
    SET @ColumnName = ''
    SET @TableName =
    (
        SELECT MIN(QUOTENAME(TABLE_SCHEMA) + '.' + QUOTENAME(TABLE_NAME))
        FROM     INFORMATION_SCHEMA.TABLES
        WHERE         TABLE_TYPE = 'BASE TABLE'
            AND    QUOTENAME(TABLE_SCHEMA) + '.' + QUOTENAME(TABLE_NAME) > @TableName
            AND    OBJECTPROPERTY(
                    OBJECT_ID(
                        QUOTENAME(TABLE_SCHEMA) + '.' + QUOTENAME(TABLE_NAME)
                         ), 'IsMSShipped'
                           ) = 0
    )
  
    WHILE (@TableName IS NOT NULL) AND (@ColumnName IS NOT NULL)
  
    BEGIN
        SET @ColumnName =
        (
            SELECT MIN(QUOTENAME(COLUMN_NAME))
            FROM     INFORMATION_SCHEMA.COLUMNS
            WHERE         TABLE_SCHEMA    = PARSENAME(@TableName, 2)
                AND    TABLE_NAME    = PARSENAME(@TableName, 1)
                AND    DATA_TYPE IN ('char', 'varchar', 'nchar', 'nvarchar', 'int', 'decimal')
                AND    QUOTENAME(COLUMN_NAME) > @ColumnName
        )
  
        IF @ColumnName IS NOT NULL
  
        BEGIN
            INSERT INTO @Results
            EXEC
            (
                'SELECT ''' + @TableName + '.' + @ColumnName + ''', LEFT(' + @ColumnName + ', 3630)
                FROM ' + @TableName + ' (NOLOCK) ' +
                ' WHERE ' + @ColumnName + ' LIKE ' + @SearchStr2
            )
        END
    END   
END
  
SELECT ColumnName, ColumnValue FROM @Results
```

## Date Formats

### Standard Date Formats

| **Date Format** | **Standard** | **SQL Statement** | **Sample Output** |
| --------------- | ------------ | ----------------- | ----------------- |
| Mon DD YYYY 1 HH:MIAM (or PM) | Default | SELECT CONVERT(VARCHAR(20), GETDATE(), 100) | Jan 1 2005 1:29PM 1 |
| MM/DD/YY | USA | SELECT CONVERT(VARCHAR(8), GETDATE(), 1) AS [MM/DD/YY] | 11/23/98 |
| MM/DD/YYYY | USA | SELECT CONVERT(VARCHAR(10), GETDATE(), 101) AS [MM/DD/YYYY] | 11/23/1998 |
| YY.MM.DD | ANSI | SELECT CONVERT(VARCHAR(8), GETDATE(), 2) AS [YY.MM.DD] | 72.01.01 |
| YYYY.MM.DD | ANSI | SELECT CONVERT(VARCHAR(10), GETDATE(), 102) AS [YYYY.MM.DD] | 1972.01.01 | 
| DD/MM/YY | British/French | SELECT CONVERT(VARCHAR(8), GETDATE(), 3) AS [DD/MM/YY] | 19/02/72 |
| DD/MM/YYYY | British/French | SELECT CONVERT(VARCHAR(10), GETDATE(), 103) AS [DD/MM/YYYY] | 19/02/1972 |
| DD.MM.YY | German | SELECT CONVERT(VARCHAR(8), GETDATE(), 4) AS [DD.MM.YY] | 25.12.05 |
| DD.MM.YYYY | German | SELECT CONVERT(VARCHAR(10), GETDATE(), 104) AS [DD.MM.YYYY] | 25.12.2005 |
| DD-MM-YY | Italian | SELECT CONVERT(VARCHAR(8), GETDATE(), 5) AS [DD-MM-YY] | 24-01-98 |
| DD-MM-YYYY | Italian | SELECT CONVERT(VARCHAR(10), GETDATE(), 105) AS [DD-MM-YYYY] | 24-01-1998 |
| DD Mon YY 1 | - | SELECT CONVERT(VARCHAR(9), GETDATE(), 6) AS [DD MON YY] | 04 Jul 06 1 | 
| DD Mon YYYY 1	| -	| SELECT CONVERT(VARCHAR(11), GETDATE(), 106) AS [DD MON YYYY] | 04 Jul 2006 1 | 
| Mon DD, YY 1 | - | SELECT CONVERT(VARCHAR(10), GETDATE(), 7) AS [Mon DD, YY] | Jan 24, 98 1 | 
| Mon DD, YYYY 1 | - | SELECT CONVERT(VARCHAR(12), GETDATE(), 107) AS [Mon DD, YYYY] | Jan 24, 1998 1 |
| HH:MM:SS | - | SELECT CONVERT(VARCHAR(8), GETDATE(), 108) | 03:24:53 |
| Mon DD YYYY HH:MI:SS:MMMAM (or PM) 1 | Default + milliseconds | SELECT CONVERT(VARCHAR(26), GETDATE(), 109) | Apr 28 2006 12:32:29:253PM 1 | 
| MM-DD-YY | USA | SELECT CONVERT(VARCHAR(8), GETDATE(), 10) AS [MM-DD-YY] | 01-01-06 |
| MM-DD-YYYY | USA | SELECT CONVERT(VARCHAR(10), GETDATE(), 110) AS [MM-DD-YYYY] | 01-01-2006 | 
| YY/MM/DD | - | SELECT CONVERT(VARCHAR(8), GETDATE(), 11) AS [YY/MM/DD] | 98/11/23 | 
| YYYY/MM/DD | - | SELECT CONVERT(VARCHAR(10), GETDATE(), 111) AS [YYYY/MM/DD] | 1998/11/23 | 
| YYMMDD | ISO | SELECT CONVERT(VARCHAR(6), GETDATE(), 12) AS [YYMMDD] | 980124 |
| YYYYMMDD | ISO | SELECT CONVERT(VARCHAR(8), GETDATE(), 112) AS [YYYYMMDD] | 19980124 | 
| DD Mon YYYY HH:MM:SS:MMM(24h) 1 | Europe default + milliseconds | SELECT CONVERT(VARCHAR(24), GETDATE(), 113) | 28 Apr 2006 00:34:55:190 1 |
| HH:MI:SS:MMM(24H) | - | SELECT CONVERT(VARCHAR(12), GETDATE(), 114) AS [HH:MI:SS:MMM(24H)] | 11:34:23:013 |
| YYYY-MM-DD HH:MI:SS(24h) | ODBC Canonical | SELECT CONVERT(VARCHAR(19), GETDATE(), 120) | 1972-01-01 13:42:24 |
| YYYY-MM-DD HH:MI:SS.MMM(24h) | ODBC Canonical (with milliseconds) | SELECT CONVERT(VARCHAR(23), GETDATE(), 121) | 1972-02-19 06:35:24.489 | 
| YYYY-MM-DDTHH:MM:SS:MMM | ISO8601 | SELECT CONVERT(VARCHAR(23), GETDATE(), 126) | 1998-11-23T11:25:43:250 |
| DD Mon YYYY HH:MI:SS:MMMAM 1 | Kuwaiti | SELECT CONVERT(VARCHAR(26), GETDATE(), 130) | 28 Apr 2006 12:39:32:429AM 1 | 
| DD/MM/YYYY HH:MI:SS:MMMAM | Kuwaiti | SELECT CONVERT(VARCHAR(25), GETDATE(), 131) | 28/04/2006 12:39:32:429AM |

### Extended Date Formats
Here are some more date formats that does not come standard in SQL Server as part of the CONVERT function.

| **Date Format** | **SQL Statement** | **Sample Output** | 
| --------------- | ----------------- | ----------------- |
| YY-MM-DD | SELECT SUBSTRING(CONVERT(VARCHAR(10), GETDATE(), 120), 3, 8) AS [YY-MM-DD]<br>SELECT REPLACE(CONVERT(VARCHAR(8), GETDATE(), 11), '/', '-') AS [YY-MM-DD] | 99-01-24 | 
| YYYY-MM-DD | SELECT CONVERT(VARCHAR(10), GETDATE(), 120) AS [YYYY-MM-DD]<br>SELECT REPLACE(CONVERT(VARCHAR(10), GETDATE(), 111), '/', '-') AS [YYYY-MM-DD] | 1999-01-24 | 
| MM/YY | SELECT RIGHT(CONVERT(VARCHAR(8), GETDATE(), 3), 5) AS [MM/YY]<br>SELECT SUBSTRING(CONVERT(VARCHAR(8), GETDATE(), 3), 4, 5) AS [MM/YY] | 08/99 |
| MM/YYYY | SELECT RIGHT(CONVERT(VARCHAR(10), GETDATE(), 103), 7) AS [MM/YYYY] | 12/2005 |
| YY/MM | SELECT CONVERT(VARCHAR(5), GETDATE(), 11) AS [YY/MM] | 99/08 | 
| YYYY/MM | SELECT CONVERT(VARCHAR(7), GETDATE(), 111) AS [YYYY/MM] | 2005/12 |
| Month DD, YYYY 1 | SELECT DATENAME(MM, GETDATE()) + RIGHT(CONVERT(VARCHAR(12), GETDATE(), 107), 9) AS [Month DD, YYYY] | July 04, 2006 1 |
| Mon YYYY 1 | SELECT SUBSTRING(CONVERT(VARCHAR(11), GETDATE(), 113), 4, 8) AS [Mon YYYY] | Apr 2006 1 |
| Month YYYY 1 | SELECT DATENAME(MM, GETDATE()) + ' ' + CAST(YEAR(GETDATE()) AS VARCHAR(4)) AS [Month YYYY] | February 2006 1 |
| DD Month 1 | SELECT CAST(DAY(GETDATE()) AS VARCHAR(2)) + ' ' + DATENAME(MM, GETDATE()) AS [DD Month] | 11 September 1 | 
| Month DD 1 | SELECT DATENAME(MM, GETDATE()) + ' ' + CAST(DAY(GETDATE()) AS VARCHAR(2)) AS [Month DD] | September 11 1 |
| DD Month YY 1	| SELECT CAST(DAY(GETDATE()) AS VARCHAR(2)) + ' ' + DATENAME(MM, GETDATE()) + ' ' + RIGHT(CAST(YEAR(GETDATE()) AS VARCHAR(4)), 2) AS [DD Month YY] | 19 February 72 1 |
| DD Month YYYY 1 | SELECT CAST(DAY(GETDATE()) AS VARCHAR(2)) + ' ' + DATENAME(MM, GETDATE()) + ' ' + CAST(YEAR(GETDATE()) AS VARCHAR(4)) AS [DD Month YYYY] | 11 September 2002 1 |
| MM-YY | SELECT RIGHT(CONVERT(VARCHAR(8), GETDATE(), 5), 5) AS [MM-YY]<br>SELECT SUBSTRING(CONVERT(VARCHAR(8), GETDATE(), 5), 4, 5) AS [MM-YY] | 12/92 |
| MM-YYYY | SELECT RIGHT(CONVERT(VARCHAR(10), GETDATE(), 105), 7) AS [MM-YYYY] | 05-2006 | 
| YY-MM | SELECT RIGHT(CONVERT(VARCHAR(7), GETDATE(), 120), 5) AS [YY-MM]<br>SELECT SUBSTRING(CONVERT(VARCHAR(10), GETDATE(), 120), 3, 5) AS [YY-MM]| 92/12 |
| YYYY-MM | SELECT CONVERT(VARCHAR(7), GETDATE(), 120) AS [YYYY-MM] | 2006-05 |
| MMDDYY | SELECT REPLACE(CONVERT(VARCHAR(10), GETDATE(), 1), '/', '') AS [MMDDYY] | 122506 |
| MMDDYYYY | SELECT REPLACE(CONVERT(VARCHAR(10), GETDATE(), 101), '/', '') AS [MMDDYYYY] | 12252006 |
| DDMMYY | SELECT REPLACE(CONVERT(VARCHAR(10), GETDATE(), 3), '/', '') AS [DDMMYY] | 240702 |
| DDMMYYYY | SELECT REPLACE(CONVERT(VARCHAR(10), GETDATE(), 103), '/', '') AS [DDMMYYYY] | 24072002 |
| Mon-YY 1 | SELECT REPLACE(RIGHT(CONVERT(VARCHAR(9), GETDATE(), 6), 6), ' ', '-') AS [Mon-YY] | Sep-02 1 |
| Mon-YYYY 1 | SELECT REPLACE(RIGHT(CONVERT(VARCHAR(11), GETDATE(), 106), 8), ' ', '-') AS [Mon-YYYY] | Sep-2002 1 |
| DD-Mon-YY 1 | SELECT REPLACE(CONVERT(VARCHAR(9), GETDATE(), 6), ' ', '-') AS [DD-Mon-YY] | 25-Dec-05 1 |
| DD-Mon-YYYY 1 | SELECT REPLACE(CONVERT(VARCHAR(11), GETDATE(), 106), ' ', '-') AS [DD-Mon-YYYY] | 25-Dec-2005 1 |

## Print all indexes for recreation

````sql linenums="1"
-- Get all existing indexes, but NOT the primary keys
DECLARE cIX CURSOR FOR
   SELECT OBJECT_NAME(SI.Object_ID), SI.Object_ID, SI.Name, SI.Index_ID, SC.name
      FROM Sys.Indexes SI
         LEFT JOIN INFORMATION_SCHEMA.TABLE_CONSTRAINTS TC ON SI.Name = TC.CONSTRAINT_NAME AND OBJECT_NAME(SI.Object_ID) = TC.TABLE_NAME
         LEFT JOIN SYS.tables TB ON (OBJECT_NAME(SI.Object_ID) = TB.name)
         LEFT JOIN SYS.schemas SC ON (TB.schema_id=SC.schema_id)
      WHERE TC.CONSTRAINT_NAME IS NULL
         AND OBJECTPROPERTY(SI.Object_ID, 'IsUserTable') = 1
      ORDER BY OBJECT_NAME(SI.Object_ID), SI.Index_ID
 
DECLARE @IxTable SYSNAME
DECLARE @IxTableID INT
DECLARE @IxName SYSNAME
DECLARE @IxID INT
DECLARE @IxSchema SYSNAME
DECLARE @PKSQL VARCHAR(50)
 
-- Loop through all indexes
OPEN cIX
FETCH NEXT FROM cIX INTO @IxTable, @IxTableID, @IxName, @IxID, @IxSchema
WHILE (@@FETCH_STATUS = 0)
BEGIN
   DECLARE @IXSQL NVARCHAR(4000)
   SET @PKSQL = ''
   SET @IXSQL = 'IF NOT EXISTS(SELECT TOP 1 1 FROM sys.indexes WHERE name=''' + @IxName + ''' AND object_id = OBJECT_ID(''' + @IxSchema + '.' + @IxTable + '''))'
   SET @IXSQL = @IXSQL + CHAR(13) + CHAR(10)
   SET @IXSQL = @IXSQL + 'BEGIN'
   SET @IXSQL = @IXSQL + CHAR(13) + CHAR(10)
   SET @IXSQL = @IXSQL + CHAR(9) + 'CREATE '
 
   -- Check if the index is unique
   IF (INDEXPROPERTY(@IxTableID, @IxName, 'IsUnique') = 1)
      SET @IXSQL = @IXSQL + 'UNIQUE '
   -- Check if the index is clustered
   IF (INDEXPROPERTY(@IxTableID, @IxName, 'IsClustered') = 1)
      SET @IXSQL = @IXSQL + 'CLUSTERED '
 
   SET @IXSQL = @IXSQL + 'INDEX [' + @IxName + '] ON [' + @IxSchema + '].[' + @IxTable + ']('
 
   -- Get all columns of the index
   DECLARE cIxColumn CURSOR FOR
      SELECT SC.Name
      FROM Sys.Index_Columns IC
         JOIN Sys.Columns SC ON IC.Object_ID = SC.Object_ID AND IC.Column_ID = SC.Column_ID
      WHERE IC.Object_ID = @IxTableID AND Index_ID = @IxID
      ORDER BY IC.Index_Column_ID
 
   DECLARE @IxColumn SYSNAME
   DECLARE @IxFirstColumn BIT, @ColumnCount INT
   SET @IxFirstColumn = 1
   SET @ColumnCount = 0
 
   -- Loop throug all columns of the index and append them to the CREATE statement
   OPEN cIxColumn
   FETCH NEXT FROM cIxColumn INTO @IxColumn
   WHILE (@@FETCH_STATUS = 0)
   BEGIN
      IF (@ColumnCount < 16)
      BEGIN
          IF (@IxFirstColumn = 1)
             SET @IxFirstColumn = 0
          ELSE
             SET @IXSQL = @IXSQL + ', '
 
          SET @IXSQL = @IXSQL + '[' + @IxColumn + ']'
          SET @ColumnCount = @ColumnCount + 1
      END
      FETCH NEXT FROM cIxColumn INTO @IxColumn
   END
   CLOSE cIxColumn
   DEALLOCATE cIxColumn
 
   SET @IXSQL = @IXSQL + ')'
 
   SET @IXSQL = @IXSQL + CHAR(13) + CHAR(10)
   SET @IXSQL = @IXSQL + 'END'
   -- Print out the CREATE statement for the index
   IF(LEN(@IXSQL) > 10)
    PRINT @IXSQL
 
   FETCH NEXT FROM cIX INTO @IxTable, @IxTableID, @IxName, @IxID, @IxSchema
END
 
CLOSE cIX
DEALLOCATE cIX
````

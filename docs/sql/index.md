# FleetWave Specific

## Create missing primary keys

``` sql linenums="1" 
SELECT
    SYS.TABLES.NAME,
    CONCAT(N'ALTER TABLE ', SYS.TABLES.NAME, ' ADD PRIMARY KEY CLUSTERED (RECORD_NUMBER_FW ASC);')
FROM
    SYS.TABLES
LEFT OUTER JOIN
    INFORMATION_SCHEMA.KEY_COLUMN_USAGE
        ON SYS.TABLES.NAME = INFORMATION_SCHEMA.KEY_COLUMN_USAGE.TABLE_NAME
WHERE
    OBJECTPROPERTY(OBJECT_ID(CONSTRAINT_SCHEMA + '.' + QUOTENAME(CONSTRAINT_NAME)), 'ISPRIMARYKEY') IS NULL
    AND RIGHT(SYS.TABLES.NAME, 3) = '_FW'
ORDER BY
    1
```

# SQL Server

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

# Oracle
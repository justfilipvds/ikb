# List database sizes

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

# Blocking queries

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
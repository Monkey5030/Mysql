# 捕捉正在执行的查询  
*  方式一
```
SELECT
	pid,--pid
  usename,--执行的用户
	backend_start,--会话开始时间
	query_start,--查询开始时间
	waiting,--是否等待执行
	now() - query_start AS current_query_time,-- 累计执行时间
	now() - backend_start AS current_session_time,
	query,
	STATE,
	state_change,
	client_addr,
	datname 
FROM
	pg_stat_activity 
ORDER BY
	STATE,
	current_query_time DESC
```
*   方式二
```
SELECT     procpid,     start,     now() - start AS lap,     current_query 
FROM     (SELECT         backendid,         pg_stat_get_backend_pid(S.backendid) AS procpid,         pg_stat_get_backend_activity_start(S.backendid) AS start,       pg_stat_get_backend_activity(S.backendid) AS current_query 
    FROM         (SELECT pg_stat_get_backend_idset() AS backendid) AS S     ) AS S WHERE    current_query <> '<IDLE>' ORDER BY    lap DESC;
```
# 终止执行的sql  
`select pg_terminate_backend(48988); --pid`

# 显示表的大小  
`SELECT sotdoid,sotdsize/1024/1024/1024 as sotdsize,sotdtoastsize,sotdadditionalsize,sotdschemaname,sotdtablename from gp_toolkit.gp_size_of_table_disk order by sotdsize desc;`

# 查看行数 
```
SELECT relname, reltuples 
FROM pg_class r JOIN pg_namespace n 
ON (relnamespace = n.oid) 
WHERE relkind = 'r' AND n.nspname = 'calc';
```

# 查看建表时间
`SELECT s.oid,s.relname,t.stausename,t.stasubtype,t.statime FROM pg_class s,pg_stat_last_operation t WHERE  t.objid = s.oid ORDER  BY t.statime`

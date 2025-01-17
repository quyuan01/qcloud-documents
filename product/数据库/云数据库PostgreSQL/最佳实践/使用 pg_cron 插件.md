pg_cron 是一个简单的基于 cron 的 PostgreSQL（10或更高版本）作业调度器，作为扩展在数据库中运行。它使用与常规 cron 相同的语法，但允许您直接从数据库调度 PostgreSQL 命令。

本文为您介绍 PostgreSQL pg_cron 插件的使用方法。
>?云数据库 PostgreSQL 引擎版本12.5及更高版本支持 pg_cron 插件的扩展。

## 启用 pg_cron 扩展
1. 修改与 PostgreSQL 数据库实例关联的参数组，并将 pg_cron 添加到 shared_preload_libraries 参数值中。此更改需要重启 PostgreSQL 数据库实例才能生效。
2. 重新启动 PostgreSQL 数据库实例后，使用具有 superuser 权限的账户运行以下命令：
```
CREATE EXTENSION pg_cron;
```
pg_cron 调度程序是在默认的 PostgreSQL 数据库（postgres）中设置的。这些 pg_cron 对象是在此数据库中创建的，所有调度操作都在此数据库中运行。
3. 您可以使用默认设置，也可以计划作业在 PostgreSQL 数据库实例的其他数据库中运行。要为 PostgreSQL 数据库实例中的其他数据库计划作业，请参见 [为 postgres 以外的数据库计划 cron 作业 中的示例](#wpjhzy)。

## 向 pg_cron 授权
作为 superuser 角色，您可以创建 pg_cron 扩展，然后将权限授予其他用户。为了使其他用户能够计划作业，需向他们授予对 cron 架构中对象的权限。
>!建议您谨慎授予 cron 架构的访问权限。

要向其他人授予 cron 架构的权限，请运行以下命令。
```
postgres=> GRANT USAGE ON SCHEMA cron TO other-user;
```
此权限为 other-user 提供了对 cron 架构的访问权限，以计划和取消计划 cron 作业。但是，为了成功运行 cron 作业，还需要具有对 cron 作业中对象的访问权限。如果用户没有权限，则作业将失败，postgresql.log 中将显示以下错误。
在此示例中，用户没有访问 pgbench_accounts 表的权限。
```
2020-12-08 16:41:00 UTC::@:[30647]:ERROR: permission denied for table pgbench_accounts
2020-12-08 16:41:00 UTC::@:[30647]:STATEMENT: update pgbench_accounts set abalance = abalance + 1
2020-12-08 16:41:00 UTC::@:[27071]:LOG: background worker "pg_cron" (PID 30647) exited with exit code 1
```
cron.job_run_details 表中的其他消息如下所示：
```
postgres=> select jobid, username, status, return_message, start_time from cron.job_run_details where status = 'failed';
jobid |  username  | status |                   return_message                    |          start_time
-------+------------+--------+-----------------------------------------------------+-------------------------------
   143 | unprivuser | failed | ERROR: permission denied for table pgbench_accounts | 2020-12-08 16:41:00.036268+00
   143 | unprivuser | failed | ERROR: permission denied for table pgbench_accounts | 2020-12-08 16:40:00.050844+00
   143 | unprivuser | failed | ERROR: permission denied for table pgbench_accounts | 2020-12-08 16:42:00.175644+00
   143 | unprivuser | failed | ERROR: permission denied for table pgbench_accounts | 2020-12-08 16:43:00.069174+00
   143 | unprivuser | failed | ERROR: permission denied for table pgbench_accounts | 2020-12-08 16:44:00.059466+00
(5 rows)
```

## 计划 pg_cron 作业
以下各节演示如何计划 pg_cron 任务以执行管理任务。
>!创建 pg_cron 任务时，请确保 max_worker_processes 的数量总是大于 cron.max_running_jobs 的数量。如果 pg_cron 任务耗尽后台工作进程，它将失败。pg_cron 任务的原定设置数量为 5；有关更多信息，请参见 [pg_cron 参数](#pgcs)。

### 1. 对表执行清理操作
您可能希望在选择的时间计划对特定表执行清理操作。
以下示例介绍了使用 cron.schedule 函数设置作业，以便每天22:00 (GMT) 在特定表上使用 VACUUM FREEZE。
```
SELECT cron.schedule('manual vacuum', '0 22 * * *', 'VACUUM FREEZE pgbench_accounts');
 schedule
----------
1
(1 row)
```
运行上述示例之后，您可以按如下方式检查 cron.job_run_details 表中的历史记录。
```
postgres=> select * from cron.job_run_details;
 jobid | runid | job_pid | database | username | command | status | return_message | start_time | end_time
-------+-------+---------+----------+----------+----------------------------------------+-----------+----------------+-------------------------------+-------------------------------
 1 | 1 | 3395 | postgres | adminuser| vacuum freeze pgbench_accounts | succeeded | VACUUM | 2020-12-04 21:10:00.050386+00 | 2020-12-04 21:10:00.072028+00
(1 row)
```

### 2. 清除 pg_cron 历史记录表
cron.job_run_details 表包含 cron 作业的历史记录，随着时间的推移，这些历史记录可能会变得非常大。我们建议您计划清除此表的作业。例如，保留一周的条目可能足以进行故障排除。

以下示例使用 cron.schedule 函数计划每天午夜运行以清除 cron.job_run_details 表的作业。这项工作只保留了过去七天的历史记录。使用您的 superuser 账户计划作业，如下所示。
```
SELECT cron.schedule('0 0 * * *', $$DELETE 
    FROM cron.job_run_details 
    WHERE end_time < now() – interval '7 days'$$);
```

### 3. 禁用 pg_cron 历史记录	
要完全禁用向 cron.job_run_details 表中写入任何内容，请修改与 PostgreSQL 数据库实例关联的参数组，然后将 cron.log_run 参数设置为关。
如果您执行此操作，pg_cron 扩展不再写入表，只会在 postgresql.log 文件中生成错误。

使用以下命令检查 cron.log_run 参数的值。
```
postgres=> SHOW cron.log_run;
```

## [为 postgres 以外的数据库计划 cron 作业](id:wpjhzy)
pg_cron 的元数据全部保存在名为 postgres 的 PostgreSQL 默认数据库中。由于后台工件用于运行维护 cron 作业，因此您可以在 PostgreSQL 数据库实例中的任何数据库中计划作业。

1. 在 cron 数据库中，以与平常使用 cron.schedule 相同的方式计划作业。
```
postgres=> SELECT cron.schedule('database1 manual vacuum', '29 03 * * *', 'vacuum freeze test_table');
```
2. 作为具有 superuser 角色的用户，请更新刚创建的作业的数据库列，使其在 PostgreSQL 数据库实例中的另一个数据库中运行。
```
postgres=> UPDATE cron.job SET database = 'database1' WHERE jobid = 106;
```
3. 通过查询 cron.job 表进行验证。
```
postgres=> select * from cron.job;
 jobid | schedule | command | nodename | nodeport | database | username | active | jobname
-------+-------------+----------------------------------------+-----------+----------+-----------+-----------+--------+-------------------------
 106 | 29 03 * * * | vacuum freeze test_table | localhost | 8192 | database1 | adminuser | t | database1 manual vacuum
 1 | 59 23 * * * | vacuum freeze pgbench_accounts | localhost | 8192 | postgres | adminuser | t | manual vacuum
(2 rows)
```
>!在某些情况下，您可以添加打算在其他数据库上运行的 cron 作业。在这些情况下，在您更新正确的数据库列之前，该作业可能会尝试在默认数据库 (postgres) 中运行。如果用户名具有权限，则作业将在默认数据库中成功运行。

## [pg_cron 参数](id:pgcs)
以下是用于控制 pg_cron 扩展行为的参数列表。

| 参数 | 描述 | 
|---------|---------|
| cron.database_name |保存 pg_cron 元数据的数据库。 | 
|cron.host |要连接到 PostgreSQL 的主机名。您无法修改此值。 | 
| cron.log_run |将运行的所有作业记录到 job_run_details 表中。值为 on 或 off。 | 
| cron.log_statement |在运行所有 cron 语句之前将其记入日志。值为 on 或 off。 | 
| cron.max_running_jobs |可以同时运行的最大作业数。 | 
| cron.use_background_workers	|使用后台工作程序而不是客户端会话。您无法修改此值。 | 

使用以下 SQL 命令来显示这些参数及其值。
```
postgres=> SELECT name, setting, short_desc FROM pg_settings WHERE name LIKE 'cron.%' ORDER BY name;
```

## cron.schedule() 函数
此函数计划 cron 作业。作业最初是在默认 postgres 数据库中计划的。该函数返回一个表示作业标识符的 bigint 值。要计划作业在 PostgreSQL 数据库实例的其他数据库中运行，请参阅 [为 postgres 以外的数据库计划 cron 作业](#wpjhzy) 中的示例。
该函数有两种语法格式。

**语法**
```
cron.schedule (job_name,
    schedule,
    command
);

cron.schedule (schedule,
    command
);
```

**参数**

| 参数 | 描述 | 
|---------|---------|
| job_name | cron 作业的名字。 | 
| schedule | 表示 cron 作业时间表的文本。格式是标准 cron 格式。 | 
| command | 要运行的命令的文本。 | 

**示例**
```
postgres=> SELECT cron.schedule ('test','0 10 * * *', 'VACUUM pgbench_history');
 schedule
----------
      145
(1 row)

postgres=> SELECT cron.schedule ('0 15 * * *', 'VACUUM pgbench_accounts');
 schedule
----------
      146
(1 row)
```

## cron.unschedule() 函数
此函数删除 cron 作业。您可以传入 job_name 或 job_id。策略可以确保您是删除作业计划的拥有者。该函数返回一个布尔值，指示成功或失败。

该函数使用以下语法格式。
**语法**
```
cron.unschedule (job_id);

cron.unschedule (job_name);
```

**参数**

| 参数 | 描述 | 
|---------|---------|
| job_id | 计划 cron 作业时从 cron.schedule 函数返回的作业标识符。 | 
| job_name | 使用该 cron.schedule 函数计划的 cron 作业的名称。 | 

**示例**
```
postgres=> select cron.unschedule(108);
 unschedule
------------
 t
(1 row)

postgres=> select cron.unschedule('test');
 unschedule
------------
 t
(1 row)
```

## pg_cron 表
将以下各表用于计划 cron 作业和记录作业完成的方式。

| 表 | 描述 | 
|---------|---------|
| cron.job | 包含有关每个计划作业的元数据。与此表的大多数交互应使用 cron.schedule 和 cron.unschedule 函数完成。<br>注意：不建议直接向此表授予更新或插入权限。这样做将允许用户更新 username 列，从而以 superuser 身份运行。 |
| cron.job_run_details | 包含过去运行的计划作业的历史信息。这对于调查运行的作业的状态、返回消息以及开始和结束时间非常有用。<br>为了防止此表无限增长，请定期清除此表。 |

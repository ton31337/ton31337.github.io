---
layout: post
title: Using `show-databases` on heavy mysql instance
categories:
- blog
---

In shared hosting world, we have a little bit different workload, where we hit weird limits. This is not an exception. We run MySQL instances with more than 300k databases in a single server, so you might guess what I'm talking about. When you type `SHOW DATABASES` on such server, you are a bit surprised, it takes ~15s to return. Wondering why? Keep reading.

```
Samples: 15K of event 'cycles', Event count (approx.): 41470603374
  Children      Self  Command          Shared Object                Symbol
+   97.89%     0.00%  mysqld           mysqld                       [.] mysql_execute_command
+   97.89%     0.00%  mysqld           mysqld                       [.] execute_sqlcom_select
+   97.89%     0.00%  mysqld           mysqld                       [.] handle_select
+   97.89%     0.00%  mysqld           mysqld                       [.] mysql_select
+   97.89%     0.00%  mysqld           mysqld                       [.] JOIN::exec
+   97.89%     0.00%  mysqld           mysqld                       [.] JOIN::exec_inner
+   97.89%     0.00%  mysqld           mysqld                       [.] get_schema_tables_result
+   97.88%     0.03%  mysqld           mysqld                       [.] fill_schema_schemata
+   72.06%    71.89%  mysqld           libc-2.17.so                 [.] __strcmp_sse42
+   19.57%    18.73%  mysqld           mysqld                       [.] acl_get
+    5.77%     5.77%  mysqld           mysqld                       [.] strcmp@plt
```

Take a look at `acl_get()`. It checks rights for every database, so it takes a huge amount of time to consume everything.

```
global elapsed;
probe process("/usr/sbin/mysqld").function("acl_get").return
{
    elapsed += gettimeofday_ms() - @entry(gettimeofday_ms());
}
probe end { printf("%d ms\n", elapsed); }
```

```
16052 ms (~16s).
```

This proves, that `acl_get()` is the winner for this task.

## Summary

* `root` user has all rights, thus it doesn't check for `acl_get()`. Regular user has strict rights, thus it takes to return on every table;
* To work around this problem use `mysql-proxy` or `ProxySQL` to rewrite SQL queries like `SELECT SCHEMA_NAME AS "Database" FROM INFORMATION_SCHEMA.SCHEMATA WHERE SCHEMA_NAME LIKE CONCAT(SUBSTRING_INDEX(CURRENT_USER(), "_", 1),"%")`.

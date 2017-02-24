---
layout: post
title: ProxySQL and deprecated COM_PROCESS_KILL
categories:
- blog
---

Today we solved an issue where ProxySQL just asserts if it receives MySQL command which is not handled by ProxySQL. So, `COM_PROCESS_KILL` was an applicant. Here is the backtrace:

```
(gdb) bt
#0  0x00007f3601ee75f7 in raise () from /lib64/libc.so.6
#1  0x00007f3601ee8ce8 in abort () from /lib64/libc.so.6
#2  0x00007f3601ee0566 in __assert_fail_base () from /lib64/libc.so.6
#3  0x00007f3601ee0612 in __assert_fail () from /lib64/libc.so.6
#4  0x0000000000466242 in MySQL_Session::handler (this=this@entry=0x7f35f8631000) at MySQL_Session.cpp:1745
#5  0x0000000000454dc4 in MySQL_Thread::process_all_sessions (this=this@entry=0x7f35f860d000) at MySQL_Thread.cpp:2678
#6  0x000000000045cfd1 in MySQL_Thread::run (this=this@entry=0x7f35f860d000) at MySQL_Thread.cpp:2518
#7  0x0000000000438f24 in mysql_worker_thread_func (arg=0x7f360043c9b0) at main.cpp:165
#8  0x00007f3603712dc5 in start_thread () from /lib64/libpthread.so.0
#9  0x00007f3601fa8ced in clone () from /lib64/libc.so.6

(gdb) x/32b pkt->ptr
0x7fddcd819660:    5    0    0    0    12    -61    0    0
0x7fddcd819668:    0    0    0    0    4    0    0    0
0x7fddcd819670:    -128    115    -81    -52    -35    127    0    0
0x7fddcd819678:    0    0    0    0    0    0    0    0
(gdb) p pkt
$4 = {ptr = 0x7fddcd819660, size = 9}
```

First bytes are mysql header, `12` is the mysql command, which in this case is https://dev.mysql.com/doc/internals/en/com-process-kill.html.

`12` = `0x0c`, so the first 4 bytes is a mysql header (`5    0    0    0`) that also specifies that the length of the payload is 5 bytes. 5 bytes payload + 4 bytes header = 9 bytes in total. So the first byte of the payload identifies the type of command.

This behavior by ProxySQL was [fixed](https://github.com/sysown/proxysql/commit/42ca44c91781d9cd9a4bda265bbd8ee68bca41bf).

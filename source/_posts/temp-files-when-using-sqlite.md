---
title: 使用SQLite/Room时产生的中间文件是什么
date: 2021-10-17 11:08:05
tags:
- Android
- SQLite
- Room
---

在学习Room时发现，当创建一个名为`book_database`的数据库文件后，同时会产生两个带“后缀”的文件：`book_database-shm`和`book_database-wal`。对此比较好奇：这两个文件时什么？为什么产生？

<!-- more -->

认为它们是临时文件的原因是注意到它们的名字前缀（`book_database`）是相同的，而且恰好是在代码中指明的数据库的名字。



Google一番，在StackOverFlow上找到这么一篇QA：https://stackoverflow.com/questions/7778723/what-are-the-db-shm-and-db-wal-extensions-in-sqlite-databases。里面给出了官网的介绍链接：https://www.sqlite.org/wal.html。

* `shm`是共享内存文件，里面仅包含了临时数据。
* `wal`是为了支持回滚事务产生的临时文件。例如，当事务执行失败时，SQLite能通过它回滚修改，确保数据的正确性。


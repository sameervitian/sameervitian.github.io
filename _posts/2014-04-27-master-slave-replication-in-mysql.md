---
layout: post
title: "Master-Slave Replication in MySQL"
description: ""
categories: ["technology", "mysql"]
tags: [mysql, db, database, master-slave setup]

---
{% include JB/setup %}

## Overview

 MySQL replication becomes necessary if your application drives a huge traffic and there is lots of interaction with database. It is but obvious that with increasing traffic the CPU usage and memory consumption of the machine also increases. We can offload the traffic if there is more than one database machine.
 The main database is called master database. 
 All the other databases which replicate the db data of master are called slave database.
 This slave database helps in many other ways including facilitating database backup, directing all read statements to slave machine. It is necesssary that all the writes to be done on master database which inturn will reflect on slave database.



##Master/Slave Replication Setup
  

### Configuring Master DB

  Binary logs created on master are fetched by slave. These binary logs contains all the details of the write operation done on master. All these write operation are performed on slave machine. Master is the producer for these bin logs. Slave acts as consumer.
  On master db we have to enable bin logs and assign a server-id to it. 
  Following lines needs to be added in mysqld block of mysql conf file (usually the file path is /etc/my.cnf).

    [mysqld]
    log-bin = /var/log/mysql/mysql-bin.log
    server_id = 1

  After making these changes one have to restart the mysql server.
  Now we need to create replication user on master by issing following command 

    mysql> GRANT REPLICATION SLAVE ON *.* TO 'replicationuser'@'slave_ipaddress' IDENTIFIED BY 'pass';
    mysql> FLUSH PRIVILEGES;

### Configuring Slave DB
  
  Slave db is also assigned with a unique server-id.
    
    [mysqld]
    server-id=2
  Now the master DB is stopped or locked over write statement and the position of bin log is read.
  To know the bin log coordinates issue the command below on master.

    mysql> SHOW MASTER STATUS;

    +------------------+----------+--------------+------------------+-------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+--------------+------------------+-------------------+
    | mysql-bin.00032 | 29359487 |              |                  |                   |
    +------------------+----------+--------------+------------------+-------------------+

  The file and postion value is used to start replication on slave.
  After issing the command we need to take db snapshot and load on slave. We can restart the slave after this is done.
  Now we need to issue `change master` command on slave db session.

    mysql> CHANGE MASTER TO MASTER_HOST='master_ipaddress', MASTER_USER='replicationuser', MASTER_PASSWORD='pass', MASTER_LOG_FILE='mysql-bin.00032', MASTER_LOG_POS=29359487; 
    mysql> START SLAVE;

  START SLAVE command starts the replication.

  To know the slave status following command is to be issued-

    mysql> SHOW SLAVE STATUS;

  Following command is to be issued to pause the replication.

    mysql> STOP SLAVE;

  To resume replication following command is used - 

    mysql> START SLAVE;


  















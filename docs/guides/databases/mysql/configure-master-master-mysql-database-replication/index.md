---
slug: configure-master-master-mysql-database-replication
author:
  name: James Stewart
  email: jstewart@linode.com
description: "Learn how to set up master-master MySQL databases replication in this simple step-by-step tutorial."
og_description: "MySQL Master-Master replication adds speed and redundancy. With replication, two separate MySQL servers act as a cluster, particularly useful for high availability website configurations. Use this guide to configure database replication on your Linode."
keywords: ["set up mysql", "replication", "master-master", "high availability"]
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
aliases: ['/databases/mysql/mysql-master-master-replication/','/databases/mysql/backup-options/','/databases/mysql/mysql-master-master/','/databases/mysql/configure-master-master-mysql-database-replication/']
modified: 2021-10-18
modified_by:
  name: Linode
published: 2014-12-24
title: "Configure Master-Master MySQL Database Replication"
external_resources:
 - '[MySQL Reference Manuals](http://dev.mysql.com/doc/)'
tags: ["ubuntu","debian","database","mysql"]
image: mysql-master-master-replication-title.jpg
---

![Configure Master-Master MySQL Database Replication](mysql-master-master-replication-title.jpg)

## What is MySQL Master-Master Replication?

MySQL Master-Master replication adds speed and redundancy for active websites. With replication, two separate MySQL servers act as a cluster. Database clustering is particularly useful for high availability website configurations. Use two separate Linodes to configure database replication, each with private IPv4 addresses.

{{< note >}}
This guide is written for a non-root user. Commands that require elevated privileges are prefixed with `sudo`. If you're not familiar with the `sudo` command, you can check our [Users and Groups](/docs/tools-reference/linux-users-and-groups) guide.

This guide is written for Debian 9 or Ubuntu 18.04.
{{< /note >}}

## Install MySQL

1.  Use the following commands to install MySQL on each of the Linodes:

        sudo apt-get update
        sudo apt-get upgrade -y
        sudo apt-get install mysql-server mysql-client

2.  Run the MySQL secure installation command. You will be asked to create a root password. It is recommended you select yes to all of the questions:

        mysql_secure_installation

## Edit MySQL's Configuration

1.  Edit the `/etc/mysql/my.cnf` file on each of the Linodes. Add or modify the following values:

    **Server 1:**

    {{< file "/etc/mysql/my.cnf" >}}
[mysqld]
server_id           = 1
log_bin             = /var/log/mysql/mysql-bin.log
log_bin_index       = /var/log/mysql/mysql-bin.log.index
relay_log           = /var/log/mysql/mysql-relay-bin
relay_log_index     = /var/log/mysql/mysql-relay-bin.index
expire_logs_days    = 10
max_binlog_size     = 100M
log_replica_updates = 1
auto-increment-increment = 2
auto-increment-offset = 1
{{< /file >}}

    If using MySQL 8.0.25 or earlier, replace `log_replica_updates` with `log_slave_updates` (within both Servers 1 and 2). See [MySQL documentation](https://dev.mysql.com/doc/refman/8.0/en/replication-options-binary-log.html#sysvar_log_slave_updates) for details.

    **Server 2:**

    {{< file "/etc/mysql/my.cnf" >}}
[mysqld]
server_id           = 2
log_bin             = /var/log/mysql/mysql-bin.log
log_bin_index       = /var/log/mysql/mysql-bin.log.index
relay_log           = /var/log/mysql/mysql-relay-bin
relay_log_index     = /var/log/mysql/mysql-relay-bin.index
expire_logs_days    = 10
max_binlog_size     = 100M
log_replica_updates = 1
auto-increment-increment = 2
auto-increment-offset = 2
{{< /file >}}

2. Edit the `bind-address` configuration in order to use the private IP addresses, for each of the Linodes.

    {{< file "/etc/mysql/my.cnf" >}}
bind-address    = x.x.x.x
{{< /file >}}

3.  Once completed, restart the MySQL application:

        sudo systemctl restart mysql

## Create Replication Users

1.  Log in to MySQL on each of the Linodes:

        mysql -u root -p

2.  Configure the replication users on each Linode. Replace `x.x.x.x` with the private IP address of the opposing Linode, and `password` with a strong password:

        GRANT REPLICATION SLAVE ON *.* TO 'replication'@'x.x.x.x' IDENTIFIED BY 'password';

3.  Run the following command to test the configuration. Use the private IP address of the opposing Linode:

        mysql -u replication -p -h x.x.x.x -P 3306

    This command should connect you to the remote server's MySQL instance.

## Configure Database Replication

1.  While logged into MySQL on Server 1, query the master status:

        SHOW MASTER STATUS;

    Note the file and position values that are displayed:

        mysql> SHOW MASTER STATUS;
        +------------------+----------+--------------+------------------+
        | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
        +------------------+----------+--------------+------------------+
        | mysql-bin.000001 |      277 |              |                  |
        +------------------+----------+--------------+------------------+
        1 row in set (0.00 sec)

2.  On Server 2 at the MySQL prompt, set up the replica functionality for that database. Replace`x.x.x.x` with the private IP from the first server. Also replace the value for `source_log_file` with the file value from the previous step, and the value for `source_log_pos` with the position value.

        STOP REPLICA;
        CHANGE REPLICATION SOURCE TO
            source_host='x.x.x.x',
            source_port=3306,
            source_user='replication',
            source_password='password',
            source_log_file='mysql-bin.000001',
            source_log_pos=106;
        START REPLICA;

    If you're using MySQL 8.0.22 or earlier, use the following statements instead, and replace the value for `master_log_file` with the file value from the previous step, and the value for `master_log_pos` with the position value. See the [MySQL documentation](https://dev.mysql.com/doc/refman/8.0/en/change-master-to.html) for details.

        STOP SLAVE;
        CHANGE MASTER TO
            master_host='x.x.x.x',
            master_port=3306,
            master_user='replication',
            master_password='password',
            master_log_file='mysql-bin.000001',
            master_log_pos=106;
        START SLAVE;

3.  On Server 2, query the master status. Again note the file and position values.

        SHOW MASTER STATUS;

4.  Set the replica database status on Server 1, utilizing similar commands as in step 2. When entering the commands, use the IP address of Server 2 and the file and position values you just collected in the previous step.

5.  Test by creating a database and inserting a row:

    **Server 1:**

        create database test;
        create table test.flowers (`id` varchar(10));

    **Server 2:**

        show tables in test;

When queried, you should see the tables from Server 1 replicated on Server 2.  Congratulations, you now have a MySQL Master-Master cluster!


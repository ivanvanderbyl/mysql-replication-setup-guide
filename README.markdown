My notes on setting up MySQL database replication on two servers (master/slave)
===============================================================================

This guide assumes you have two server instances, we're using two AussieHQ Dedicated Server Environments (DSE's) (Their fancy name for VMWare vCenter instances), configured as per my [Rails, Ubuntu, Nginx stack guide](http://github.com/ivanvanderbyl/rails-nginx-passenger-ubuntu)
with the same MySQL root password on both machines.

In this article, I will refer to these servers as having the following private network IPs:

* server1: 10.0.0.10
* server2: 10.0.0.20  

Configuring The Master
----------------------

Select which machine will run the master database.

So make sure that the replication can work, we must unbind from the local address and listen on all interfaces

    sudo vi /etc/mysql/my.cnf

Edit the my.cnf file and comment the line *bind-address = 127.0.0.1*

    [...]
    #
    # Instead of skip-networking the default is now to listen only on
    # localhost which is more compatible and is not less secure.
    # bind-address    = 127.0.0.1
    [...]

Restart MySQL afterwards

    sudo restart mysql
    
Then check it is bound to all interfaces

    netstat -tap | grep mysql

You should see a line like this:
    
    deploy@server1:~$ netstat -tap | grep mysql
    tcp        0      0 *:mysql      *:*                     LISTEN      -
    

Setup replication user
----------------------

Now we set up a replication user slave_user that can be used by *server2* to access the MySQL database on *server1*:

    mysql -u root -p
    
In the MySQL shell, run the following commands:

    GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'server2' IDENTIFIED BY 'slave_password';

Configure master for replication
--
    sudo vi /etc/mysql/my.cnf
    
Uncomment the lines indicated below, this tells MySQL for which database it should write logs (these logs are used by the slave to see what has changed on the master), which log file it should use, and we have to specify that this MySQL server is the master. We want to replicate the database *exampledb*
    
    [...]
    # The following can be used as easy to replay backup logs or for replication.
    # note: if you are setting up a replication slave, see README.Debian about
    #       other settings you may need to change.
    server-id               = 1
    log_bin                 = /var/log/mysql/mysql-bin.log
    expire_logs_days        = 10
    max_binlog_size         = 100M
    binlog_do_db            = exampledb
    [...]
    
Restart MySQL:

    sudo restart mysql
    
Next we lock the exampledb database on server1, find out about the master status of server1, create an SQL dump of exampledb (that we will import into exampledb on server2 so that both databases contain the same data), and unlock the database so that it can be used again:

    mysql -u root -p
    
On the MySQL shell, run the following commands:

    USE exampledb;
    FLUSH TABLES WITH READ LOCK;
    SHOW MASTER STATUS;
    
The last command should show something like this (please write it down, we'll need it later on):

    mysql> SHOW MASTER STATUS;
    +------------------+----------+--------------+------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
    +------------------+----------+--------------+------------------+
    | mysql-bin.000001 |  3096424 | exampledb    |                  |
    +------------------+----------+--------------+------------------+
    1 row in set (0.00 sec)

    mysql>
    
Now don't leave the MySQL shell, because if you leave it, the database lock will be removed, and this is not what we want right now because we must create a database dump now. While the MySQL shell is still open, we open a second command line window where we create the SQL dump snapshot.sql and transfer it to server2 (using scp; again, make sure that the root account is enabled on server2):

On *server1*:
    cd /tmp
    mysqldump -u root -pyourrootsqlpassword --opt exampledb > snapshot.sql
    scp snapshot.sql deploy@server2:

Afterwards, you can close the second command line window. On the first command line window, we can now unlock the database and leave the MySQL shell:

    UNLOCK TABLES; 
    quit;

Configuring The Slave
=====================

Now we must configure the slave. Open /etc/mysql/my.cnf and make sure you have the following settings in the [mysqld] section:
    
    sudo vi /etc/mysql/my.cnf
    
    [...]
    server-id = 2
    master-connect-retry=60
    replicate-do-db=exampledb
    [...]

The value of server-id must be unique and thus different from the one on the master!

Restart MySQL:

    sudo restart mysql
    
Before we start setting up the replication, we create an empty database exampledb on server2:

    mysql -u root -p

    CREATE DATABASE exampledb;
    quit;
    
On server2, we can now import the SQL dump snapshot.sql like this:

    mysqladmin --user=root --password=yourrootsqlpassword stop-slave
    mysql -u root -pyourrootsqlpassword exampledb < ~/snapshot.sql

Now connect to MySQL again...

    mysql -u root -p

and run the following command to make server2 a slave of server1 (it is important that you replace the values in the following command with the values you got from the SHOW MASTER STATUS; command that we ran on server1!):
    
    CHANGE MASTER TO MASTER_HOST='10.0.0.10', MASTER_USER='slave_user', MASTER_PASSWORD='slave_password', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=4290;

Available options are:
    **MASTER_HOST** is the IP address or hostname of the master (in this example it is 192.168.0.100).
    **MASTER_USER** is the user we granted replication privileges on the master.
    **MASTER_PASSWORD** is the password of MASTER_USER on the master.
    **MASTER_LOG_FILE** is the file MySQL gave back when you ran SHOW MASTER STATUS; on the master.
    **MASTER_LOG_POS** is the position MySQL gave back when you ran SHOW MASTER STATUS; on the master.
    **MASTER_SSL** makes the slave use an SSL connection to the master.
    **MASTER_SSL_CA** is the path to the ca-cert.pem file on the slave.
    **MASTER_SSL_CERT** is the path to the client-cert.pem file on the slave.
    **MASTER_SSL_KEY** is the path to the client-key.pem file on the slave.
  
Finally start the slave:

    START SLAVE;

Then check the slave status:

    SHOW SLAVE STATUS \G
  
It is important that both *Slave_IO_Running* and *Slave_SQL_Running* have the value Yes in the output (otherwise something went wrong, and you should check your setup again and take a look at /var/log/syslog to find out about any errors):

    mysql> SHOW SLAVE STATUS \G
    *************************** 1. row ***************************
                   Slave_IO_State: Waiting for master to send event
                      Master_Host: 10.0.0.10
                      Master_User: slave_user
                      Master_Port: 3306
                    Connect_Retry: 60
                  Master_Log_File: mysql-bin.000001
              Read_Master_Log_Pos: 3096424
                   Relay_Log_File: mysqld-relay-bin.000002
                    Relay_Log_Pos: 251
            Relay_Master_Log_File: mysql-bin.000001
                 Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
                  Replicate_Do_DB: exampledb
              Replicate_Ignore_DB:
               Replicate_Do_Table:
           Replicate_Ignore_Table:
          Replicate_Wild_Do_Table:
      Replicate_Wild_Ignore_Table:
                       Last_Errno: 0
                       Last_Error:
                     Skip_Counter: 0
              Exec_Master_Log_Pos: 3096424
                  Relay_Log_Space: 407
                  Until_Condition: None
                   Until_Log_File:
                    Until_Log_Pos: 0
               Master_SSL_Allowed: Yes
               Master_SSL_CA_File:
               Master_SSL_CA_Path:
                  Master_SSL_Cert:
                Master_SSL_Cipher:
                   Master_SSL_Key:
            Seconds_Behind_Master: 0
    Master_SSL_Verify_Server_Cert: No
                    Last_IO_Errno: 0
                    Last_IO_Error:
                   Last_SQL_Errno: 0
                   Last_SQL_Error:
    1 row in set (0.00 sec)

    mysql>

Afterwards, you can leave the MySQL shell on server2:

    quit;

That's it! Now whenever exampledb is updated on the master, all changes will be replicated to exampledb on the slave. Test it!

References
==========

* http://www.howtoforge.com/how-to-set-up-mysql-database-replication-with-ssl-encryption-on-ubuntu-9.10-p2

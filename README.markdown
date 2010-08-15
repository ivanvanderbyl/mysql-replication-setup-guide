My notes on setting up MySQL database replication on two servers (master/slave)
============================

This guide assumes you have two server instances, we're using two AussieHQ Dedicated Server Environments (DSE's) (Their fancy name for VMWare vCenter instances), configured as per my [Rails, Ubuntu, Nginx stack guide](http://github.com/ivanvanderbyl/rails-nginx-passenger-ubuntu)
with the same MySQL root password on both machines.

In this article, I will refer to these servers as having the following private network IPs:
server1: 10.0.0.10
server2: 10.0.0.20

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
--

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
    

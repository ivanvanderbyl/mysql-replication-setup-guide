My notes on setting up MySQL database replication on two servers (master/slave)
============================

This guide assumes you have two server instances, we're using two AussieHQ Dedicated Server Environments (DSE's) (Their fancy name for VMWare vCenter instances), configured as per my [Rails, Ubuntu, Nginx stack guide](http://github.com/ivanvanderbyl/rails-nginx-passenger-ubuntu)
with the same MySQL root password on both machines.

Configuring The Master
----------------------

Select which machine will run the master database.

So make sure that the replication can work, we must first make MySQL listen on all interfaces.

    sudo vi /etc/mysql/my.cnf

Edit the my.cnf file and comment out the line bind-address = 127.0.0.1

    [..]
    #
    # Instead of skip-networking the default is now to listen only on
    # localhost which is more compatible and is not less secure.
    # bind-address    = 127.0.0.1
    [..]

Restart MySQL afterwards

    sudo restart mysql
    
Then check it is bound to all interfaces

    netstat -tap | grep mysql

You should see a line like this:
    
    deploy@server1:~$ netstat -tap | grep mysql
    tcp        0      0 *:mysql                 *:*                     LISTEN      -
    

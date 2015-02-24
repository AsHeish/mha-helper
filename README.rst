===============================
mha_helper
===============================

.. image:: https://img.shields.io/travis/ovaistariq/mha_helper.svg
        :target: https://travis-ci.org/ovaistariq/mha_helper

.. image:: https://img.shields.io/pypi/v/mha_helper.svg
        :target: https://pypi.python.org/pypi/mha_helper


MHA helper (mha-helper) is a set of helper scripts that supplement in doing proper failover using MHA (https://code.google.com/p/mysql-master-ha/). MHA is responsible for executing the important failover steps such as finding the most recent slave to failover to, applying differential logs, monitoring master for failure, etc. But it does not deal with additional steps that need to be taken before and after failover. These would include steps such as setting the read-only flag, killing connections, moving writer virtual IP, etc. Furthermore, the monitor that does the monitoring of masters to test for failure is not daemonized and exits after performing the failover which might not be intended, because of course we need the monitor to keep monitoring even after failover.

* Documentation: https://mha-helper.readthedocs.org.

MHA Helper
==========
There are three functions of mha-helper:

1. Execute pre-failover and post-failover steps during an online failover. An online failover is one in which the original master is not dead and the failover is performed for example for maintenance purposes.
2. Execute pre-failover and post-failover steps during master failover. In this case the original master is dead, meaning either the host is dead or the MySQL server is dead.
3. Daemonize the monitor that monitors the masters for failure.

Package Requirements and Dependencies
=====================================
mha-helper is written using Python 2.6 so if you have an older version of Python running you must upgrade to Python 2.6 or change the shebang line to point to the appropriate python 2.6 binary.

In addition to Python 2.6, you would need the following packages installed:
+ **MHA** - Of course you need the MHA manager and node packages installed. You can read more about installing MHA and its dependencies here: http://code.google.com/p/mysql-master-ha/wiki/Installation
+ **MySQL-python** on RHEL 6.0 and above, or **python26-mysqldb** on RHEL versions less than 6.0, or **python-mysqldb** on Debian6 / Ubuntu Lucid and above

In addition there are other MHA specific requirements, please go through the link below to read about them:
https://code.google.com/p/mysql-master-ha/wiki/Requirements

Configuration
=============
mha-helper uses ini-style configuration files.

There are two configuration files needed by mha-helper, one of them is the mha-helper specific global configuration file named **global.conf** and the other is the MHA specific application configuration file. Currently mha-helper always assumes that the global configuration file is available in the conf directory inside the mha-helper directory. So if you have mha-helper at the location /usr/local/mha-helper, then the global configuration file will be available at /usr/local/mha-helper/conf/global.conf

The **global configuration** file has a section named **'default'** and also has other sections named after the hostnames of the master and slave servers that are being managed by MHA. Moreover the options defined in the host sections override the options defined in the default section.

mha-helper supports the following options in the **global configuration** file that can be specified in the **'default'** section:

+ requires_sudo
+ manage_vip
+ writer_vip_cidr
+ writer_vip
+ cluster_interface
+ report_email


All the options above can also be specified in the host specific sections and they will override the values of options defined in the 'default' section. The host specific section has one additional option:

+ cluster_conf_path


Note that you must have separate sections defined for each of the master-slave servers that MHA is managing.


Let me show you an example global configuration file:

---
    [default]
    requires_sudo       = yes
    manage_vip          = yes
    writer_vip_cidr     = 192.168.1.155/24
    writer_vip          = 192.168.1.155
    cluster_interface   = eth0
    report_email        = ovaistariq@gmail.com

    [db1]
    cluster_conf_path   = /usr/local/mha-helper/conf/test_cluster.conf.sample

    [db2]
    cluster_conf_path   = /usr/local/mha-helper/conf/test_cluster.conf.sample

    [db3]
    cluster_conf_path   = /usr/local/mha-helper/conf/test_cluster.conf.sample

    [db4]
    cluster_conf_path   = /usr/local/mha-helper/conf/test_cluster.conf.sample
---

Note that this **global configuration** file is specific to mha-helper and is different from the global configuration file specific to MHA.

Apart from the global configuration file is the MHA specific application configuration file which basically defines the master-slave hosts.

Please read the content at this link to see how the application configuration file should be written:
https://code.google.com/p/mysql-master-ha/wiki/Configuration#Writing_an_application_configuration_file

I would also suggesting that you go through this URL to see all the available MHA configuration options:
https://code.google.com/p/mysql-master-ha/wiki/Parameters


Following are the important options that must be specified in the MHA application configuration file:

+ user
+ password
+ ssh_user
+ manager_workdir
+ manager_log
+ master_ip_failover_script       = /usr/local/mha-helper/scripts/master_ip_failover
+ master_ip_online_change_script  = /usr/local/mha-helper/scripts/master_ip_online_change
+ report_script                   = /usr/local/mha-helper/scripts/failover_report


Let me show you an example application configuration file:

---
    [server default]
    user                            = mha_helper
    password                        = helper
    ssh_user                        = mha_helper
    ssh_options                     = '-i /home/mha_helper/.ssh/id_rsa'
    ssh_port                        = 2202
    repl_user                       = repl
    repl_password                   = repl
    master_binlog_dir               = /var/lib/mysql
    manager_workdir                 = /var/log/mha/test_cluster
    manager_log                     = /var/log/mha/test_cluster/test_cluster.log
    remote_workdir                  = /var/log/mha/test_cluster
    master_ip_failover_script       = /usr/local/mha-helper/scripts/master_ip_failover
    master_ip_online_change_script  = /usr/local/mha-helper/scripts/master_ip_online_change
    report_script                   = /usr/local/mha-helper/scripts/failover_report

    [server1]
    hostname            = db1
    candidate_master    = 1
    check_repl_delay    = 0

    [server2]
    hostname            = db2
    candidate_master    = 1
    check_repl_delay    = 0

    [server3]
    hostname            = db3
    no_master           = 1

    [server4]
    hostname            = db4
    no_master           = 1
---


An important things to note is that the MySQL user you specify in the MHA config must have all the privileges.


Pre-failover Steps During an Online Failover
============================================
To make sure that the failover is safe and does not cause any data inconsistencies, mha-helper takes the following steps before the failover:

1. Set read_only on the new master to avoid any data inconsistencies
2. Execute the following steps on the original master:
   1. Set read_only=1 on the original master
   2. Remove the writer VIP if manage_vip=yes in the global.conf file
   3. Wait upto 5 seconds for all connected threads to disconnect
   4. Terminate all the threads still connected except the replication related threads, the thread corresponding to the connection made by mha-helper and those threads which have been sleeping for more than 1 second as these threads would automatically be closed by MySQL server after wait_timeout seconds have elapsed.
   5. Disconnect from the original master


If any of the above steps fail, any changes made during pre-failover are rolledback.

Post-failover Steps During an Online Failover
============================================
Once the failover is completed by MHA, mha-helper takes the following steps:

1. Remove the read_only flag from the new master
2. Assign the writer VIP if manage_vip=yes in the global.conf file


Pre-failover Steps During Master Failover
=========================================
If the original master is accessible via SSH, i.e. in cases where MySQL crashed and stopped but the host is still up, then mha-helper takes the following step:

1. Remove the writer VIP if manage_vip=yes in the global.conf file


Post-failover Steps During Master Failover
==========================================
Once the failover is completed by MHA, mha-helper script takes the following steps:

1. Assign the writer VIP if manage_vip=yes in the global.conf file
2. Remove the read_only flag from the new master


Automated Failover and Monitoring via MHA Manager Daemon
========================================================
The daemon that daemonizes the MHA manager which monitors the master-slave hosts is available in the support-files directory. It supports 'start', 'stop', 'restart' and 'status' commands.

You can start the daemon like this:

    /usr/local/mha-helper/support-files/mha_manager_daemon --conf=/usr/local/mha-helper/conf/test_cluster.conf start

You can stop the daemon like this:

    /usr/local/mha-helper/support-files/mha_manager_daemon --conf=/usr/local/mha-helper/conf/test_cluster.conf stop

And you can restart the daemon like this:

    /usr/local/mha-helper/support-files/mha_manager_daemon --conf=/usr/local/mha-helper/conf/test_cluster.conf restart

And you can check the status of the MHA Manager process like this:

    /usr/local/mha-helper/support-files/mha_manager_daemon --conf=/usr/local/mha-helper/conf/test_cluster.conf status

There is also a sample SySV style init script available, you only need to change the name of the script and modify the configuration path:

    /usr/local/mha-helper/support-files/mha_manager_daemon-test_cluster-init


Manual Failover Examples
========================
Once everything is configured and running, doing the failover is pretty simple.

Do a failover when the master db1 goes down:

    /usr/local/mha-helper/bin/mysql_failover -d db1 -c /usr/local/mha-helper/conf/test_cluster.conf

Do an online failover:

    /usr/local/mha-helper/bin/mysql_online_failover -c /usr/local/mha-helper/conf/test_cluster.conf


Using Non-root User
===================
If you are using non-root user for connecting to master-slave hosts via ssh (the user that you use for this purpose is taken from the **ssh_user** option) then you need to make sure that the user can execute the following commands:
+ /sbin/ip
+ /sbin/arping

The user should be able to execute the above commands using sudo, and should not have to provide a password. This can accomplished by editing the file /etc/sudoers using visudo and adding the following lines:

    mha_helper   ALL=NOPASSWD: /sbin/ip, /sbin/arping
    Defaults:mha_helper !requiretty

In the example above I am assuming that ssh_user=mha_helper.


Some General Recommendations
============================
There are some general recommendations that I want to make, to prevent race-condition in special cases that can cause data inconsistencies
+ Do not persist interface with writer VIP in the network scripts. This is important for example in cases where both the candidate masters go down i.e. host goes down and then come back online. In which case we should need to manually intervene because there is no automated way to find out which MySQL server should be the source of truth
+ Persist read_only in the MySQL configuration file of all the candidate masters as well. This is again important for example in cases where both the candidate masters go down.


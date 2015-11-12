# clustercheck-iptables

A background script that checks the availability of a Galera node, and adds a redirection port using iptables if the Galera node is healthy (instead of returning HTTP response). This allows other TCP-load balancers with limited health check capabilities to monitor the backend Galera nodes correctly. 

Other than HAProxy, you can now use your favorite reverse proxy to load balance requests across Galera nodes, namely:
- nginx 1.9 (--with-stream)
- keepalived
- IPVS
- distributor
- balance
- pen

# How does it work?

0) Requires iptables

1) Monitors Galera node

2) If healthy (Synced + read_only=OFF) or (Donor + xtrabackup[-v2]), a port redirection will be setup using iptables (default: 3308 redirects to 3306)

3) Else, the port redirection will be ruled out from the iptables PREROUTING chain

On the load balancer, define the designated redirection port (3308) instead. For example on nginx 1.9 (configured with --with-stream):
```bash
upstream stream_backend {
  zone tcp_servers 64k;
  server 192.168.0.201:3308;
  server 192.168.0.202:3308;
  server 192.168.0.203:3308;
}
```
If the backend node is "unhealthy", 3308 will be unreachable because the corresponding iptables rule is removed on the database node and nginx will exclude it from the load balancing set accordingly.

# Install

1) On the Galera node, install the script into /usr/local/sbin:
```bash
$ git clone https://github.com/ashraf-s9s/clustercheck-iptables
$ cp clustercheck-iptables/mysqlchk_iptables /usr/local/sbin/
$ chmod 755 /usr/local/sbin/mysqlchk_iptables
```

2) Configure DB user/password (default as per below):
```mysql
mysql> GRANT PROCESS ON *.* TO 'mysqlchk_user'@'localhost' IDENTIFIED BY 'mysqlchk_password';
```
** If you choose not to use the default user, you can use -u and -p or -e in the command line. Example in the Run section.

3) Make sure iptables is running and ensure we setup the firewall rules for Galera services:
```bash
chkconfig iptables on # or systemctl enable iptables.service
service iptables start # or systemctl start iptables.service
iptables -I INPUT -m tcp -p tcp --dport 3306 -j ACCEPT
iptables -I INPUT -m tcp -p tcp --dport 3308 -j ACCEPT
iptables -I INPUT -m tcp -p tcp --dport 4444 -j ACCEPT
iptables -I INPUT -m tcp -p tcp --dport 4567:4568 -j ACCEPT
service iptables save
service iptables restart
```
** CentOS 7 comes with firewalld by default. You probably have to install iptables services beforehand, use ``yum install iptables-services``

# Run

The script must run as root/sudo to allow iptables changes. Append 'sudo' at the beginning of the command line if you want to run it as non-root user.

Test the script and ensure it detects Galera node healthiness correctly:
```bash
mysqlchk_iptables -t
```

Once satisfied, run the script as daemon:
```bash
mysqlchk_iptables -d
```

If you use non-default user/password, specify them as per below (replace -d with -t if you just want to test):
```bash
mysqlchk_iptables -d --user=check --password=checkpassword
```

To stop it:
```bash
mysqlchk_iptables -x
```

To check the process status:
```bash
mysqlchk_iptables -s
```

To make it starts on boot, add the command into ``/etc/rc.local``:
```bash
echo '/usr/local/sbin/mysqlchk_iptables -d' >> /etc/rc.local
```

** Make sure /etc/rc.local has permission to run on startup. Verify with:
```bash
chmod +x /etc/rc.local
```
Other parameters are available with ``--help``.

You can also use [supervisord](http://supervisord.org/) or [monit](https://mmonit.com/monit/) to automate and monitor the process.

# Caveats

### Credentials exposure

The script defaults to fork a background process which expose full parameters containing sensitive information e.g user/password. Example of the ps output:
```bash
$ ps aux | grep mysqlchk_iptables
root      26768  0.2  0.0 113248  1612 pts/4    S    07:08   0:01 /bin/bash /usr/local/sbin/mysqlchk_iptables --username=mysqlchk_user --password=mysqlchk_password --mirror-port=3308 --real-port=3306 --log-file=/var/log/mysqlchk_iptables --source-address=0.0.0.0/0 --check-interval=1 --defaults-extra-file=/etc/my.cnf -R
```

If you don't want user/password values to be exposed in the command line, specify the user credentials under [client] directive inside MySQL default extra file. In the command line, pass empty values for username and password (-u and -p) and use -e to include the extra file:
```bash
mysqlchk_iptables -d -u "" -p "" -e /root/.my.cnf
```

The example content of ``/root/.my.cnf``:
```bash
[client]
user='root'
password='password!@#$'
```

### Log rotation

By default, the script will log all activities into ``/var/log/mysqlchk_iptables``. It's recommended to setup a log rotation so it won't fill up your disk space. Create a new file at ``/etc/logrotate.d/mysqlchk_iptables`` and add following lines:

```bash
/var/log/mysqlchk_iptables {
     size 50M
     create 0664 root root
     rotate 3
     compress
}
```

To disable logging, redirect the output to ``/dev/null`` in the command line:
```bash
mysqlchk_iptables -d --log-file=/dev/null 
```

### Tested environment

This script is built and tested on:

* Percona XtraDB Cluster 5.6, MySQL Galera Cluster 5.6 and MariaDB Galera 10.0, galera 3.x
* CentOS 7.1
* iptables v1.4.21
* nginx/1.9.6
* nginx/1.9.4 (nginx-plus-r7-p1)

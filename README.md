# clustercheck-iptables

A background script to check Galera node availability and add a port redirection using iptables if Galera node is healthy. Derived from percona-clustercheck, but utilizing iptables port redirection instead of returning HTTP response. This allows TCP load balancer to perform health checks without custom monitoring port (percona-clustercheck runs on 9200 through xinetd).

This script allowing other TCP load balancers to monitor Galera nodes correctly, namely:
- nginx 1.9 (--with-stream)
- keepalived
- IPVS
- distributor
- balance
- pen

# How it works?

0) Requires iptables

1) Monitors Galera node (default: 1 sec)

2) If healthy (Synced + read_only=OFF), a port redirection will be setup using iptables (default: 3308 redirects to 3306)

3) Else, the port redirection will be ruled out from the iptables NAT chain

On the load balancer, define the designated redirection port (3308) instead. For example on nginx 1.9 (configured with --with-stream):
```bash
upstream stream_backend {
  zone tcp_servers 64k;
  server 192.168.0.201:3308;
  server 192.168.0.202:3308;
  server 192.168.0.203:3308;
}
```
If the backend node is not "healthy", 3308 will be unreachable because the corresponding iptables rule is removed on the database node and nginx will exclude it from the load balancing set.

# Install

1) On the Galera node, install the script into /usr/local/bin:
```bash
$ git clone https://github.com/ashraf-s9s/clustercheck-iptables
$ cp clustercheck-iptables/mysqlchk_iptables /usr/local/bin/
$ chmod 755 /usr/local/bin/mysqlchk_iptables
```

2) Configure DB user/password (default as per below):
```mysql
mysql> GRANT PROCESS ON *.* TO 'clustercheckuser'@'localhost' IDENTIFIED BY 'clustercheckpassword!';
```

3) Make sure iptables is running and ensure we setup the firewall rules for Galera services:
```bash
iptables -I INPUT -m tcp -p tcp --dport 3306 -j ACCEPT
iptables -I INPUT -m tcp -p tcp --dport 3308 -j ACCEPT
iptables -I INPUT -m tcp -p tcp --dport 4444 -j ACCEPT
iptables -I INPUT -m tcp -p tcp --dport 4567:4568 -j ACCEPT
service iptables save
service iptables restart
```
** CentOS 7 comes with firewalld by default. You probably have to install iptables beforehand, use ``yum install iptables-services``

# Run

Run the script as root/sudo (since it requires iptables changes) in the background:
```bash
sudo /usr/local/bin/mysqlchk_iptables &
```
** Omit sudo if you run as root

To make it starts on boot, add the command into ``/etc/rc.local``:
```bash
echo '/usr/local/bin/mysqlchk_iptables &' >> /etc/rc.local
```
You can also use [supervisord](http://supervisord.org/) or [monit](https://mmonit.com/monit/) to monitor the process.

# Logging

By default, the script will log all activities into ``/var/log/mysqlchk_iptables``. It's recommended to setup a log rotation so it won't fill up your disk space. Create a new file at ``/etc/logrotate.d/mysqlchk_iptables`` and add following lines:

```bash
/var/log/mysqlchk_iptables {
     size 50M
     create 0664 root root
     rotate 3
     compress
}
```

To disable logging, just replace the line ``LOG_FILE="/var/log/mysqlchk_iptables"`` with ``LOG_FILE=/dev/null``.

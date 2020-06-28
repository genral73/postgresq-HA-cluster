# Postgresql-HA-Cluster on Centos 8 / RHEL 8
Highly Available PostgreSQL cluster using  Etcd, Patroni and HAProxy on Redhat 8 

## Prerequisites
you will need 4 (physical or virtual) machines with RHEL 8 server with minimal installed.
> HOSTNAME  |   IP ADDRESS    | 	PURPOSE
>
> node01	  | 192.168.100.41  |	Postgresql, Etcd, Patroni
>
> node02	  | 192.168.100.42	| Postgresql, Etcd, Patroni
>
> node03	  | 192.168.100.43	| Postgresql, Etcd, Patroni
>
> node04    | 192.168.100.45	| HAProxy


## 1) Environment Preparation

#### a) If you have SELinux running in enforcing mode, then generate a local policy module to allow access to data directories:
```bash
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
```

#### b) Ensure you have vim and wget installed on your RHEL / CentOS 8 server:
```bash
yum update
yum -y install wget vim curl net-tools tar
```
### c) Make sure you have performed all of the above steps [a,b] on each node [01,02,03,04].

## 2) Installing PostgreSQL
We are installing PostgreSQL version 12 for this guide. If you wish, you can install any other version of your choice: 

#### a) Install the repository RPM:
```bash
dnf -y install https://github.com/genral73/postgresql-HA-cluster/blob/RHEL8/RPMS/pgdg-redhat-repo-latest.noarch.rpm
```
#### b) Disable the built-in PostgreSQL module:
```bash
dnf -qy module disable postgresql
```
#### c) Install PostgreSQL:
```bash
dnf -y install postgresql12-server postgresql12 postgresql12-devel
```
#### d) Make sure you have performed all of the above steps [a,b,c] on each node that are designated for postgresql,etcd and patroni (node01, node02, node03 in our case) before going to the next step of installing and configuring etcd.

## 3) Installing etcd
Etcd is a fault-tolerant, distributed key-value store that is used to store the state of the Postgres cluster. Using Patroni, all of the Postgres nodes make use of etcd to keep the Postgres cluster up and running.
#### a) Get and extract the archive file, after that move etcd and etcdctl binary files to /usr/local/bin directory :
```bash
wget https://github.com/genral73/postgresql-HA-cluster/blob/RHEL8/RPMS/etcd-v3.3.22-linux-amd64.tar.gz
tar xvf etcd-v3.3.22-linux-amd64.tar.gz
mv etcd-v3.3.22-linux-amd64/etcd* /usr/local/bin/
etcdctl sudo groupadd --system etcd
groupadd --system etcd
useradd -s /sbin/nologin --system -g etcd etcd
etcd --version
```
#### b) At this point, you need to edit the /etc/etcd/etcd.conf file like below:
```bash
mkdir /etc/etcd
vim /etc/etcd/etcd.conf
  """
  name: "etcd1"
  data-dir: "/var/lib/etcd"
  initial-advertise-peer-urls: "http://192.168.100.41:2380"
  listen-peer-urls: "http://192.168.100.41:2380"
  listen-client-urls: "http://192.168.100.41:2379,http://127.0.0.1:2379"
  advertise-client-urls: "http://192.168.100.41:2379"
  initial-cluster-token: "etcd-cluster-0"
  initial-cluster: "etcd1=http://192.168.100.41:2380,etcd2=http://192.168.100.42:2380,etcd3=http://192.168.100.43:2380"
  initial-cluster-state: "new"
  """
chown etcd:etcd -R /etc/etcd
```
#### c) At this point, you need to edit the /etc/systemd/system/etcd.service file like below:
```bash
mkdir -p /var/lib/etcd/
chown -R etcd:etcd /var/lib/etcd/
vim /etc/systemd/system/etcd.service
  """
  Description=Etcd Server
  After=network.target
  After=network-online.target
  Wants=network-online.target

  [Service]
  Type=notify
  WorkingDirectory=/var/lib/etcd/
  User=etcd
  # set GOMAXPROCS to number of processors
  ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd  --config-file /etc/etcd/etcd.conf"
  Restart=always
  RestartSec=10s
  LimitNOFILE=65536

  [Install]
  WantedBy=multi-user.target
  """
```
#### d) Reload systemd service and start etcd:
```bash
systemctl daemon-reload
systemctl enable etcd
systemctl restart etcd
systemctl status etcd -l
```
#### e) Test your etcd installation:
```bash
ss -tunelp | grep 2379
etcdctl member list
etcdctl set /message "Hello World"
etcdctl get /message
etcdctl rm /message
```

#### f) Make sure you have performed all of the above steps [a,b,c,d,e] on each node that are designated for postgresql,etcd and patroni (node01, node02, node03 in our case) before going to the next step of installing and configuring patroni.


## 4) Installing Patroni
Patroni is an open-source python package that manages Postgres configuration. It can be configured to handle tasks like replication, backups, and restorations.
#### a) Type below command to install patroni on (node01, node02, node03) only:
```bash
dnf -y install https://github.com/genral73/postgresql-HA-cluster/blob/RHEL8/RPMS/python3-psycopg2-2.7.7-2.el7.x86_64.rpm
dnf -y install https://github.com/genral73/postgresql-HA-cluster/blob/RHEL8/RPMS/patroni-1.6.5-1.rhel7.x86_64.rpm
```
#### b) Type below command on (node01, node02, node03) to edit postgresql.yml:
```bash
vim /opt/app/patroni/etc/postgresql.yml
  """
    scope: pg-cluster
    name: node01

    restapi:
      listen: 192.168.100.41:8008
      connect_address: 192.168.100.41:8008
    etcd:
        host: 192.168.100.41:2379
        host: 192.168.100.42:2379
        host: 192.168.100.43:2379

    bootstrap:
      dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
          use_pg_rewind: true
          use_slots: true
          parameters:
      initdb:  # Note: It needs to be a list (some options need values, others are switches)
      - encoding: UTF8
      - data-checksums
      pg_hba:  # Add following lines to pg_hba.conf after running 'initdb'
      - host replication replicator 127.0.0.1/32 md5
      - host replication replicator 192.168.100.41/0 md5
      - host replication replicator 192.168.100.42/0 md5
      - host replication replicator 192.168.100.43/0 md5
      - host all all 0.0.0.0/0 md5
      users:
        admin:
          password: admin
          options:
            - createrole
            - createdb

    postgresql:
      listen: 192.168.100.41:5432
      connect_address: 192.168.100.41:5432
      data_dir: /var/lib/pgsql/12/data
      bin_dir: /usr/pgsql-12/bin
      pgpass: /tmp/pgpass
      authentication:
        replication:
          username: replicator
          password: reppassword
        superuser:
          username: postgres
          password: postgrespassword
    tags:
        nofailover: false
        noloadbalance: false
        clonefrom: false
        nosync: false
  """    
```
#### c) Starting Patroni At this point, you need to start patroni service on your first node (node01 in our case):
```bash
systemctl enable patroni
systemctl start patroni
systemctl status patroni
```
When starting patroni on subsequent nodes, (node02, node03 in our case).

#### d) Make sure patroni service is running on each node (node01, node02, node03 in our case) before going to the next step of installing and configuring haproxy.

## 4) Installing HAProxy
HAProxy forwards the connection to whichever node is currently the master. It does this using a REST endpoint that Patroni provides. Patroni ensures that, at any given time, only the master Postgres node will appear as online, forcing HAProxy to connect to the correct node.

#### a) Type the below command to install HAProxy on (node04 in our case):
```bash
dnf -y install haproxy
```
#### b) Edit configuration file like below :
```bash
vim /etc/haproxy/haproxy.cfg
  """
  global
      maxconn 100

  defaults
      log global
      mode tcp
      retries 2
      timeout client 30m
      timeout connect 4s
      timeout server 30m
      timeout check 5s

  listen stats
      mode http
      bind *:7000
      stats enable
      stats uri /

  listen postgres
      bind *:5000
      option httpchk
      http-check expect status 200
      default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
      server node01 192.168.100.41:5432 maxconn 100 check port 8008
      server node02 192.168.100.42:5432 maxconn 100 check port 8008
      server node03 192.168.100.43:5432 maxconn 100 check port 8008
  """
```
#### c) Now start HAProxy to take changes into effect with the below command:
```bash
systemctl enable haproxy
systemctl restart haproxy
systemctl status haproxy
```
#### d) Testing Postgres HA Cluster Setup
###### d-1) You can also access HAProxy node (192.168.100.44 in our case) on port 7000 using any of your preferred web browsers to see your HA Cluster status:
[http://192.168.100.44:7000](http://192.168.100.44:7000)


###### d-2) Connect Postgres clients to the HAProxy:
```bash
PGPASSWORD=postgrespassword psql --host=192.168.100.44  --port=5000 --username=postgres
```








































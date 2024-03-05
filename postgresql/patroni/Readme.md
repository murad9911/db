### Set host file

10.10.10.10    node1
10.10.10.20    node2    
10.10.10.30    node3

### Install the repository RPM
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm

### Uninstall default postgresql#########
yum remove postgresql.x86_64

### Install postgresql server
dnf install -y postgresql15-server postgresql-contrib postgresql15

### Install reqired package
yum install python3-pip python3-devel binutils -y
yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm
percona-release setup ppg13
yum install percona-patroni etcd python3-python-etcd percona-pgbackrest

### Firewall

firewall-cmd --add-port=2379/tcp --permanent
firewall-cmd --add-port=2380/tcp --permanent
firewall-cmd --add-port=8008/tcp --permanent
firewall-cmd --add-port=5432/tcp --permanent
firewall-cmd --reload

### ETCD config

systemctl stop {etcd,patroni,postgresql}
systemctl disable {etcd,patroni,postgresql}
mv  /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bk
vi /etc/etcd/etcd.conf                             *Add etcd.conf*
systemctl enable --now etcd

### Patroni setup

mkdir -p /var/data/patroni
chown -R postgres:postgres /var/data/
chmod 700 /var/data/patroni
vi /etc/patroni.yml
chown postgres:postgres /etc/patroni.yml
vi /etc/systemd/system/patroni.service
systemctl daemon-reload
systemctl enable --now patroni


### Test cluster

psql -h 10.10.10.10 -p 5432 -U postgres
CREATE DATABASE app;
 \c app;
 CREATE TABLE accountss (
	user_id serial PRIMARY KEY,
	username VARCHAR ( 50 ) UNIQUE NOT NULL,
	password VARCHAR ( 50 ) NOT NULL,
	email VARCHAR ( 255 ) UNIQUE NOT NULL,
	created_on TIMESTAMP NOT NULL,
        last_login TIMESTAMP
);
 \d accounts;
 INSERT INTO accounts (username, password, email, created_at, last_login)
VALUES ('user', '123456', 'user@test.local', 11-12-13, 11-12-13);

Select * from accounts;


GRANT ALL ON SCHEMA public TO demo;
##########################


#######Patroni switchover datacenter#########
####Check repilication
\x on;
select * from pg_stat_replication ;  ###leader node
select * from pg_stat_wal_receiver;  #### replica node
################
systemctl stop patroni  ####all node primary DR
patronictl -c /etc/patroni.yml list  ### check patroni status on primary cluster
patronictl -c /etc/patroni.yml edit-config --force -s standby_cluster.host='' -s standby_cluster.port='' -s standby_cluster.create_replica_methods=''   #### Set standby as primary
patronictl -c /etc/patroni.yml edit-config --force -s standby_cluster.host=<dc2-ip> -s standby_cluster.port=<port> -s standby_cluster.create_replica_methods='- basebackup'   ##### Set Main Primary as Standby-cluster
patronictl -c /etc/patroni.yml reinit ####first primary node2 and node3

# Automatic-Failover-High-Availibility-postgresql
PostgreSQL Automatic Failover High-Availibility pacemaker corosync pcs
## PostgreSQL Automatic Failover High-Availibility for Redmine-Postgres based on Pacemaker


The master/slave database replication is a process of copying (syncing) data from a database on one server (the master) to a database on another server (the slaves). The main benefit of this process is to distribute databases to multiple machines, so when the master server has a problem, there is a backup machine with same data available for handling requests without interruption.





### Repository setup

In this tutorial, we will install the latest version of PostgreSQL 9.6. In the official Ubuntu repository, they provide PostgreSQL 9.5 only, so we need to install the latest version from the PostgreSQL repository directly.

<pre><code class="text">
cat <<EOF >> /etc/apt/sources.list.d/pgdg.list
deb http://apt.postgresql.org/pub/repos/apt/ stretch-pgdg main
EOF
</code></pre>

Now, update your local apt cache:

<pre><code class="text">
apt install ca-certificates
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add
apt update
apt install pgdg-keyring
</code></pre>

### Hosts setup

<pre><code class="text">
**.226.179.205   oreo    server1
**.226.179.206   cupcone server2
**.226.179.207   twinkie server3
**.226.179.208   rcluster

</code></pre>

Now, the three servers should be able to ping each others

<pre><code class="text">
for s in server1 server2 server3; do ping -W1 -c1 $s; done| grep icmp_seq
</code></pre>

### Install PostgreSQL

Run on ALL nodes.

<pre><code class="text">
apt install -y postgresql-9.6 postgresql-contrib-9.6 postgresql-client-9.6
apt install resource-agents-paf

</code></pre>

Please change the password for the postgres user and check the connection info with postgres queries below

<pre><code class="text">
su - postgres
psql
\password postgres
\conninfo
\q
exit

</code></pre>

### Config PostgreSQL 
 

In the postgresql.conf file, the archive mode is enabled, so we need to create a new directory for the archive. Create a new archive directory, change the permission and change the owner to the postgres user. *do only on server1*

<pre><code class="text">
mkdir -p /var/lib/postgresql/9.6/main/archive/
chmod 700 /var/lib/postgresql/9.6/main/archive/
chown -R postgres:postgres /var/lib/postgresql/9.6/main/archive/
</code></pre>


edit  the postgresql config file

<pre><code class="text">
nano  /etc/postgresql/9.6/main/postgresql.conf
</code></pre>

*on server1* 


<pre><code class="text">
listen_addresses = '*'
wal_level = replica
synchronous_commit = local
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/9.6/main/archive/%f'
max_wal_senders = 10
wal_keep_segments = 10
hot_standby = on
hot_standby_feedback = on
logging_collector = on
</code></pre>

*on slave servers*

<pre><code class="text">
listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
hot_standby = on
hot_standby_feedback = on
logging_collector = on

</code></pre>

Edit pg_hba.conf on all file differently

*On Server1*

<pre><code class="text">
host replication postgres **.226.179.208/32 reject

host replication postgres ** .226.179.205/32 reject

# allow any standby connection
host replication postgres 0.0.0.0/0 trust
</code></pre>

*On Server 2*

<pre><code class="text">
host replication postgres **.226.179.205/32  reject
host replication postgres **.226.179.206/32  reject

# allow any standby connection
host replication postgres 0.0.0.0/0 trust
</code></pre>

*On server 3*
<pre><code class="text">
# forbid self-replication
host replication postgres **.226.179.205/32  reject
host replication postgres **.226.179.207/32  reject

# allow any standby connection
host replication postgres 0.0.0.0/0 trust
</code></pre>

Create recovery file in all server

<pre><code class="text">
nano /etc/postgresql/9.6/main/recovery.conf.pcmk
</code></pre>

you need change _application_name=*_  on each server. if it is in server2 then _application_name=_ will be server2

<pre><code class="text">
standby_mode = on
primary_conninfo = 'host= **.226.179.205  application_name=server1'
recovery_target_timeline = 'latest'
</code></pre>



Restart PostgreSQL in all servers

<pre><code class="text">
systemctl restart postgresql
netstat -plntu |grep 5432
</code></pre>

Now, on each standby (server2 and server3 here), we have to cleanup the instance created by the package and clone the primary.

we want to replace the postgres main directory on the 'SLAVE' server with the main data directory from 'MASTER' server.

<pre><code class="text">
su - postgres
cd 9.6/

mv main main-bekup
mkdir main/
chmod 700 main/
pg_basebackup -h ** .226.179.205 -D ~postgres/9.6/main/ -X stream -P
</code></pre>

#### Recovery file

create a recovery file on both slave server

<pre><code class="text">
cd /var/lib/postgresql/9.6/main/
nano recovery.conf
</code></pre>

<pre><code class="yaml">
standby_mode = 'on'
primary_conninfo = 'host=**.226.179.205  user=postgres password=Qwe12345! application_name=server2'
restore_command = 'cp /var/lib/postgresql/9.6/main/archive/%f %p'
trigger_file = '/tmp/postgresql.trigger.5432'
</code></pre>

<pre><code class="text">
chmod 600 recovery.conf
</code></pre>

<pre><code class="text">
exit
systemctl start postgresql
netstat -plntu |grep 5432
</code></pre>



#### Testing PostgreSQL replication success (Server1)


For testing, we will check the replication status of the PostgreSQL 9.6 and try to create a new table on the MASTER server, then check the replication by checking all data from the SLAVE server.


<pre><code class="text">
su - postgres
psql -c "select application_name, state, sync_priority, sync_state from pg_stat_replication;" 
psql -x -c "select * from pg_stat_replication;"
</code></pre>

#### Migrate redmine database to postgres

By default redmine in redmine_default database. we need to change to Postgres user and redmine database

<pre><code class="text">
cd 
su postgres
psql
</code></pre>

<pre><code class="sql">
CREATE DATABASE redmine WITH ENCODING='UTF8' OWNER=postgres;
</code></pre>

<pre><code class="text">
\q
exit
</code></pre>

  
edit on all nodes

<pre><code class="text">
nano /etc/redmine/default/database.yml
</code></pre>


<pre><code class="yaml">
#production:
#  adapter: postgresql
#  database: redmine_default
#  host: localhost
#  port:
#  username: redmine/instances/default
#  password: lPio5mZFTj6O
#  encoding: utf8

production:
  adapter: postgresql
  encoding: utf8
  database: redmine
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: postgres
  password: Qwe12345!
  host: localhost
</code></pre>

<pre><code class="text">
cd /usr/share/redmine
rake db:migrate
</code></pre>

### Cluster creation

you should config _nano /etc/corosync/corosync.conf_ based on 3 severs

<pre><code class="text">
passwd hacluster
</code></pre>


Enter password for hacluster (for example 12345678)

<pre><code class="text">
pcs cluster auth server1 server2 server3 -u hacluster -p 12345678 --force
pcs cluster setup --name mycluster server1 server2 server3 --force
</code></pre>

start all cluster node

<pre><code class="text">
pcs cluster start --all
pcs status
</code></pre>

resource-stickiness: adds a sticky score for the resource on its current node. It helps avoiding a resource move back and forth between nodes where it has the same score.
migration-threshold: this controls how many time the cluster tries to recover a resource on the same node before moving it on another one.

<pre><code class="text">
pcs resource defaults migration-threshold=5
pcs resource defaults resource-stickiness=10
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
</code></pre>

### Cluster resources

Now we need to setup Resources  MasterVip (virtual ip address), Apache for Redmine , Master&slave resource for PostgreSQL.

*Create Resource for virtual Ip Address* 

<pre><code class="text">
pcs resource create MasterVip ocf:heartbeat:IPaddr2 ip=10.226.179.208 nic="lo" cidr_netmask=32 iflabel="pgrepvip" op monitor interval=90s meta target-role="Started"
</code></pre>

*Create Resource for Apache*

<pre><code class="text">
pcs resource create Apache ocf:heartbeat:apache  configfile=/etc/apache2/apache2.conf  statusurl="http://localhost/server-status"  op monitor interval=1min --force
</code></pre>

* Ensure Resources Run on the Same Host*

<pre><code class="text">
pcs constraint colocation add Apache MasterVip  INFINITY
</code></pre>


The _pgsqld_ defines the properties of a PostgreSQL instance: where it is located, where are its binaries, its configuration files, how to montor it, and so on.

The _pgsql-ha_ resource controls all the PostgreSQL instances _pgsqld_ in your cluster, decides where the primary is promoted and where the standbys are started.

<pre><code class="text">
 pcs resource create pgsqld ocf:heartbeat:pgsqlms    \
    bindir="/usr/lib/postgresql/9.6/bin"                            \
    pgdata="/etc/postgresql/9.6/main"                               \
    datadir="/var/lib/postgresql/9.6/main"                          \
    pghost="/var/run/postgresql"                                    \
    recovery_template="/etc/postgresql/9.6/main/recovery.conf.pcmk" \
    op start timeout=60s                                            \
    op stop timeout=60s                                             \
    op promote timeout=30s                                          \
    op demote timeout=120s                                          \
    op monitor interval=15s timeout=10s role="Master"               \
    op monitor interval=16s timeout=10s role="Slave"                \
    op notify timeout=60s
</code></pre>

We also set the notify option so that the cluster will tell agent when its peer changes state.

<pre><code class="text">
pcs  resource master pgsql-ha pgsqld notify=true
pcs  resource cleanup
pcs status
</code></pre>

We now define the collocation between pgsql-ha and MasterVip . The start/stop and promote/demote order for these resources must be asymetrical: we MUST keep the master IP on the master during its demote process so the standbies receive everything during the master shutdown.

We do this by adding an ordering constraint. By default, all order constraints are mandatory, which means that the recovery of MasterVip will also trigger the recovery of pgsql-ha.

<pre><code class="text">
pcs constraint colocation add MasterVip  with master pgsql-ha INFINITY
pcs  constraint order promote pgsql-ha then start MasterVip symmetrical=false kind=Mandatory
pcs  constraint order demote pgsql-ha then stop MasterVip symmetrical=false kind=Mandatory
</code></pre>

#### shutdown server1 

<pre><code class="text">
pcs cluster stop server1
</code></pre>


Need around 10 to 15 min to make redmine to connact client server. so need little patience

### Output

<pre><code class="text">
root@oreo:~# pcs status
Cluster name: mycluster
Stack: corosync
Current DC: server3 (version 1.1.16-94ff4df) - partition with quorum
Last updated: Tue Aug 25 14:19:47 2020
Last change: Tue Aug 25 12:34:03 2020 by root via crm_attribute on server2

3 nodes configured
5 resources configured

Online: [ server1 server2 server3 ]

Full list of resources:

 MasterVip      (ocf::heartbeat:IPaddr2):       Started server2
 Apache (ocf::heartbeat:apache):        Started server2
 Master/Slave Set: pgsql-ha [pgsqld]
     Masters: [ server2 ]
     Slaves: [ server1 server3 ]

Daemon Status:
  corosync: active/disabled
  pacemaker: active/disabled
  pcsd: active/enabled
</code></pre>

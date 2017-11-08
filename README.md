# Grafana SiriDB HTTP example
The goal of this blog is to setup a Grafana dashboard using the SiriDB plugin.

This is a tutorial on how to setup SiriDB including a replica and a grafana dashboad.

at least version 2.0.14
```
siridb-server --version
```

at least version 1.1.2
```
siridb-admin --version
```

at least 2.0.3
```
siridb-http --version
```

create config files for siridb:
```
for i in {0..3}; do `cat <<EOT > siridb$i.conf
[siridb]
listen_client_port = 900$i
server_name = %HOSTNAME:901$i
ip_support = ALL
optimize_interval = 900
heartbeat_interval = 30
default_db_path = ./dbpath$i
max_open_files = 512
EOT` && mkdir dbpath$i; done
```

start siridb
```
siridb-server -c siridb0.conf &
```

```
siridb-server -c siridb1.conf &
```

Create a database:
```
siridb-admin -u sa -p siri -s localhost:9000 new-database -d dbtest -t "s" --duration-num "40w"
```

create a replica
```
siridb-admin -u sa -p siri -s localhost:9001 new-replica -d dbtest -U iris -P siri -S localhost:9000 --pool 0 --force
```

or create a pool
```
siridb-admin -u sa -p siri -s localhost:9001 new-pool -d dbtest -U iris -P siri -S localhost:9000 --force
```

insert data...
for this demo we create a Python3 script.

requirements:
```
pip3 install siridb-connector
pip3 install psutil
```

```
python3 mon2siridb.py -u iris -p siri -d dbtest -n 0 -i 5 -t s -s localhost:9000,localhost:9001 &
```

create siridb http configuration file: (make sure to enable basic authentication)
```
cat <<EOT > siridb-http.conf
[Database]
user = iris
password = siri
dbname = dbtest
servers = localhost:9000,localhost:9001

[Configuration]
port = 5050
require_authentication = True
enable_socket_io = True
enable_ssl = False
enable_web = True
enable_basic_auth = True
enable_multi_user = False
cookie_max_age = 604800
insert_timeout = 60
EOT
```

Start siridb-http
```
siridb-http -c siridb-http.conf &
```


```
create group `received` for /siridb-server.*received_points/
create group `selected` for /siridb-server.*selected_points/
create group `mem_usage` for /siridb-server.*mem_usage/
create group `uptime` for /siridb-server.*uptime/
create group `series` for /siridb-database.*series/
create group `points` for /siridb-database.*points/

```
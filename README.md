# Grafana and SiriDB tutorial

The goal of this blog is to setup a Grafana dashboard using the SiriDB plugin. For a nice dashboard we need some data which we can viaualize.
In this tutorial we use a Python script which collects some data about the host and the running SiriDB processes. This data will be stored in
a SiriDB database and by using a Grafana Dashboard we are able to monitor the data.

We use a fresh Ubnutu 16.04 installation so you might want to skip some steps or change some commands according to your operating system.

Update and install Git, Pip and libuv1 by using apt.
```
sudo apt update
sudo apt upgrade
sudo apt install libuv1 git python3-pip
```

We start with getting this documentation and Python script wight are made available in a git repository
```
git clone https://github.com/transceptor-technology/grafana-siridb-http-example.git
cd /grafana-siridb-http-example
```

Install SiriDB Server
```
wget https://github.com/transceptor-technology/siridb-server/releases/download/2.0.25/siridb-server_2.0.25_amd64.deb
sudo dpkg -i siridb-server_2.0.25_amd64.deb
```

Since we don't require SiriDB to start at startup, we disable the service:
```
sudo systemctl disable siridb-server.service
```

SiriDB has an admin tool which can be used to create and manage databases so we should install that tool:
```
wget https://github.com/transceptor-technology/siridb-admin/releases/download/1.1.2/siridb-admin_1.1.2_linux_amd64.bin
chmod +x siridb-admin_1.1.2_linux_amd64.bin
sudo cp siridb-admin_1.1.2_linux_amd64.bin /usr/local/bin
sudo ln -s /usr/local/bin/siridb-admin_1.1.2_linux_amd64.bin /usr/local/bin/siridb-admin
```

There are several native clients available for communicating with SiriDB but for Grafana we will use SiriDB HTTP which
provides a HTTP(S) API.
```
wget https://github.com/transceptor-technology/siridb-http/releases/download/2.0.4/siridb-http_2.0.4_linux_amd64.bin
chmod +x siridb-http_2.0.4_linux_amd64.bin
sudo cp siridb-http_2.0.4_linux_amd64.bin /usr/local/bin
sudo ln -s /usr/local/bin/siridb-http_2.0.4_linux_amd64.bin /usr/local/bin/siridb-http
```

SiriDB can scale data accross multiple pools and each pool can have two servers for redundancy. We can play with this
concept on a single host by running SiriDB multiple times using different ports. In a real scenario you should use
different nodes but for now we will create four SiriDB nodes and setup two pools, each with two SiriDB "servers".

This will create four SiriDB configuration files:
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

We can start the siridb servers! The following command start the four SiriDB server in the background.
```
for i in {0..3}; do siridb-server -c siridb$i.conf > siridb$i.log & done
```

> Tip: you can view the output from a siridb process by using for example `cat siridb0.log` or `tail -f siridb0.log`.

Now we need the SiriDB Admin tool to create the actual database. SiriDB has a default service account `sa` with password `siri` which we will use.
For our tutorial we only need a database with `second` precision so we should add the `-t` flag. If you want to learn more about the
admin tool, you can look at the Github page: https://github.com/transceptor-technology/siridb-admin#readme

This command creates the database on the first siridb server:

```
siridb-admin -u sa -p siri -s localhost:9000 new-database -d tutorialdb -t "s"
```

Now we have a database and we can use the default database user `iris` with password `siri` to extend the database
with a replica on the second server:
```
siridb-admin -u sa -p siri -s localhost:9001 new-replica -d tutorialdb -U iris -P siri -S localhost:9000 --pool 0 --force
```

Ok, everything is ready to collect data. As said we use a Python script for collecting some sample data. Before you can run the python script make sure all
dependencies are installed:
```
pip3 install siridb-connector psutil
```

Start the script. The script accepts arguments which can be viewed with `python3 mon2siridb.py -h`. If you followed the toturial then the defaults should be fine.
```
python3 mon2siridb.py > mon.log &
```

Let's setup Grafana so we can view what we are collecting. First install Grafana:
```
wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana_4.6.1_amd64.deb
sudo dpkg -i grafana_4.6.1_amd64.deb
```

And install the Grafana-SiriDB-Datasource plugin:
```
sudo mkdir /var/lib/grafana/plugins/
sudo git clone https://github.com/transceptor-technology/grafana-siridb-http-datasource.git /var/lib/grafana/plugins/grafana-siridb-http-datasource
```

Start (or restart) Grafana:
```
sudo systemctl restart grafana-server.service
```

Before we can use the SiriDB datasource, we also need to configure and start the SiriDB HTTP connector.
SiriDB HTTP requires a configuration file. For more information you can view the following Github page:
https://github.com/transceptor-technology/siridb-http#readme

This will create a basic configuration file which is fine for our tutorial. Note that we connect
to both to the first and second siridb server for redundancy.
```
cat <<EOT > siridb-http.conf
[Database]
user = iris
password = siri
dbname = tutorialdb
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

Start SiriDB HTTP
```
siridb-http -c siridb-http.conf > siridb-http.log &
```

Open a browser and go to http://localhost:3000. You should see the following page:

![alt Grafana login](/grafana-login.png?raw=true)

Sign-in by using username `admin` and password `admin`.

Click on 'New datasource' to create the SiriDB data source. Fill in the form like below:

![alt Grafana add data source](/grafana-add-data-source.png?raw=true)

Clicking on 'Save and test' should return message that everyting is working!

From the menu, click on 'Dashboards -> Import'

![alt Grafana menu dashboard import](/grafana-menu-dashboard-import.png?raw=true)

Click on 'Upload JSON' and select the 'tutorial-dashboard.json' from this folder.
On the next window you should choose the SiriDB HTTP data source.

![alt Grafana import dashboard](/grafana-import-dashboard.png?raw=true)

After clicking on 'Import' you shout see a dashboard similar to this:

![alt Grafana tutorial dashboard](/grafana-tutorial-dashboard.png?raw=true&v=1)

We can now continue by expanding the database with another pool.

```
siridb-admin -u sa -p siri -s localhost:9002 new-pool -d tutorialdb -U iris -P siri -S localhost:9000 --force
```

In the dashboard you should see the new server. The status for the existing servers includes 're-indexing' while
the series are spread across the pools.

![alt Grafana re-indexing](/grafana-re-indexing.png?raw=true)

Wait until the status for all three server is just 'running' and then create another replica on the fourth server for `pool 1`:
```
siridb-admin -u sa -p siri -s localhost:9003 new-replica -d tutorialdb -U iris -P siri -S localhost:9000 --pool 1 --force
```

The dashboard should show the forth server with status 'synchronizing'
![alt Grafana synchronizing](/grafana-synchronizing.png?raw=true)




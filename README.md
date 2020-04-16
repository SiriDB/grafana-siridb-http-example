# Using Grafana with SiriDB

The goal of this blog is to setup a Grafana dashboard using the SiriDB plugin. [SiriDB](https://github.com/SiriDB/siridb-server) is an open source time series database that can be used to store any time related data.

For an attractive dashboard we need some data which we can visualize.
In this tutorial we use a Python script that collects some cpu, disk and memory data from the localhost and some information about the running SiriDB processes. All this data will be stored in a SiriDB database and by using a Grafana Dashboard we are able to monitor the data.

We use a fresh Ubuntu 19.10 (desktop) installation so you might want to skip some steps or change some commands according to your operating system.

Update and install Git, Pip and libuv1 by using apt.
```
sudo apt update
sudo apt upgrade
sudo apt install libuv1 git python3-pip
```

We start by downloading the Python script (and this tutorial):
```
git clone https://github.com/SiriDB/grafana-siridb-http-example.git
cd ./grafana-siridb-http-example
```

Next we install SiriDB:
```
wget https://github.com/SiriDB/siridb-server/releases/download/2.0.35/siridb-server_2.0.35_amd64.deb
sudo dpkg -i siridb-server_2.0.35_amd64.deb
```

We don't require SiriDB to start at startup so we disable the service:
```
sudo systemctl disable siridb-server.service
```

Since version 2.0.35 it is possible to use the HTTP API to create and manage databases. However it is also possible to use the [admin tool](https://github.com/SiriDB/siridb-admin) for this.

There are several native clients available for communicating with SiriDB, for Grafana we will use SiriDB HTTP which
provides a HTTP(S) API.
```
wget https://github.com/SiriDB/siridb-http/releases/download/2.0.14/siridb-http_2.0.14_linux_amd64.bin
chmod +x siridb-http_2.0.14_linux_amd64.bin
sudo cp siridb-http_2.0.14_linux_amd64.bin /usr/local/bin
sudo ln -s /usr/local/bin/siridb-http_2.0.14_linux_amd64.bin /usr/local/bin/siridb-http
```

SiriDB can scale data across multiple pools and each pool can have two servers for redundancy. We can play with this
concept on a single host by running SiriDB multiple times using different ports. In a real scenario you should use
different nodes but for now we will create four SiriDB nodes and setup two pools, each with two SiriDB servers.

This action will create four SiriDB configuration files:
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
http_api_port = 902$i
EOT` && mkdir dbpath$i; done
```

Now we can start the SiriDB servers! The following command starts the four SiriDB servers in the background.
```
for i in {0..3}; do siridb-server -c siridb$i.conf > siridb$i.log & done
```

> Hint: you can view the output from a SiriDB process by using for example `cat siridb0.log` or `tail -f siridb0.log`.

Now we use the SiriDB HTTP API to create the actual database. SiriDB has a default service account `sa` with password `siri` which we will use.
For our tutorial we will only need a database with `second` precision. We also select a shard duration of 6 hours for this database because our measurement interval will only be a few seconds. Sometimes you might want to store one value per measurement in each hour or even per day in which case your database will perform better by using a larger shard duration.

Create the database on the first SiriDB server which is running on port `9000` using curl with basic authentication:

```
curl --location --request POST 'http://localhost:9020/new-database' \
--header 'Content-Type: application/json' \
--header 'Authorization: Basic c2E6c2lyaQ==' \
--header 'Content-Type: text/plain' \
--data-raw '{
    "dbname": "tutorialdb",
    "time_precision": "s",
    "buffer_size": 8192,
    "duration_num": "6h",
    "duration_log": "3d"
}'
```

Now we have a database and we can use the default database user `iris` with password `siri` to extend the database
with a replica on the second server (running on port `9001`):
```
curl --location --request POST 'http://localhost:9021/new-replica' \
--header 'Content-Type: application/json' \
--header 'Authorization: Basic c2E6c2lyaQ==' \
--header 'Content-Type: text/plain' \
--data-raw '{
    "dbname": "tutorialdb",
    "username": "iris",
    "password": "siri",
    "host": "localhost",
    "port": 9000,
    "pool": 0
}'
```

Ok, now everything is ready to collect data (we configure the other two SiriDB servers later in this tutorial). Before we start the Python script to collect data we must install its dependencies:
```
pip install siridb-connector psutil
```

Start the script. The script accepts arguments which can be viewed with `python mon2siridb.py -h`. If you are following this tutorial then the defaults should be fine.
```
python mon2siridb.py &> mon.log &
```

Let's setup Grafana so we can view what we are collecting. First download and install Grafana:
```
sudo apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/oss/release/grafana_6.7.2_amd64.deb
sudo dpkg -i grafana_6.7.2_amd64.deb
```

And install the Grafana-SiriDB-Datasource plugin:
```
cd /var/lib/grafana/plugins/
sudo git clone https://github.com/SiriDB/grafana-siridb-http-datasource.git /var/lib/grafana/plugins/grafana-siridb-http-datasource
```

Start (or restart) Grafana:
```
sudo systemctl restart grafana-server.service
```

Before we can use the SiriDB datasource, we also need to configure and start the SiriDB HTTP connector.
SiriDB HTTP requires a configuration file. For more information you can view the following Github page:
https://github.com/SiriDB/siridb-http#readme

This will create a basic configuration file which is fine for our tutorial. Note that we connect
to both, the first and second SiriDB server, for redundancy.
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

![Grafana login](/png/grafana-login.png?raw=true)

Sign-in by using username `admin` and password `admin`.

Click on ***Add data source*** to create the SiriDB data source. Fill in the form like in the picture below (use `siri` as password):

![Grafana add data source](/png/grafana-add-data-source.png?raw=true)

Click on ***Save and test*** this should return the message that everything is working!

Go to the menu and click on ***Dashboards*** -> ***Import***

![Grafana menu dashboard import](/png/grafana-menu-dashboard-import.png?raw=true)

Click on ***Upload .json File*** and select the `tutorial-dashboard.json` from this folder.
In the next window you should choose the SiriDB HTTP data source.

![Grafana import dashboard](/png/grafana-import-dashboard.png?raw=true)

After clicking on ***Import*** you should see a dashboard similar to this:

![Grafana tutorial dashboard](/png/grafana-tutorial-dashboard.png?raw=true&v=1)

We can now continue by expanding the database with another pool and use the third server on port `9002`.

```
curl --location --request POST 'http://localhost:9022/new-pool' \
--header 'Content-Type: application/json' \
--header 'Authorization: Basic c2E6c2lyaQ==' \
--header 'Content-Type: text/plain' \
--data-raw '{
    "dbname": "tutorialdb",
    "username": "iris",
    "password": "siri",
    "host": "localhost",
    "port": 9000
}'
```

In the dashboard you should see the new server. The status for the existing servers includes ***re-indexing*** while
the series are spread across the pools.

![Grafana re-indexing](/png/grafana-re-indexing.png?raw=true)

Wait until the status for all three server is ***running*** and then create another replica on the fourth server (on port `9003`):

```
curl --location --request POST 'http://localhost:9023/new-replica' \
--header 'Content-Type: application/json' \
--header 'Authorization: Basic c2E6c2lyaQ==' \
--header 'Content-Type: text/plain' \
--data-raw '{
    "dbname": "tutorialdb",
    "username": "iris",
    "password": "siri",
    "host": "localhost",
    "port": 9000,
    "pool": 1
}'
```

The dashboard should show the fourth server with status ***synchronizing***

![Grafana synchronizing](/png/grafana-synchronizing.png?raw=true)

From this point on it should be rather easy to create your own Grafana Dashboard by using a SiriDB database.
As an example we will add two extra graphs for Disk IO counters.

It might be helpful to test SiriDB queries. We can do this by using the running SiriDB HTTP webserver.

Go to http://localhost:5050. Here you should see the following screen:

![SiriDB HTTP login](/png/siridb-http-login.png?raw=true)

Login by using the default user `iris` with password `siri`.

Now you have a prompt available where you can test queries, for example:

![SiriDB HTTP list series](/png/siridb-http-list-series.png?raw=true)

![SiriDB HTTP select](/png/siridb-http-select.png?raw=true)

If you want to select series based on regular expression, then a best practice is to create a dynamic group.
For the current example we create the following two groups:
```
create group `disk_io_counters_read_bytes` for /.*disk_io_counters_read_bytes/
create group `disk_io_counters_write_bytes` for /.*disk_io_counters_write_bytes/
```

![SiriDB HTTP create group](/png/siridb-http-create-group.png?raw=true)

Go back to Grafana and click on ***Add panel*** -> **Add Query**.

![Grafana graph](/png/grafana-add-panel.png?raw=true&v=1)

At ***select*** fill in ``` `disk_io_counters_read_bytes` ```, choose ***max*** as aggregation and enable ***Diffps***.

![Grafana read bytes](/png/grafana-add-read-bytes.png?raw=true&v=2)

On the General tab you can change the panel title to "Disk IO counters (read bytes)".

![Grafana graph](/png/grafana-add-panel-add-title.png?raw=true&v=1)

Repeat these steps for the ***write*** counters and when you are finished you should have the following result:

![Grafana disk io counters bytes](/png/grafana-disk-io-counters-bytes.png?raw=true)

I hope this tutorial was helpful and I am looking forward to hear what you can create by using Grafana and SiriDB!


This document describes how to use CloudTracker with ElasticSearch.

Requirements
============
* CloudTrail logs must be loaded into ElasticSearch.  For instructions on setting up ElasticSearch and ingesting an archive of CloudTrail logs into it see below.
  * ElasticSearch 6.x is supported, but there are reports of ElasticSearch 1.x being used successfully.

Installation
============

### Step 1
Install the Python libraries using one of the provided Makefile targets:

For elasticsearch v6.x:
```
#sudo apt-get update
#sudo apt-get install python3-venv -y
python3 -m venv ./venv && source venv/bin/activate
pip install git+https://github.com/duo-labs/cloudtracker.git#egg=cloudtracker[es6]
```

For older versions, such as elasticsearch v1.x:
```
python3 -m venv ./venv && source venv/bin/activate
pip install git+https://github.com/duo-labs/cloudtracker.git#egg=cloudtracker[es1]
```

### Step 2
Get the IAM data of the account

```
#mkdir account-data
#sudo apt install awscli
aws iam get-account-authorization-details > account-data/demo_iam.json
```

### Step 3
#Create
Edit the `config.yaml`.  You need to specify how to connect to the ElasticSearch cluster, what index the CloudTrail logs are stored in, and information about your AWS account, including the location of the IAM file created in Step 3.

Example `config.yaml` file:
```
elasticsearch:
  host: localhost
  port: 9200
  index: "cloudtrail"
  key_prefix: ""
  timestamp_field: "eventTime"
accounts:
  - name: demo
    id: 123456789012
    iam: account-data/demo_iam.json
```

The ElasticSearch configuration section works the same as what is available to the ElasticSearch python library documented here: http://elasticsearch-py.readthedocs.io/en/master/api.html#elasticsearch

Additionally, you can configure:

- `index`: The index you loaded your files at.
- `key_prefix`: Any prefix you have to your CloudTrail records.  For example, if your `eventName` is queryable via `my_cloudtrail_data.eventName`, then the `key_prefix` would be `my_cloudtrail_data`.



Install ElasticSearch
=====================

You can use an AWS managed ElasticSearch cluster or one that you manage, including one running locally on a VM on your laptop.  However, if your logs exceed a few dozen GBs, or over 100M records, you'll likely run into issues running locally.  You'll also want to install Kibana to look at the loaded logs.

Configure the ElasticSearch mapping
-----------------------------------
Using Kibana and clicking on "Dev Tools" you can send commands to ElasticSearch. You can also do this using `curl`.  Run the following to setup a `cloudtrail` index and increase it's total fields to 5000.  If you don't increase that limit, records will be silently dropped.

#sudo add-apt-repository ppa:webupd8team/java
#sudo apt update
#sudo apt -y install oracle-java8-installer
#java -version
#sudo apt -y install oracle-java8-set-default
#sudo vi /etc/environment
#JAVA_HOME="/usr/lib/jvm/java-8-oracle/"
#echo $JAVA_HOME
#wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
#echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-6.x.list
#sudo apt update
#sudo apt install elasticsearch
#sudo systemctl start elasticsearch
#sudo systemctl enable elasticsearch
#curl -X GET "localhost:9200"


The commands to send
```
{
    "mappings": {
      "doc": {
        "properties": {
          "@timestamp": {
            "type": "date"
          },
          "@version": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          },
          "host": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          },
          "message": {
            "properties": {
              "additionalEventData": {
                "properties": {
                  "Note": {
                    "type": "text",
                    "fields": {
                      "keyword": {
                        "type": "keyword",
                        "ignore_above": 256
                      }
                    }
                  }
                }
              },
              "apiVersion": {
                "type": "text"
              }
            }
          }
        }
      }
    }
}
```

You can save the above file as `cloudtrail_mapping.json` and then send it to your ElasticSearch cluster using:

```
curl -X PUT https://YOUR_ES_SERVER/cloudtrail -T cloudtrail_mapping.json  -H "Content-Type: application/json"
```


Do the same for:
```
#PUT /cloudtrail/_settings
{
  "index.mapping.total_fields.limit": 5000
}
```

You can save that to a file named `cloudtrail_settings.json` and then run:
```
curl -X PUT https://YOUR_ES_SERVER/cloudtrail/_settings -T cloudtrail_settings.json  -H "Content-Type: application/json"
```


Ingest CloudTrail logs into ElasticSearch using Hindsight
=========================================================

Copy your CloudTrail logs locally and convert them to a single flat file.

```
# Replace YOUR_BUCKET and YOUR_ACCOUNT_ID in the following command
aws s3 sync s3://YOUR_BUCKET/AWSLogs/YOUR_ACCOUNT_ID/CloudTrail/ .
#cd ..
#touch cloudtrail.json
#chmod 777 cloudtrail.json
#cd ubuntu
find . -name "*.json.gz" -exec gunzip -c {} \; | jq -cr '.Records[] | del(.responseElements.endpoint)' >> ../cloudtrail.json
```

I'm deleting `.responseElements.endpoint` because different API calls return an object or a string for that value and ElasticSearch can't handle mixed types, so I just ignore that value since it is of little use.


Install Hindsight
-----------------
Hindsight is hard to install as it has a number of dependencies.  The project is at https://github.com/mozilla-services/hindsight

Here are some notes, but you'll still probably run into trouble.  Help in improving the installation of those projects would be good.

For the dependencies:
```
sudo apt install -y libcurl-devel autoconf automake libtool cmake
#libcurl4-openssl-dev instead of libcurl-devel

git clone https://github.com/mozilla-services/lua_sandbox.git
cd lua_sandbox/home/ubuntu/lua_sandbox_extensions/release/CMakeFiles/CMakeError.log
mkdir release
cd release

cmake -DCMAKE_BUILD_TYPE=release ..
make
#ctest
#cpack -G TGZ # (DEB|RPM|ZIP)
# if error coming try:
#sudo chown ubuntu install_manifest.txt
#sudo chgrp ubuntu install_manifest.txt
sudo make install

cd ../..

git clone https://github.com/mozilla-services/lua_sandbox_extensions.git
cd lua_sandbox_extensions
mkdir release
cd release
#sudo apt-get install cmake gcc g++
#remove 'CMakeCache.txt'
#run compilation again.
# Disable a bunch of extensions when we build this to avoid further dependencies
cmake -DCMAKE_BUILD_TYPE=release -DEXT_aws=off -DEXT_snappy=off -DEXT_bit=on -DEXT_bloom_filter=on -DEXT_circular_buffer=on -DEXT_cjson=on -DEXT_compat=on -DEXT_cuckoo_filter=on -DEXT_elasticsearch=on -DEXT_gcp=off -DEXT_hindsight=on -DEXT_heka=on -DEXT_hyperloglog=on -DEXT_irc=off -DEXT_jose=off -DEXT_kafka=off -DEXT_libinjection=on -DEXT_lfs=on -DEXT_lpeg=on -DEXT_lsb=on -DEXT_maxminddb=off -DEXT_moz_ingest=on -DEXT_moz_logging=on -DEXT_moz_pioneer=on -DEXT_moz_security=on -DEXT_moz_telemetry=on -DEXT_openssl=off -DEXT_papertrail=on -DEXT_parquet=off -DEXT_postgres=off -DEXT_rjson=off -DEXT_sax=on -DEXT_socket=on -DEXT_ssl=off -DEXT_struct=on -DEXT_syslog=on -DEXT_systemd=off -DEXT_xxhash=on -DEXT_zlib=off -DCPACK_GENERATOR=DEB ..
make
make packages
sudo make install
# In my experience I needed to manually install files, or copy or link them, as you should have files named
# `rjson.so` and `ltn12.lua` at `/usr/local/lib/luasandbox/io_modules/`.
```

Now install Hindsight from https://github.com/mozilla-services/hindsight

git clone https://github.com/mozilla-services/hindsight.git
cd hindsight
mkdir release
cd release

# Linux
cmake -DCMAKE_BUILD_TYPE=release -DWITHOUT_OPENSSL=true ..
make
ctest
cpack -G TGZ # (DEB|RPM|ZIP)

Run a proxy
-----------
This may not be needed, but it's helpful, especially when using an AWS managed ElasticSearch cluster.

```
var http = require('http'),
    httpProxy = require('http-proxy');

var proxy = httpProxy.createProxyServer({});

proxy.on('proxyReq', function(proxyReq, req, res, options) {
  console.log("> Proxying: ", req.url);
  proxyReq.setHeader('content-type', 'application/json');
});

proxy.on('proxyRes', function (proxyRes, req, res) {
  console.log("< ", proxyRes.statusCode);
});

var server = http.createServer(function(req, res) {
  proxy.web(req, res, {
    target: 'https://MY_ES_INSTANCE.us-west-2.es.amazonaws.com', secure: false
  });
});

console.log("listening on port 9201")
server.listen(9201);
```

Here you can see I am ignoring any cert errors when making the TLS connection, so you'll need to decide if that is acceptable for your use case.

Run this with:
```
node proxy.js
```


Configure Hindsight
-------------------
This repo includes a `hindsight/run` directory. Copy the `run` directory to your hindsight repo.

Replace `YOUR_FILE` in `run/input/file.cfg` with the full path to your `cloudtrail.json` file.

Replace `127.0.0.1` and the port `9200` in `run/output/elasticsearch_bulk_api.cfg` if you are not running ElasticSearch on your localhost.


Run hindsight
-------------
To run hindsight use:

```
hindsight_cli hindsight.cfg
```

You will also want to run `rm -rf output/*` in between runs to clear out the cached files.
You may need to modify `hindsight.cfg` to tell it the `io_lua_path` and other paths are in `/usr/local/lib/` not `/usr/lib/`


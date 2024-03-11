+++
title = 'Install Elasticsearch 8 on Ubuntu 22.04'
+++
- [Install lasticsearch Cluster on Ubuntu ( 1 Master , 4 data tier )](#install-lasticsearch-cluster-on-ubuntu--1-master--4-data-tier-)
  - [Refereces](#refereces)
  - [Useful Link](#useful-link)
  - [Glossary](#glossary)
  - [Install Prereq](#install-prereq)
    - [Setup hosts file](#setup-hosts-file)
    - [Prepare Elastic Backup Repository (NFS)](#prepare-elastic-backup-repository-nfs)
      - [Install NFS-Common on Ubuntu](#install-nfs-common-on-ubuntu)
      - [Create /var/data/elasticsearch for elasticsearch backup](#create-vardataelasticsearch-for-elasticsearch-backup)
      - [Mount nfs to /etc/fstab](#mount-nfs-to-etcfstab)
  - [Install Elasticsearch ( Master Node )](#install-elasticsearch--master-node-)
    - [Add Elasticsearch 8 Repository](#add-elasticsearch-8-repository)
    - [Install Elastic search](#install-elastic-search)
    - [Change Owner /var/data/elasticsearch to elasticsearch](#change-owner-vardataelasticsearch-to-elasticsearch)
    - [Backup Elasticsearch yaml](#backup-elasticsearch-yaml)
    - [Select Node Role](#select-node-role)
      - [Master, data\_content, data\_hot](#master-data_content-data_hot)
    - [Start Elasticsearch](#start-elasticsearch)
    - [Reset Elasticsearch Password](#reset-elasticsearch-password)
    - [Test Curl, You know for search](#test-curl-you-know-for-search)
  - [Install Elasticsearch ( Other Node eg. data node, ingest node )](#install-elasticsearch--other-node-eg-data-node-ingest-node-)
    - [Add Elasticsearch 8 Repository](#add-elasticsearch-8-repository-1)
    - [Install Elastic search](#install-elastic-search-1)
    - [Change Owner /var/data/elasticsearch to elasticsearch](#change-owner-vardataelasticsearch-to-elasticsearch-1)
    - [Generate Enrollment Token on Master Node](#generate-enrollment-token-on-master-node)
    - [Enrollment From Joinee Node](#enrollment-from-joinee-node)
    - [Select Node Role](#select-node-role-1)
      - [data\_warm](#data_warm)
      - [data\_cold](#data_cold)
      - [data\_frozen](#data_frozen)
    - [Start Elasticsearch](#start-elasticsearch-1)
  - [Install Kibana](#install-kibana)
    - [Install Kibana](#install-kibana-1)
    - [Generate Kibana Token](#generate-kibana-token)
    - [Enroll](#enroll)
    - [Kibeana yaml , listen 0.0.0.0](#kibeana-yaml--listen-0000)
  - [Kibana Dev Tools](#kibana-dev-tools)
    - [Indices Name](#indices-name)
    - [Get Node](#get-node)
    - [Monitor](#monitor)
    - [Insert Data](#insert-data)
    - [Kibana Dev Tools](#kibana-dev-tools-1)
  - [Generate Fleet policy with correct url and port](#generate-fleet-policy-with-correct-url-and-port)
  - [Install Fleet Server](#install-fleet-server)
    - [Elastic Bug , Elastic agent use local cahced file](#elastic-bug--elastic-agent-use-local-cahced-file)
    - [Uninstall Elastic Agent](#uninstall-elastic-agent)
    - [Install Kibana Agent](#install-kibana-agent)
    - [Install Kube-metrics-server](#install-kube-metrics-server)
    - [add --kubelet-insecure-tls to metric server deployment](#add---kubelet-insecure-tls-to-metric-server-deployment)
    - [Install kube-state-metric](#install-kube-state-metric)
    - [Replace Ubuntu Mirror](#replace-ubuntu-mirror)
    - [Download fleet daemonset agent manifest from kibana](#download-fleet-daemonset-agent-manifest-from-kibana)
    - [Create Elastic SSL Cert Configmap](#create-elastic-ssl-cert-configmap)
    - [Create Cert](#create-cert)
    - [Edit username \& password \& kibana url \& insecure flag](#edit-username--password--kibana-url--insecure-flag)
    - [Add CA Cert to RHEL](#add-ca-cert-to-rhel)



# Install lasticsearch Cluster on Ubuntu ( 1 Master , 4 data tier )

## Refereces

[:page_with_curl: **Install**  Elasticsearch 8 on Ubuntu 22.04 LTS](https://blog.devops.dev/how-to-install-elastic-stack-on-ubuntu-22-04-lts-18c3d9120494)

[:tv: **Youtube** : Setup Elasticsearch Cluster + Kibana 8.x](https://www.youtube.com/watch?v=TfhcJXdNSdI)

[:thumbsup: **Workshop** How to Set Up a Hot/Warm/Cold/Frozen Elasticsearch Architecture](https://opster.com/guides/elasticsearch/capacity-planning/elasticsearch-hot-warm-cold-frozen-architecture/)

[:mag: **Info** Kubenetes Monitoring](https://www.elastic.co/blog/kubernetes-cluster-metrics-logs-monitoring)

## Useful Link

[:link: ELK : Partially mounted index](https://www.elastic.co/guide/en/elasticsearch/reference/current/searchable-snapshots.html?fbclid=IwAR0qix2bkw-5zk8GbRhakWGrab2FYtt3croxwXWSr7SX-8b1yZt7oh_gHcU#partially-mounted)

[:link: ELK : Searchable Snapshot ](https://opster.com/guides/elasticsearch/data-architecture/elasticsearch-searchable-snapshots/)

[:link: ELK : Snapshot ](https://opster.com/analysis/elasticsearch-the-specified-location-should-start-with-a-repository-path-specified-by/?fbclid=IwAR3sLxb90WkqlPizvFX6YD_3AMH7NxOb7RiGvoeMsfu3NF5n07cKs8AilCg)

[:link: ELK : Snapshot and Restore](https://www.elastic.co/blog/found-elasticsearch-snapshot-and-restore?fbclid=IwAR0L5JgP8-Me5X8-0HIqpgVOe9L9kVw2IXCaVBhbuFw7h9xOsUBBi4dyjas)

[:link: ELK : How do Elasticsearch snapshots works](https://www.elastic.co/blog/how-do-incremental-snapshots-work?fbclid=IwAR07zwFsSDoV3y6dSmlI53l04gYbukrIppSTx83VA0qrLbxkkPimGkPG5y8)

[:link: ELK : Backup Repository : S3](https://opster.com/guides/elasticsearch/how-tos/elasticsearch-snapshot/?fbclid=IwAR0NTWQdfFXv1xHRkareLn0f5o45SGXBuz1ANiHgOkUYXgt5hZAjTzlTNvM)

[:link: ELK : Backup Repository : NFS](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-filesystem-repository.html)

## Glossary

Query Node

```
curl -XGET --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200/_cat/nodes?pretty;
10.10.51.32 19 92 0 0.02 0.02 0.00 w   - k-gong-elk-02
10.10.51.33  6 90 0 0.00 0.00 0.00 c   - k-gong-elk-03
10.10.51.31 21 87 1 0.16 0.13 0.09 hms * k-gong-elk-01
10.10.51.34  9 91 0 0.01 0.02 0.00 f   - k-gong-elk-04
```

Glossary

```
m: master node
s: content tier
h: hot data tier
w: warm data tier
c: cold data tier
f: frozen data tier
```

## Install Prereq

### Setup hosts file

```
{
echo "
10.10.51.31     k-gong-elk-01
10.10.51.32     k-gong-elk-02
10.10.51.33     k-gong-elk-03
10.10.51.34     k-gong-elk-04" >> /etc/hosts
}
```

### Prepare Elastic Backup Repository (NFS)

#### Install NFS-Common on Ubuntu 

```
{
apt update
apt install -y nfs-common
}
```

#### Create /var/data/elasticsearch for elasticsearch backup

```
{
mkdir -p /var/data/elasticsearch
mkdir -p /var/data/elasticsearch/snapshots
chown -R elasticsearch:elasticsearch /var/data/elasticsearch
}
```

#### Mount nfs to /etc/fstab

```
{
export NFS_IP=10.10.54.8
export NFS_PATH=/var/nfs/gong-elasticsearch-snapshot
export ELASTIC_SNAPSHOT_PATH=/var/data/elasticsearch/snapshots
echo "${NFS_IP}:${NFS_PATH}   ${ELASTIC_SNAPSHOT_PATH}    nfs    defaults,soft    0 0" >> /etc/fstab
mount -a
}
```

## Install Elasticsearch ( Master Node )


### Add Elasticsearch 8 Repository

```
{
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
}
```

### Install Elastic search 

Note : save output text after install

```
{
sudo apt-get update
sudo apt-get install -y elasticsearch=8.12.2
}
```

### Change Owner /var/data/elasticsearch to elasticsearch

```
{
chown -R elasticsearch:elasticsearch /var/data/elasticsearch
}
```

### Backup Elasticsearch yaml

```
{
cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml.bak
}
```

### Select Node Role

#### Master, data_content, data_hot
```
{
export CLUSTER_NAME="gong-cluster"
export INITIAL_MASTER_NODES=k-gong-elk-01
echo "
cluster.initial_master_nodes:
  - ${INITIAL_MASTER_NODES}
discovery.seed_hosts:
  - ${INITIAL_MASTER_NODES}:9300
cluster.name: ${CLUSTER_NAME}
node.roles: [master,data_content,data_hot]
node.name: ${HOSTNAME}
network.host: ${HOSTNAME}
path.data: /var/data/elasticsearch/elasticsearch
path.logs: /var/log/elasticsearch
path.repo: /var/data/elasticsearch/snapshots
###### Allow HTTP API connections from anywhere
###### Connections are encrypted and require user authentication
http.host: 0.0.0.0
###### Allow other nodes to join the cluster from anywhere
###### Connections are encrypted and mutually authenticated
transport.host: 0.0.0.0

###### xpack
xpack.security.enabled: true

xpack.security.enrollment.enabled: true

xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12

xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
" > /etc/elasticsearch/elasticsearch.yml
}
```

### Start Elasticsearch

```
{
systemctl start elasticsearch.service
systemctl status elasticsearch.service
}
```

### Reset Elasticsearch Password

Note : Keep Elasticsearch Password from output 

export ELASTIC_PASSWORD=password

```
{
/usr/share/elasticsearch/bin/elasticsearch-reset-password -i -u elastic
export ELASTIC_PASSWORD="password"
}
```

### Test Curl, You know for search

cert location : /etc/elasticsearch/certs/http_ca.crt

```
{
export ELASTIC_PASSWORD=password
curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200
curl -XGET --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200/_cat/nodes?pretty
}
```


---

## Install Elasticsearch ( Other Node eg. data node, ingest node )

### Add Elasticsearch 8 Repository

```
{
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
}
```

### Install Elastic search 

Note : save output text after install

```
{
sudo apt-get update
sudo apt-get install -y elasticsearch=8.12.2
}
```

### Change Owner /var/data/elasticsearch to elasticsearch

```
{
chown -R elasticsearch:elasticsearch /var/data/elasticsearch
}
```

### Generate Enrollment Token on Master Node

This step must be done on Master node

```
{
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node
}
```

### Enrollment From Joinee Node

This step must be done on Joinee node

```
{
export ENROLLMENT_TOKEN=abcdef
/usr/share/elasticsearch/bin/./elasticsearch-reconfigure-node --enrollment-token ${ENROLLMENT_TOKEN}
}
```

### Select Node Role

#### data_warm
```
{
export CLUSTER_NAME="gong-cluster"
export INITIAL_MASTER_NODES=k-gong-elk-01
echo "
cluster.initial_master_nodes:
  - ${INITIAL_MASTER_NODES}
discovery.seed_hosts:
  - ${INITIAL_MASTER_NODES}:9300
cluster.name: ${CLUSTER_NAME}
node.roles: [data_warm]
node.name: ${HOSTNAME}
network.host: ${HOSTNAME}
path.data: /var/data/elasticsearch/elasticsearch
path.logs: /var/log/elasticsearch
path.repo: /var/data/elasticsearch/snapshots
###### Allow HTTP API connections from anywhere
###### Connections are encrypted and require user authentication
http.host: 0.0.0.0
###### Allow other nodes to join the cluster from anywhere
###### Connections are encrypted and mutually authenticated
transport.host: 0.0.0.0

###### xpack
xpack.security.enabled: true

xpack.security.enrollment.enabled: true

xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12

xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
" > /etc/elasticsearch/elasticsearch.yml
}
```

#### data_cold
```
{
export CLUSTER_NAME="gong-cluster"
export INITIAL_MASTER_NODES=k-gong-elk-01
echo "
cluster.initial_master_nodes:
  - ${INITIAL_MASTER_NODES}
discovery.seed_hosts:
  - ${INITIAL_MASTER_NODES}:9300
cluster.name: ${CLUSTER_NAME}
node.roles: [data_cold]
node.name: ${HOSTNAME}
network.host: ${HOSTNAME}
path.data: /var/data/elasticsearch/elasticsearch
path.logs: /var/log/elasticsearch
path.repo: /var/data/elasticsearch/snapshots
###### Allow HTTP API connections from anywhere
###### Connections are encrypted and require user authentication
http.host: 0.0.0.0
###### Allow other nodes to join the cluster from anywhere
###### Connections are encrypted and mutually authenticated
transport.host: 0.0.0.0

###### xpack
xpack.security.enabled: true

xpack.security.enrollment.enabled: true

xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12

xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
" > /etc/elasticsearch/elasticsearch.yml
}
```

#### data_frozen
```
{
export CLUSTER_NAME="gong-cluster"
export INITIAL_MASTER_NODES=k-gong-elk-01
echo "
cluster.initial_master_nodes:
  - ${INITIAL_MASTER_NODES}
discovery.seed_hosts:
  - ${INITIAL_MASTER_NODES}:9300
cluster.name: ${CLUSTER_NAME}
node.roles: [data_frozen]
node.name: ${HOSTNAME}
network.host: ${HOSTNAME}
path.data: /var/data/elasticsearch/elasticsearch
path.logs: /var/log/elasticsearch
path.repo: /var/data/elasticsearch/snapshots
###### Allow HTTP API connections from anywhere
###### Connections are encrypted and require user authentication
http.host: 0.0.0.0
###### Allow other nodes to join the cluster from anywhere
###### Connections are encrypted and mutually authenticated
transport.host: 0.0.0.0

###### xpack
xpack.security.enabled: true

xpack.security.enrollment.enabled: true

xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12

xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
" > /etc/elasticsearch/elasticsearch.yml
}
```

### Start Elasticsearch

```
{
systemctl start elasticsearch.service
systemctl status elasticsearch.service
}
```

---

## Install Kibana

### Install Kibana


```
{
sudo apt-get install -y kibana=8.12.2
}
```


### Generate Kibana Token

Note : Save Token from output

Token : eyJ2ZXIiOiI4LjEyLjIiLCJhZHIiOlsiMTAuMTAuNTEuMzE6OTIwMCJdLCJmZ3IiOiJjNTE2MTQ5ZTZiOWMxNDM2Y2I4YTQ3NjZmYWIwMTUyOWU4OWNlOTdhNjJhNTVhNTdlYzQyMzUwZTYzZDIxMTIxIiwia2V5IjoiM2VrRERvNEJ6QWQ2cm1hdDItbnU6b094S0xyYWNTYmUwTmg0U1pFWHFjQSJ9

```
{
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
}
```

### Enroll


```
{
export KIBANA_ENROLL_KEY=eyJ2ZXIiOiI4LjEyLjIiLCJhZHIiOlsiMTAuMTAuNTEuMzE6OTIwMCJdLCJmZ3IiOiJjNTE2MTQ5ZTZiOWMxNDM2Y2I4YTQ3NjZmYWIwMTUyOWU4OWNlOTdhNjJhNTVhNTdlYzQyMzUwZTYzZDIxMTIxIiwia2V5IjoiM2VrRERvNEJ6QWQ2cm1hdDItbnU6b094S0xyYWNTYmUwTmg0U1pFWHFjQSJ9
cd /usr/share/kibana/
./bin/kibana-setup --enrollment-token ${KIBANA_ENROLL_KEY}
}
```


### Kibeana yaml , listen 0.0.0.0

```
{
sed -i -Ee '/^#server.host:/s/^.*$/server.host: 0.0.0.0/' /etc/kibana/kibana.yml
sudo systemctl restart kibana
}
```

---

## Kibana Dev Tools

### Indices Name

```
{
echo "xXx
RFGcXL_EQ2eoP8NzvLo87A
vTtRO_4mSvmr-LHmw7nheQ
VYPwuajsSreKQ2th3Y7Ihw
nIZy_RGUQPWCXv5zUoVYLA
xXx" > /tmp/indices_name
find /var/data/elasticsearch -maxdepth 3 -type d | grep -f /tmp/indices_name
}
```

### Get Node
```
{
export ELASTIC_PASSWORD=password
curl -XGET --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200/_cat/nodes?pretty
}
```

### Monitor
```
{
export ELASTIC_PASSWORD=password
while [ true ] ; do
echo "==== $(date) ==========="
curl -XGET --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200/_cat/shards/orders*\?v\&s=index\&h=index,node
sleep 30
done
}
```

### Insert Data
```
{
export ELASTIC_PASSWORD=password
for ((i=1; i<=999999; i++))
do
echo "{\"@timestamp\":\"$(date +%s)\",\"client_id\":\"${i}\"}"
curl -XPOST -H "Content-Type: application/json" -d "{\"@timestamp\":\"$(date +%s)\",\"client_id\":\"${i}\"}" --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200/orders/_doc/
sleep 5
done
}
```


### Kibana Dev Tools


```
GET _cat/shards/orders*?v&s=index&h=index,node

GET _cat/indices/orders*?v&s=index&h=index,uuid
GET _cat/indices/orders*?v&s=index&h=uuid

PUT _index_template/orders
{
  "index_patterns": ["orders-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0
    },
    "mappings": {
      "properties": {
        "client_id": {
          "type": "integer"
        },
        "created_at": {
          "type": "date"
        }
      }
    }
  }
}

# Today
PUT orders-20211020

GET orders-20211020

DELETE orders-20211020

# Last 30 days
PUT orders-20211010

GET orders-20211010

DELETE orders-20211010

# Last 365 days
PUT orders-20210601

GET orders-20210601

DELETE orders-20210601

# Older than 365 days
PUT orders-20200601

GET orders-20200601

DELETE orders-20200601

DELETE orders-20200601t_recovered

GET orders-20200601/_search

POST orders-20200601/_doc
{
    "msg" : "should be frozen"
}

# ======================================
# Migrate Index
# ======================================

PUT orders-20211020/_settings
{
  "index.routing.allocation.include._tier_preference": "data_hot"
}
 
PUT orders-20211010/_settings
{
  "index.routing.allocation.include._tier_preference": "data_warm"
}
 
PUT orders-20210601/_settings
{
  "index.routing.allocation.include._tier_preference": "data_cold"
}

PUT orders-20200601/_settings
{
  "index.routing.allocation.include._tier_preference": "data_frozen"
}


# ======================================
# Create NFS Backup Repositories
# ======================================

# chown -R elasticsearch:elasticsearch /var/data/elasticsearch/snapshots
PUT _snapshot/orders-snapshots-repository
{
  "type": "fs",
  "settings": {
    "location": "/var/data/elasticsearch/snapshots"
  }
}

DELETE _snapshot/orders-snapshots-repository

# ======================================
# Create a snapshot
# ======================================

PUT _snapshot/orders-snapshots-repository/snapshot-orders-20200601-01
{
  "indices": "orders-20200601"
}

DELETE orders-20200601

POST _snapshot/orders-snapshots-repository/snapshot-orders-20200601-01/_restore

# ======================================
# Mount as searchable snapshot
# ======================================

POST /_snapshot/orders-snapshots-repository/snapshot-orders-20200601-01/_mount?wait_for_completion=true
{
  "index": "orders-20200601",
  "renamed_index": "orders-20200601t_recovered",
  "index_settings": { 
    "index.number_of_replicas": 0,
    "index.routing.allocation.include._tier_preference": "data_frozen"
  }
}

POST /_snapshot/orders-snapshots-repository/snapshot-orders-20200601-01/_mount?storage=shared_cache
{
  "index": "orders-20200601",
  "renamed_index": "orders-20200601t_recovered",
  "index_settings": { 
    "index.number_of_replicas": 0,
    "index.routing.allocation.include._tier_preference": "data_frozen"
  }
}

# ====================================================
# orders slm (snapshot lifecycle management)
# ====================================================

PUT _slm/policy/orders-snapshot-policy
{
  "schedule": "0 */15 * * * ?", 
  "name": "<orders-snapshot-{now/d}>", 
  "repository": "orders-snapshots-repository", 
  "config": { 
    "indices": ["orders"] 
  },
  "retention": { 
    "expire_after": "30d", 
    "min_count": 5, 
    "max_count": 50 
  }
}

DELETE _slm/policy/orders-snapshot-policy

# ====================================================
# Enable Kibana Trial 30 Days
# ====================================================
# Note : https://www.elastic.co/subscriptions
GET _license

POST _license/start_trial?acknowledge=true

# ====================================================
# orders ilm (index lifecycle management)
# ====================================================

PUT _ilm/policy/orders-lifecycle-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "30s"
          }
        }
      },
      "warm": {
        "min_age": "1m",
        "actions": {}
      },
      "cold": {
        "min_age": "2m",
        "actions": {}
      },
      "frozen": {
        "min_age": "3m",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "orders-snapshots-repository"
          }
        }
      },
      "delete": {
        "min_age": "3m",
        "actions": {
          "wait_for_snapshot": {
            "policy": "orders-snapshot-policy"
          },
          "delete": {}
        }
      }
    }
  }
}

DELETE _ilm/policy/orders-lifecycle-policy

# ==============================================================
# By default ILM checks every 10 minutes, reduce to 15 sec
# ==============================================================

PUT _cluster/settings
{
  "persistent": {
    "indices.lifecycle.poll_interval": "15s" 
  }
}

PUT /_cluster/settings
{
"transient": {"indices.lifecycle.poll_interval": "15s"}
}

GET _cluster/settings

# ==============================================================
# Assign ILM
# ==============================================================

PUT orders-20211020/_settings
{
  "index.lifecycle.name": "orders-lifecycle-policy"
}

# ==============================================================
# Create Index Template Components, Type : Settings
# ==============================================================
PUT _component_template/orders-settings
{
  "template": {
    "settings": {
      "index.lifecycle.name": "orders-lifecycle-policy",
      "number_of_shards": 1,
      "number_of_replicas": 0
    }
  },
  "_meta": {
    "description": "Settings for orders indices"
  }
}

DELETE _component_template/orders-settings
 
# ==============================================================
# Create Index Template Components, Type : Mapping
# ==============================================================
PUT _component_template/orders-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date",
          "format": "date_optional_time||epoch_millis"
        },
        "client_id": {
          "type": "integer"
        },
        "created_at": {
          "type": "date"
        }
      }
    }
  },
  "_meta": {
    "description": "Mappings for orders indices"
  }
}

DELETE _component_template/orders-mappings

# ==============================================================
# Create Index Template , Combine Setting & Mapping 
# ==============================================================
PUT _index_template/orders
{
  "index_patterns": ["orders"],
  "data_stream": { },
  "composed_of": [ "orders-settings", "orders-mappings" ],
  "priority": 500,
  "_meta": {
    "description": "Template for orders data stream"
  }
}

DELETE _index_template/orders

# ==============================================================
# Shell to Monitor
# ==============================================================
# clear ; watch -d curl -s 'http://localhost:9200/_cat/shards/orders*\?v\&s=index\&h=index,node'


# ==============================================================
# Create Data Stream
# ==============================================================
PUT _data_stream/orders

POST orders/_doc/
{"@timestamp":"1710073862","client_id":"1"}

DELETE _data_stream/orders

GET orders/_search

POST /orders/_rollover
```

---

## Generate Fleet policy with correct url and port

fleet server name : fleet-01

fleet URL : https://10.10.51.31:8220

## Install Fleet Server

Generate Install Fleet Server from kibana

Add Flag --insecure at the end

```
{
wget http://10.10.54.4/elastic-agent/elastic-agent-8.11.0-linux-x86_64.tar.gz
tar -xzvf elastic-agent-8.11.0-linux-x86_64.tar.gz
cd elastic-agent-8.11.0-linux-x86_64
sudo ./elastic-agent install \
  --fleet-server-es=https://10.10.51.31:9200 \
  --fleet-server-service-token=AAEAAWVsYXN0aWMvZmxlZXQtc2VydmVyL3Rva2VuLTE3MDI4MzczNDk2MTA6RE9FMG90aVhSRC1ra3BFQVo1dnBFdw \
  --fleet-server-policy=fleet-server-policy \
  --fleet-server-es-ca-trusted-fingerprint=357ff7ad1c13e25ff3111574918de78f3bc21d9bd6696150b93cb36b4cdf174e \
  --fleet-server-port=8220 --insecure
}
```

### Elastic Bug , Elastic agent use local cahced file

https://github.com/elastic/elastic-agent/issues/3586

fix by : rm -rf /var/lib/elastic-agent-managed/kube-system/state



---


### Uninstall Elastic Agent

```
{
sudo /opt/Elastic/Agent/elastic-agent uninstall
}
```

### Install Kibana Agent

```
{
wget http://10.10.54.4/elastic-agent/elastic-agent-8.11.3-linux-x86_64.tar.gz
tar xzvf elastic-agent-8.11.3-linux-x86_64.tar.gz
cd elastic-agent-8.11.3-linux-x86_64
sudo ./elastic-agent install --url=https://10.10.51.31:8220 --enrollment-token=dDV5VGVJd0I1U2Qyd0ZxWEp3MEo6NU9VbERSdWxRM0dXaGtyUE1ZMFhkUQ== \
--insecure
}
```

tar xzvf elastic-agent-8.11.3-linux-x86_64.tar.gz
cd elastic-agent-8.11.3-linux-x86_64
sudo ./elastic-agent install --url=https://10.10.51.31:8220 --enrollment-token=cndZS2Q0d0JDbmY5RUJWRUQ5Nk86QWE2eHV4VUNTV09Jb3VpTHY5OGhsUQ== --insecure

### Install Kube-metrics-server

```
{
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
}
```

### add --kubelet-insecure-tls to metric server deployment 

k edit -n=kube-system deployments.apps metrics-server

```
{
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
}
```

---

### Install kube-state-metric

```
{
git clone https://github.com/kubernetes/kube-state-metrics.git
cd kube-state-metrics/examples/
kubectl apply -f standard
}
```


---

### Replace Ubuntu Mirror
```
{
sed -i 's/mirror1.ku.ac.th/mirrors.nipa.cloud/g' /etc/apt/sources.list
}
```
---

### Download fleet daemonset agent manifest from kibana

login into kibana & download fleeat daemonset agent manifest


### Create Elastic SSL Cert Configmap

### Create Cert

{
kubectl delete configmap elastic-ca-cert
cd ~
echo '-----BEGIN CERTIFICATE-----
MIIFWjCCA0KgAwIBAgIVAPYBdw7JTeakx1BviZavVysiO4pjMA0GCSqGSIb3DQEB
CwUAMDwxOjA4BgNVBAMTMUVsYXN0aWNzZWFyY2ggc2VjdXJpdHkgYXV0by1jb25m
aWd1cmF0aW9uIEhUVFAgQ0EwHhcNMjMxMjE3MTgxNjExWhcNMjYxMjE2MTgxNjEx
WjA8MTowOAYDVQQDEzFFbGFzdGljc2VhcmNoIHNlY3VyaXR5IGF1dG8tY29uZmln
dXJhdGlvbiBIVFRQIENBMIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEA
pwP6wx97mZIyk+9Uoemw+eWLR637vA18NTtUIh0FePWXg2I4cj6QwXVixf3/ZtS6
//KtyrWWoTf4qc573p1IRl5GoleDOV9JBok2d1clkJVINw4EIahNAQppiybnyZn6
azzB06WQjf6r7EkXIlg7di2/l62EfEJ7GOOmf6WnDjJL/JHIY8b0cBTcQFRxcilz
4jmbSgjAgG3qZktyK0bj9fGSGcdQ/VMPpPnZ6JqFlOq2AxRnoohlUWYrznMYv100
oU9OLg0sGg6BEaJ7sgMXhv1oYlXdzwNExAazAzQO9ef9YYQQDHcICcnnyQmrSEPV
4yQsfS02qK9edk2gyDJzyWPhAInlEd9YLHD6qqpTnq1+EpBI5GZiiGPat3AmQWsP
g1uMudshpQuHcHHX6ZkBFG9Rlv0HLz7mBshn3q+0Jpq9zJzJi7/26s2wwilZbSeS
tKWEOs6HpdEkfe+fM6pklgAqupQuK39PgDLjPSniCNbRRq/HZtkCmO0EIRLaledb
NsZvYPdZrWVWOEwXBxnkRugwhn43aLyFSYSzcncdPsXkNDYk0DRUgwzPWp5vuzE0
G79S8z0IcCpPkLbod3Mg61MD8yxjoCT59haopLELkl9S69uPq/+JKZTUovriIp2a
fUlmU0cxw452rCKPF3iw9orhIMhrfUOPwt4r9uGuaJUCAwEAAaNTMFEwHQYDVR0O
BBYEFJ4eQjX9rV9NkwOyrxQiurba8nb5MB8GA1UdIwQYMBaAFJ4eQjX9rV9NkwOy
rxQiurba8nb5MA8GA1UdEwEB/wQFMAMBAf8wDQYJKoZIhvcNAQELBQADggIBAHyM
1+iY8rFSTu2gRpAWJbz3D56YLxRrvyhx8smIZcp+hnGGV7dUtE5Zt6ExnXavWSNX
kYvixTix/3dvB7mhqz6KHA/tOPBGVo6ELTHLiEicMnbYoEqcPuj4I8z8Dpj0ZeDd
gJPccevk7PcUvy9fekkcA9FJ1RAuPh53Ts+DYjR1Qvx3dxvggkCDvU5oE4o0LX3M
C0OtK+HJzznAA41hwYRw140Ias9roVSEv8Rbsq+AQ8fPdHQyYOqM1HWnmMhozYVt
sC3LtUlQ6SJk8exK9T5EokSHGwCfgDgCzMs2vx5kZVsXT2YcwsMTZOHGBDtjyOSE
W5jFc0D2aWHkdsPqrYcyp4sPrbl9J3AoKZ0vy6XRj7w/K0v3MnkPY4NKJe193J2V
3JamOguNqOWE4s/Jb7l2yHq8e7YcXJQg5YMvEHukcpnt0buLkdo6oLZI8s4x1Hh/
Wo6BsXTksjQgspRQiuyp7yMZWiwDPBTezyaM5yyHL8hfYW8hAHNp6gcBRDjUa7YG
/xIDzKnbZhyOk2XeraTzT2+z1V/ItE3DYHCKYXUI/eAsmR+2LiUy0pn4L0sqCBEA
MlmByisDlPElh0DOBkZ40X+Y2l6nZ5PKSsULS6uI1g5uiTeUuXSuBzGK5c8pytKM
xfxiLf0x+20bo7bmrEvEsftWVy0v53vMhNEQl03Y
-----END CERTIFICATE-----' > ~/elastic-ca.crt
export CA_CERT_FILE=~/elastic-ca.crt
echo "==== Certificate : Subject ===="
openssl x509 -noout -text -in ${CA_CERT_FILE} | grep 'Subject:';
echo " "
kubectl create configmap elastic-ca-cert --from-file=${CA_CERT_FILE} --namespace=kube-system
kubectl get configmap elastic-ca-cert
}

### Edit username & password & kibana url & insecure flag

edit ds manifest

```
            - name: FLEET_INSECURE
              value: "true"
            # Fleet Server URL to enroll the Elastic Agent into
            # FLEET_URL can be found in Kibana, go to Management > Fleet > Settings
            - name: FLEET_URL
              value: "https://10.10.51.31:8220"
            # Elasticsearch API key used to enroll Elastic Agents in Fleet (https://www.elastic.co/guide/en/fleet/current/fleet-enrollment-tokens.html#fleet-enrollment-tokens)
            # If FLEET_ENROLLMENT_TOKEN is empty then KIBANA_HOST, KIBANA_FLEET_USERNAME, KIBANA_FLEET_PASSWORD are needed
            - name: FLEET_ENROLLMENT_TOKEN
              value: "cS1yblFvd0JKSTFmNnVwWHRVSkg6TW5oa2hfWGtRZE84aUhuM0I0eGUxZw=="
            - name: KIBANA_HOST
              value: "http://10.10.51.31:5601/"
            # The basic authentication username used to connect to Kibana and retrieve a service_token to enable Fleet
            - name: KIBANA_FLEET_USERNAME
              value: "elastic"
            # The basic authentication password used to connect to Kibana and retrieve a service_token to enable Fleet
            - name: KIBANA_FLEET_PASSWORD
              value: "9H9o4-I1=c8SEykmLR8*"
```

```
      volumes:
        - name: elastic-ca-cert
          configMap:
            name: elastic-ca-cert
```

```
          volumeMounts:
            - mountPath: /etc/ssl/certs/elastic-ca.crt
              subPath: elastic-ca.crt
              name: elastic-ca-cert
```

---

### Add CA Cert to RHEL

{
chmod 660 /tmp/http_ca.crt 
cp /tmp/http_ca.crt /etc/pki/ca-trust/source/anchors/
cd /etc/pki/ca-trust/source/anchors/
update-ca-trust enable
update-ca-trust extract
}


---


{
du -sh /var/log/pods
find /var/log/pods/ -name "*.log" -exec cp /dev/null {} \;
find /var/log/pods/ -name "*log*" -exec cp /dev/null {} \;
find /var/log/pods/ -name "*log*.gz" -exec rm -f {} \;
du -sh /var/log/pods
}

---

hey -d hello-gong -c 10 -z 10s http://log-debug:8080
hey -D loadtest.txt -c 100 -z 5m  http://log-debug:8080
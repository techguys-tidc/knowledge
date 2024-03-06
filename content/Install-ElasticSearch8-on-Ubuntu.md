- [Install Elasticsearch Cluster on Ubuntu](#install-elasticsearch-cluster-on-ubuntu)
  - [Ref](#ref)
  - [Glossary](#glossary)
  - [Install Elasticsearch](#install-elasticsearch)
    - [Setup hosts file](#setup-hosts-file)
    - [Add Elasticsearch 8 Repository](#add-elasticsearch-8-repository)
    - [Install Elastic search](#install-elastic-search)
    - [Create /var/data](#create-vardata)
    - [Backup Elasticsearch yaml](#backup-elasticsearch-yaml)
    - [Select Node Role](#select-node-role)
      - [Master, data\_content, data\_hot](#master-data_content-data_hot)
    - [Start Elasticsearch](#start-elasticsearch)
    - [Reset Elasticsearch Password](#reset-elasticsearch-password)
    - [Test Curl, You know for search](#test-curl-you-know-for-search)
  - [Join other Node](#join-other-node)
    - [Add Elasticsearch 8 Repository](#add-elasticsearch-8-repository-1)
    - [Install Elastic search](#install-elastic-search-1)
    - [Create /var/data](#create-vardata-1)
    - [Generate Enrollment Token on Master Node](#generate-enrollment-token-on-master-node)
    - [Reconfigure joinee node](#reconfigure-joinee-node)
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



# Install Elasticsearch Cluster on Ubuntu

## Ref

[:page_with_curl: **Install**  Elasticsearch 8 on Ubuntu 22.04 LTS](https://blog.devops.dev/how-to-install-elastic-stack-on-ubuntu-22-04-lts-18c3d9120494)

[:tv: **Youtube** : Setup Elasticsearch Cluster + Kibana 8.x](https://www.youtube.com/watch?v=TfhcJXdNSdI)

[:thumbsup: **Workshop** How to Set Up a Hot/Warm/Cold/Frozen Elasticsearch Architecture](https://opster.com/guides/elasticsearch/capacity-planning/elasticsearch-hot-warm-cold-frozen-architecture/)

[:mag: **Info** Kubenetes Monitoring](https://www.elastic.co/blog/kubernetes-cluster-metrics-logs-monitoring)


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

## Install Elasticsearch

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

### Create /var/data

```
{
mkdir -p /var/data/elasticsearch
rm -rf /var/data/elasticsearch/*
chown -R elasticsearch:elasticsearch /var/data
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

## Join other Node

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

### Create /var/data

```
{
mkdir -p /var/data/elasticsearch
rm -rf /var/data/elasticsearch/*
chown -R elasticsearch:elasticsearch /var/data
}
```


### Generate Enrollment Token on Master Node

```
{
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node
}
```

### Reconfigure joinee node

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

+++
title = 'Install Loki and Promtail on Ubuntu 22.04'
+++

- [References](#references)
- [Lab Access](#lab-access)
  - [Grafana](#grafana)
  - [Loki Push](#loki-push)
- [Install Loki](#install-loki)
  - [Install Loki](#install-loki-1)
  - [Prepare Loki Configuration](#prepare-loki-configuration)
  - [Create Services](#create-services)
  - [Start Services](#start-services)
- [Install Promtail](#install-promtail)
  - [Install Promtail](#install-promtail-1)
  - [Prepare Confinguration](#prepare-confinguration)
  - [Create Services](#create-services-1)
  - [Start Services](#start-services-1)
  - [Experiment Configuration](#experiment-configuration)
    - [Extrace Regex environment1 \& environment2](#extrace-regex-environment1--environment2)


# References 

[:link: Install Loki](https://psujit775.medium.com/how-to-setup-loki-in-ubuntu-20-04-f7aab49910fc)

[:link: Install Promtail](https://psujit775.medium.com/how-to-setup-promtail-in-ubuntu-20-04-ed652b7c47c3)


# Lab Access

## Grafana

| Desc | Value |
|---|---|
| URL | http://10.10.51.41:3000/ |
| Username | admin |
| Password | password |

## Loki Push

| Desc | Value |
|---|---|
| URL | http://10.10.51.41:3100/loki/api/v1/push |


# Install Loki

## Install Loki

```
{
apt install -y unzip
curl -O -L "https://github.com/grafana/loki/releases/download/v2.9.5/loki-linux-amd64.zip"
unzip "loki-linux-amd64.zip"
chmod a+x "loki-linux-amd64"
sudo cp loki-linux-amd64 /usr/local/bin/loki
loki --version
}
```

## Prepare Loki Configuration

```
{
sudo useradd --system loki
sudo mkdir -p /etc/loki /etc/loki/logs
echo "auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /etc/loki
  storage:
    filesystem:
      chunks_directory: /etc/loki/chunks
      rules_directory: /etc/loki/rules
  replication_factor: 1
  ring:
    instance_addr: 0.0.0.0
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093">/etc/loki/loki-local-config.yaml
sudo chown -R loki: /etc/loki
}
```

## Create Services

```
{
echo "[Unit] 
Description=Loki service 
After=network.target 
 
[Service] 
Type=simple 
User=loki 
ExecStart=/usr/local/bin/loki -config.file /etc/loki/loki-local-config.yaml 
Restart=on-failure 
RestartSec=20 
StandardOutput=append:/etc/loki/logs/loki.log 
StandardError=append:/etc/loki/logs/loki.log 
 
[Install] 
WantedBy=multi-user.target">/etc/systemd/system/loki.service
}
```

## Start Services

```
{
sudo systemctl daemon-reload
sudo systemctl start loki
sudo systemctl status loki
sudo systemctl restart loki
sudo systemctl enable loki.service
curl http://localhost:3100/metrics
}
```

# Install Promtail

## Install Promtail

```
{
apt install -y unzip
curl -O -L "https://github.com/grafana/loki/releases/download/v2.9.5/promtail-linux-amd64.zip"
unzip "promtail-linux-amd64.zip"
chmod a+x "promtail-linux-amd64"
sudo cp promtail-linux-amd64 /usr/local/bin/promtail
promtail --version
}
```

## Prepare Confinguration

```
{
export LOKI_IPADDRESS=localhost
sudo mkdir -p /etc/promtail /etc/promtail/logs
echo "server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /etc/promtail/positions.yaml

clients:
  - url: http://${LOKI_IPADDRESS}:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log

  - job_name: nginx
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx
          __path__: /var/log/nginx/*.log" > /etc/promtail/promtail-config.yaml
}
```


## Create Services

```
{
echo "[Unit] 
Description=Promtail service 
After=network.target 
 
[Service] 
Type=simple 
User=root 
ExecStart=/usr/local/bin/promtail -config.file /etc/promtail/promtail-config.yaml 
Restart=on-failure 
RestartSec=20 
StandardOutput=append:/etc/promtail/logs/promtail.log 
StandardError=append:/etc/promtail/logs/promtail.log 
 
[Install] 
WantedBy=multi-user.target">/etc/systemd/system/promtail.service
}
```

## Start Services

```
{
sudo systemctl daemon-reload
sudo systemctl start promtail
sudo systemctl status promtail
sudo systemctl restart promtail
sudo systemctl enable promtail.service
}
```

---

## Experiment Configuration
### Extrace Regex environment1 & environment2

regexr : https://regex101.com/r/0mnhGs/1

```
{
export LOKI_IPADDRESS=localhost
export REGEX_TIME="^(?s)(?P<time>\\S+?) (?P<ip>\\S+?) (?P<content>.*)$"
export ESCAPED_REGEX_TIME=$(printf '%s\n' "${REGEX_TIME}" | sed -e 's/[\/&]/\\&/g')
sudo mkdir -p /etc/promtail /etc/promtail/logs
echo "server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /etc/promtail/positions.yaml

clients:
  - url: http://${LOKI_IPADDRESS}:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/gong.log
    pipeline_stages:
      - regex:
          expression: '(?P<environment>.*)-(?P<environment2>.*).*'
      - labels:
          environment:
          environment2:" > /home/nattawat.ujj/promtail-config.yaml
systemctl restart promtail
sleep 3
systemctl status promtail
echo "GONG1-$(date +%Y%m%d.%H:%M:%S)" >> /var/log/gong.log
}
```



+++
title = 'Sent-PFSense-to-rsyslog-promtail-loki'
+++



# References 

[:link: Using Rsyslog and Promtail to relay syslog messages to Loki](https://alexandre.deverteuil.net/post/syslog-relay-for-loki/)

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

# Setup Ubuntu rsyslogd and promtail

## Setup rsyslogd configuration

### Set rsyslogd relay

```
{
echo '# https://www.rsyslog.com/doc/v8-stable/concepts/multi_ruleset.html#split-local-and-remote-logging
ruleset(name="remote"){
  # https://www.rsyslog.com/doc/v8-stable/configuration/modules/omfwd.html
  # https://grafana.com/docs/loki/latest/clients/promtail/scraping/#rsyslog-output-configuration
  action(type="omfwd" Target="localhost" Port="1514" Protocol="tcp" Template="RSYSLOG_SyslogProtocol23Format" TCP_Framing="octet-counted")
}


# https://www.rsyslog.com/doc/v8-stable/configuration/modules/imudp.html
module(load="imudp")
input(type="imudp" port="514" ruleset="remote")

# https://www.rsyslog.com/doc/v8-stable/configuration/modules/imtcp.html
module(load="imtcp")
input(type="imtcp" port="514" ruleset="remote")
' > /etc/rsyslog.d/00-promtail-relay.conf
systemctl restart rsyslog
sleep 3
systemctl status rsyslog
}
```

## Setup promtail configuration
### Extract Regex to label environment1 & environment2

regexr : https://regex101.com/r/0mnhGs/1

```
{
export LOKI_IPADDRESS=localhost
#export REGEX_TIME="^(?s)(?P<time>\\S+?) (?P<ip>\\S+?) (?P<content>.*)$"
#export ESCAPED_REGEX_TIME=$(printf '%s\n' "${REGEX_TIME}" | sed -e 's/[\/&]/\\&/g')
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


### Extract Regex to label environment1 & environment2



```
{
#export LOKI_IPADDRESS=localhost
#export REGEX_TIME="^(?s)(?P<time>\\S+?) (?P<ip>\\S+?) (?P<content>.*)$"
#export ESCAPED_REGEX_TIME=$(printf '%s\n' "${REGEX_TIME}" | sed -e 's/[\/&]/\\&/g')
sudo mkdir -p /etc/promtail /etc/promtail/logs
echo "server:
  http_listen_port: 9081
  grpc_listen_port: 0

positions:
  filename: /etc/promtail/promtail-syslog-positions.yml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
  - job_name: pfsense-syslog
    syslog:
      listen_address: 0.0.0.0:1514
      labels:
        job: pfsense-syslog
    relabel_configs:
      - source_labels: [__syslog_message_hostname]
        target_label: host
      - source_labels: [__syslog_message_hostname]
        target_label: hostname
      - source_labels: [__syslog_message_severity]
        target_label: level
      - source_labels: [__syslog_message_app_name]
        target_label: application
      - source_labels: [__syslog_message_facility]
        target_label: facility
      - source_labels: [__syslog_connection_hostname]
        target_label: connection_hostname
" > /home/nattawat.ujj/promtail-config.yaml
systemctl restart promtail
sleep 3
systemctl status promtail
}
```
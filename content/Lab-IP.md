
+++
title = 'Lab IP'
+++

- [Lab IP](#lab-ip)
  - [Vcenter](#vcenter)
    - [Host File](#host-file)
  - [LGTM](#lgtm)
    - [Grafana](#grafana)
    - [Loki](#loki)
  - [Prometheus](#prometheus)
    - [Prometheus](#prometheus-1)
  - [InfluxDB](#influxdb)
    - [InfluxDB](#influxdb-1)


# Lab IP

## Vcenter

### Host File

Append Host Files

```
10.98.20.198    vc01.nx.local
```

| Desc | Value |
|---|---|
| URL | https://vc01.nx.local/ |

## LGTM

### Grafana

| Desc | Value |
|---|---|
| URL | http://10.10.51.41 |
| Username | admin |
| Password | n/a |

Password not shown


### Loki

| Desc | Value |
|---|---|
| URL | http://10.10.51.41:3100/loki/api/v1/push |

## Prometheus

### Prometheus

| Desc | Value |
|---|---|
| URL | http://10.10.51.41:9090/ |

## InfluxDB

### InfluxDB


| Desc | Value |
|---|---|
| URL | http://10.10.51.42:8086 |
| Username | admin |
| Password | password |
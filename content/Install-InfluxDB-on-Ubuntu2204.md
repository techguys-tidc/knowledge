+++
title = 'Install-InfluxDB-on-Ubuntu2204'
+++



# References 

[How to Install InfluxDB on Ubuntu 22.04](https://vitux.com/how-to-install-influxdb-on-ubuntu/)

# Useful Links

[:link: Downloads InfluxDB](https://www.influxdata.com/downloads/)

[:link: Install InfluxDB](https://docs.influxdata.com/influxdb/v2/install/)

[:link: InfluxDB v2 Documents](https://docs.influxdata.com/influxdb/v2/)



# Lab Access

## InfluxDB

| Desc | Value |
|---|---|
| URL | http://10.10.51.42:8086 |
| Username | admin |
| Password | password |

## InfluxDB Setup

| Desc | Value |
|---|---|
| Organization | TechGuys |
| Bucket | TG |
| Retention Period | 48h0m0s |


# How to

## Install InfluxDB

### Install InfluxDB

```
{
# influxdata-archive_compat.key GPG fingerprint:
#     9D53 9D90 D332 8DC7 D6C8 D3B9 D8FF 8E1F 7DF8 B07E
wget -q https://repos.influxdata.com/influxdata-archive_compat.key
echo '393e8779c89ac8d958f81f942f9ad7fb82a25e133faddaf92e15b16e6ac9ce4c influxdata-archive_compat.key' | sha256sum -c && cat influxdata-archive_compat.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg > /dev/null
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list

sudo apt-get update && sudo apt-get -y install influxdb2
}
```

### Start InfluxDB & Check Status

```
{
sudo systemctl start influxdb
sudo systemctl enable influxdb
slep 3
sudo systemctl status influxdb
}
```

### Verify InfluxDB port 8086

```
{
ss -tunelp | grep 8086
}
```

### InfluxDB Setup

setup
```
{
influx setup
}
```

Results
```
root@k-influxdb-01:~# influx setup
> Welcome to InfluxDB 2.0!
? Please type your primary username admin
? Please type your password ********
? Please type your password again ********
? Please type your primary organization name TechGuys
? Please type your primary bucket name TG
? Please type your retention period in hours, or 0 for infinite 48
? Setup with these parameters?
  Username:          admin
  Organization:      TechGuys
  Bucket:            TG
  Retention Period:  48h0m0s
 Yes
User	Organization	Bucket
admin	TechGuys	TG
root@k-influxdb-01:~# 
```

## Install Telegraf

### Install Telegraf

```
{
# influxdata-archive_compat.key GPG fingerprint:
#     9D53 9D90 D332 8DC7 D6C8 D3B9 D8FF 8E1F 7DF8 B07E
wget -q https://repos.influxdata.com/influxdata-archive_compat.key
echo '393e8779c89ac8d958f81f942f9ad7fb82a25e133faddaf92e15b16e6ac9ce4c influxdata-archive_compat.key' | sha256sum -c && cat influxdata-archive_compat.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg > /dev/null
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list

sudo apt-get update && sudo apt-get install telegraf
}
```
 

### Apply Auto-Configuration from InfluxDB 10.10.51.42

```
{
export INFLUX_TOKEN=qF2WEr9pzW3HrTSSv4uT8sbk0Ibb5IVIj8alT22mjqRSNRmLrqwi1r_zOOSGZooItw0nf2RQuNJj2PCkesbGkA==
telegraf --config http://10.10.51.42:8086/api/v2/telegrafs/0cc939f148893000
}
```
+++
title = 'Bind9 DNS'
+++
- [Ref](#ref)
- [Step](#step)
  - [Install bind9](#install-bind9)
    - [Install bind9](#install-bind9-1)
    - [Create Zone : server.maas](#create-zone--servermaas)
      - [Add Slave DNS to maas](#add-slave-dns-to-maas)
      - [Allow ingress.maas apparmor](#allow-ingressmaas-apparmor)
    - [Create Zone : ingress.k8s.maas](#create-zone--ingressk8smaas)
      - [Generate TSIG Key for Kubernetes External-DNS](#generate-tsig-key-for-kubernetes-external-dns)
      - [Create Zone gong.k8s.maas](#create-zone-gongk8smaas)
      - [Allow gong.k8s.maas apparmor](#allow-gongk8smaas-apparmor)
    - [Create Zone : monpf.k8s.maas](#create-zone--monpfk8smaas)
      - [Generate TSIG Key for Kubernetes External-DNS](#generate-tsig-key-for-kubernetes-external-dns-1)
      - [Create Zone monpf.k8s.maas](#create-zone-monpfk8smaas)
      - [Allow monpf.k8s.maas apparmor](#allow-monpfk8smaas-apparmor)
    - [Create Zone : monpf-workload.ingress.maas](#create-zone--monpf-workloadingressmaas)
      - [Generate TSIG Key for Kubernetes External-DNS](#generate-tsig-key-for-kubernetes-external-dns-2)
      - [Create Zone monpf-workload.ingress.maas](#create-zone-monpf-workloadingressmaas)
      - [Allow monpf-workload.ingress.maas apparmor](#allow-monpf-workloadingressmaas-apparmor)
      - [Set Forwarder and grant query to any](#set-forwarder-and-grant-query-to-any)
    - [Create Zone : play.k8s.maas](#create-zone--playk8smaas)
      - [Create Zone play.k8s.maas](#create-zone-playk8smaas)
      - [Allow monpf.k8s.maas apparmor](#allow-monpfk8smaas-apparmor-1)
    - [Create Zone : pso.k8s.maas](#create-zone--psok8smaas)
      - [Create Zone pso.k8s.maas](#create-zone-psok8smaas)
      - [Allow pso.k8s.maas apparmor](#allow-psok8smaas-apparmor)
    - [Create Zone : drapp.maas](#create-zone--drappmaas)
      - [Create Zone drapp.maas](#create-zone-drappmaas)
    - [DNS Lookup Test](#dns-lookup-test)



# Ref

[bind9 - example 1](https://www.richinfante.com/2020/02/21/bind9-on-my-lan)

# Step

## Install bind9

### Install bind9

```shell
{
sudo apt install bind9
}
```

### Create Zone : server.maas

#### Add Slave DNS to maas

```shell
{
ZONE_NAME=server.maas
MAAS_DNS_SERVER=10.10.0.6
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
mkdir -p ${DB_OUTPUT_DIR}
mkdir -p ${CONFIG_OUTPUT_DIR}
cat <<EOF > ${CONFIG_OUTPUT_DIR}/named.conf
zone "$ZONE_NAME" {
    type slave;
    file "${ZONE_DB_FILE}";
    masters { ${MAAS_DNS_SERVER}; };
    allow-query { any; };
};
EOF
#chown bind:bind ${ZONE_DB_FILE}
echo "include \"${CONFIG_OUTPUT_DIR}/named.conf\";" | sudo tee -a /etc/bind/named.conf.local
sleep .5
systemctl restart bind9
}
```

#### Allow ingress.maas apparmor

```shell
{
ZONE_NAME=server.maas
NS1_IP=10.10.0.6
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
echo "======= Copy below to apparmor ==================="
echo "${DB_OUTPUT_DIR}/ r,
${DB_OUTPUT_DIR}/** rw,
${ZONE_DB_FILE}.jnl rw,"
echo "======= End, After edit app armor , reload using below command ==================="
echo "vi /etc/apparmor.d/usr.sbin.named"
echo "sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.named" 
}
```

### Create Zone : ingress.k8s.maas

#### Generate TSIG Key for Kubernetes External-DNS

command

```shell
{
tsig-keygen -a hmac-sha256 externaldns-key
}
```


output

```shell
root@k-k8s-deployment-01:/etc/bind/ingress.maas# tsig-keygen -a hmac-sha256 externaldns-key
key "externaldns-key" {
	algorithm hmac-sha256;
	secret "GH6RDefrrZngKZZs57eTVf/tGWiqLAUSoPOPaZVFFLk=";
};
root@k-k8s-deployment-01:/etc/bind/ingress.maas# 
```

#### Create Zone gong.k8s.maas

```shell
{
ZONE_NAME=gong.k8s.maas
NS1_IP=10.10.0.8
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
mkdir -p ${DB_OUTPUT_DIR}
mkdir -p ${CONFIG_OUTPUT_DIR}
cat <<EOF > ${CONFIG_OUTPUT_DIR}/named.conf
key "externaldns-key" {
	algorithm hmac-sha256;
	secret "GH6RDefrrZngKZZs57eTVf/tGWiqLAUSoPOPaZVFFLk=";
};
zone "$ZONE_NAME" IN {
    type master;
    file "${ZONE_DB_FILE}";
    allow-query { any; };
    allow-transfer {
        key "externaldns-key";
    };
    update-policy {
        grant externaldns-key zonesub ANY;
    };
};
EOF
cat <<EOF2 > ${ZONE_DB_FILE}
@       IN SOA  ${ZONE_NAME}. nobody.${ZONE_NAME}. (
                                2014030801      ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
@   30 IN NS ${ZONE_NAME}.
ns01                  IN      A       ${NS1_IP}
@ 30 IN A ${NS1_IP}
EOF2
chown -R bind:bind ${DB_OUTPUT_DIR}
named-checkzone ${ZONE_NAME} ${ZONE_DB_FILE}
echo "include \"${CONFIG_OUTPUT_DIR}/named.conf\";" | sudo tee -a /etc/bind/named.conf.local
sleep .5
systemctl restart bind9
}
```

#### Allow gong.k8s.maas apparmor

```shell
{
ZONE_NAME=gong.k8s.maas
NS1_IP=10.10.0.8
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
echo "======= Copy below to apparmor ==================="
echo "${DB_OUTPUT_DIR}/ r,
${DB_OUTPUT_DIR}/** rw,
${ZONE_DB_FILE}.jnl rw,"
echo "======= End, After edit app armor , reload using below command ==================="
echo "vi /etc/apparmor.d/usr.sbin.named"
echo "sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.named" 
}
```

### Create Zone : monpf.k8s.maas

#### Generate TSIG Key for Kubernetes External-DNS

command

```shell
{
tsig-keygen -a hmac-sha256 externaldns-key
}
```


output

```shell
root@k-k8s-deployment-01:/etc/bind/ingress.maas# tsig-keygen -a hmac-sha256 externaldns-key
key "externaldns-key" {
	algorithm hmac-sha256;
	secret "GH6RDefrrZngKZZs57eTVf/tGWiqLAUSoPOPaZVFFLk=";
};
root@k-k8s-deployment-01:/etc/bind/ingress.maas# 
```

#### Create Zone monpf.k8s.maas

```shell
{
ZONE_NAME=monpf.k8s.maas
NS1_IP=10.10.0.8
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
mkdir -p ${DB_OUTPUT_DIR}
mkdir -p ${CONFIG_OUTPUT_DIR}
cat <<EOF > ${CONFIG_OUTPUT_DIR}/named.conf
# key "externaldns-key" {
# 	algorithm hmac-sha256;
# 	secret "GH6RDefrrZngKZZs57eTVf/tGWiqLAUSoPOPaZVFFLk=";
# };
zone "$ZONE_NAME" IN {
    type master;
    file "${ZONE_DB_FILE}";
    allow-query { any; };
    allow-transfer {
        key "externaldns-key";
    };
    update-policy {
        grant externaldns-key zonesub ANY;
    };
};
EOF
cat <<EOF2 > ${ZONE_DB_FILE}
@       IN SOA  ${ZONE_NAME}. nobody.${ZONE_NAME}. (
                                2014030801      ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
@   30 IN NS ${ZONE_NAME}.
ns01                  IN      A       ${NS1_IP}
@ 30 IN A ${NS1_IP}
EOF2
chown -R bind:bind ${DB_OUTPUT_DIR}
named-checkzone ${ZONE_NAME} ${ZONE_DB_FILE}
echo "include \"${CONFIG_OUTPUT_DIR}/named.conf\";" | sudo tee -a /etc/bind/named.conf.local
sleep .5
systemctl restart bind9
}
```

#### Allow monpf.k8s.maas apparmor

```shell
{
ZONE_NAME=monpf.k8s.maas
NS1_IP=10.10.0.8
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
echo "======= Copy below to apparmor ==================="
echo "${DB_OUTPUT_DIR}/ r,
${DB_OUTPUT_DIR}/** rw,
${ZONE_DB_FILE}.jnl rw,"
echo "======= End, After edit app armor , reload using below command ==================="
echo "vi /etc/apparmor.d/usr.sbin.named"
echo "sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.named" 
}
```

### Create Zone : monpf-workload.ingress.maas

#### Generate TSIG Key for Kubernetes External-DNS

command

```shell
{
tsig-keygen -a hmac-sha256 externaldns-key
}
```


output

```shell
root@k-k8s-deployment-01:/etc/bind/ingress.maas# tsig-keygen -a hmac-sha256 externaldns-key
key "externaldns-key" {
	algorithm hmac-sha256;
	secret "GH6RDefrrZngKZZs57eTVf/tGWiqLAUSoPOPaZVFFLk=";
};
root@k-k8s-deployment-01:/etc/bind/ingress.maas# 
```

#### Create Zone monpf-workload.ingress.maas

```shell
{
ZONE_NAME=monpf-workload.ingress.maas
NS1_IP=10.10.0.8
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
mkdir -p ${DB_OUTPUT_DIR}
mkdir -p ${CONFIG_OUTPUT_DIR}
cat <<EOF > ${CONFIG_OUTPUT_DIR}/named.conf
# key "externaldns-key" {
# 	algorithm hmac-sha256;
# 	secret "GH6RDefrrZngKZZs57eTVf/tGWiqLAUSoPOPaZVFFLk=";
# };
zone "$ZONE_NAME" IN {
    type master;
    file "${ZONE_DB_FILE}";
    allow-query { any; };
    allow-transfer {
        key "externaldns-key";
    };
    update-policy {
        grant externaldns-key zonesub ANY;
    };
};
EOF
cat <<EOF2 > ${ZONE_DB_FILE}
@       IN SOA  ${ZONE_NAME}. nobody.${ZONE_NAME}. (
                                2014030801      ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
@   30 IN NS ${ZONE_NAME}.
ns01                  IN      A       ${NS1_IP}
@ 30 IN A ${NS1_IP}
EOF2
chown -R bind:bind ${DB_OUTPUT_DIR}
named-checkzone ${ZONE_NAME} ${ZONE_DB_FILE}
echo "include \"${CONFIG_OUTPUT_DIR}/named.conf\";" | sudo tee -a /etc/bind/named.conf.local
sleep .5
systemctl restart bind9
}
```

#### Allow monpf-workload.ingress.maas apparmor

```shell
{
ZONE_NAME=monpf-workload.ingress.maas
NS1_IP=10.10.0.8
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
echo "======= Copy below to apparmor ==================="
echo "${DB_OUTPUT_DIR}/ r,
${DB_OUTPUT_DIR}/** rw,
${ZONE_DB_FILE}.jnl rw,"
echo "======= End, After edit app armor , reload using below command ==================="
echo "vi /etc/apparmor.d/usr.sbin.named"
echo "sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.named" 
}
```

#### Set Forwarder and grant query to any

```shell
{
cat <<EOF3 > /etc/bind/named.conf.options
options {
	directory "/var/cache/bind";

	dnssec-validation auto;

	listen-on-v6 { any; };

    recursion yes;
    allow-query { any; };
	allow-query-cache { any; };
    forwarders {
      1.1.1.1; // Cloudflare
      8.8.8.8; // Google
    };
};
EOF3
systemctl restart bind9
}
```


external-dns --txt-owner-id gong --provider rfc2136 --rfc2136-host=10.10.54.4 --rfc2136-port=53 --rfc2136-zone=ingress.maas --rfc2136-tsig-secret=GH6RDefrrZngKZZs57eTVf/tGWiqLAUSoPOPaZVFFLk= --rfc2136-tsig-secret-alg=hmac-sha256 --rfc2136-tsig-keyname=externaldns-key --rfc2136-tsig-axfr --source ingress --once --domain-filter=ingress.maas --dry-run


### Create Zone : play.k8s.maas

#### Create Zone play.k8s.maas

```shell
{
ZONE_NAME=play.k8s.maas
NS1_IP=10.10.0.8
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
mkdir -p ${DB_OUTPUT_DIR}
mkdir -p ${CONFIG_OUTPUT_DIR}
cat <<EOF > ${CONFIG_OUTPUT_DIR}/named.conf
# key "externaldns-key" {
# 	algorithm hmac-sha256;
# 	secret "GH6RDefrrZngKZZs57eTVf/tGWiqLAUSoPOPaZVFFLk=";
# };
zone "$ZONE_NAME" IN {
    type master;
    file "${ZONE_DB_FILE}";
    allow-query { any; };
    allow-transfer {
        key "externaldns-key";
    };
    update-policy {
        grant externaldns-key zonesub ANY;
    };
};
EOF
cat <<EOF2 > ${ZONE_DB_FILE}
@       IN SOA  ${ZONE_NAME}. nobody.${ZONE_NAME}. (
                                2014030801      ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
@   30 IN NS ${ZONE_NAME}.
ns01                  IN      A       ${NS1_IP}
@ 30 IN A ${NS1_IP}
EOF2
chown -R bind:bind ${DB_OUTPUT_DIR}
named-checkzone ${ZONE_NAME} ${ZONE_DB_FILE}
echo "include \"${CONFIG_OUTPUT_DIR}/named.conf\";" | sudo tee -a /etc/bind/named.conf.local
sleep .5
systemctl restart bind9
}
```

#### Allow monpf.k8s.maas apparmor

```shell
{
ZONE_NAME=play.k8s.maas
NS1_IP=10.10.0.8
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
echo "======= Copy below to apparmor ==================="
echo "${DB_OUTPUT_DIR}/ r,
${DB_OUTPUT_DIR}/** rw,
${ZONE_DB_FILE}.jnl rw,"
echo "======= End, After edit app armor , reload using below command ==================="
echo "vi /etc/apparmor.d/usr.sbin.named"
echo "sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.named" 
}
```

### Create Zone : pso.k8s.maas

#### Create Zone pso.k8s.maas

```shell
{
ZONE_NAME=pso.k8s.maas
NS1_IP=10.10.0.8
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
mkdir -p ${DB_OUTPUT_DIR}
mkdir -p ${CONFIG_OUTPUT_DIR}
cat <<EOF > ${CONFIG_OUTPUT_DIR}/named.conf
# key "externaldns-key" {
# 	algorithm hmac-sha256;
# 	secret "GH6RDefrrZngKZZs57eTVf/tGWiqLAUSoPOPaZVFFLk=";
# };
zone "$ZONE_NAME" IN {
    type master;
    file "${ZONE_DB_FILE}";
    allow-query { any; };
    allow-transfer {
        key "externaldns-key";
    };
    update-policy {
        grant externaldns-key zonesub ANY;
    };
};
EOF
cat <<EOF2 > ${ZONE_DB_FILE}
@       IN SOA  ${ZONE_NAME}. nobody.${ZONE_NAME}. (
                                2014030801      ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
@   30 IN NS ${ZONE_NAME}.
ns01                  IN      A       ${NS1_IP}
@ 30 IN A ${NS1_IP}
EOF2
chown -R bind:bind ${DB_OUTPUT_DIR}
named-checkzone ${ZONE_NAME} ${ZONE_DB_FILE}
echo "include \"${CONFIG_OUTPUT_DIR}/named.conf\";" | sudo tee -a /etc/bind/named.conf.local
sleep .5
systemctl restart bind9
}
```

#### Allow pso.k8s.maas apparmor

```shell
{
ZONE_NAME=pso.k8s.maas
NS1_IP=10.10.0.8
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
echo "======= Copy below to apparmor ==================="
echo "${DB_OUTPUT_DIR}/ r,
${DB_OUTPUT_DIR}/** rw,
${ZONE_DB_FILE}.jnl rw,"
echo "======= End, After edit app armor , reload using below command ==================="
echo "vi /etc/apparmor.d/usr.sbin.named"
echo "sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.named" 
}
```

### Create Zone : drapp.maas

#### Create Zone drapp.maas

```shell
{
ZONE_NAME=drapp.maas
NS1_IP=10.10.0.8
CONFIG_OUTPUT_DIR=/etc/bind/${ZONE_NAME}
DB_OUTPUT_DIR=/var/lib/bind/db/${ZONE_NAME}
ZONE_DB_FILE=${DB_OUTPUT_DIR}/${ZONE_NAME}.db
mkdir -p ${DB_OUTPUT_DIR}
mkdir -p ${CONFIG_OUTPUT_DIR}
cat <<EOF > ${CONFIG_OUTPUT_DIR}/named.conf
# key "externaldns-key" {
# 	algorithm hmac-sha256;
# 	secret "GH6RDefrrZngKZZs57eTVf/tGWiqLAUSoPOPaZVFFLk=";
# };
zone "$ZONE_NAME" IN {
    type master;
    file "${ZONE_DB_FILE}";
    allow-query { any; };
    allow-transfer {
        key "externaldns-key";
    };
    update-policy {
        grant externaldns-key zonesub ANY;
    };
};
EOF
cat <<EOF2 > ${ZONE_DB_FILE}
$TTL 30s    ; default TTL for zone
@       IN SOA  ${ZONE_NAME}. nobody.${ZONE_NAME}. (
                                2014030801      ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
@   30 IN NS ${ZONE_NAME}.
ns01                  IN      A       ${NS1_IP}
@ 30 IN A ${NS1_IP}
grafana IN A 10.10.52.80
EOF2
chown -R bind:bind ${DB_OUTPUT_DIR}
named-checkzone ${ZONE_NAME} ${ZONE_DB_FILE}
echo "include \"${CONFIG_OUTPUT_DIR}/named.conf\";" | sudo tee -a /etc/bind/named.conf.local
sleep .5
systemctl restart bind9
}
```

### DNS Lookup Test

command


```shell
{
BIND_DNS_SERVER=localhost
dig +noall +answer @${BIND_DNS_SERVER} ns01.ingress.maas
dig +noall +answer @${BIND_DNS_SERVER} server1.ingress.maas
dig +noall +answer @${BIND_DNS_SERVER} server2.ingress.maas
dig +noall +answer @${BIND_DNS_SERVER} server3.ingress.maas
}
```


output

```shell
root@k-k8s-deployment-01:/etc/bind# {
BIND_DNS_SERVER=localhost
dig +noall +answer @${BIND_DNS_SERVER} ns01.ingress.maas
dig +noall +answer @${BIND_DNS_SERVER} server1.ingress.maas
dig +noall +answer @${BIND_DNS_SERVER} server2.ingress.maas
dig +noall +answer @${BIND_DNS_SERVER} server3.ingress.maas
}
ns01.ingress.maas.	10800	IN	A	192.168.1.1
server1.ingress.maas.	10800	IN	A	172.30.1.101
server2.ingress.maas.	10800	IN	A	172.30.1.102
server3.ingress.maas.	10800	IN	A	172.30.1.103
root@k-k8s-deployment-01:/etc/bind# 

```
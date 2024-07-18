
- [Limitation \& References](#limitation--references)
- [Step](#step)
  - [Install RKE2](#install-rke2)
    - [Install RKE2 - Server](#install-rke2---server)
      - [Download Air-Gapped Image from RKE2 Release to /var/lib/rancher/rke2/agent/images](#download-air-gapped-image-from-rke2-release-to-varlibrancherrke2agentimages)
      - [Create RKE2 Server - Configuration File](#create-rke2-server---configuration-file)
      - [Add Insecure Registry (k-harbor-01.maas)](#add-insecure-registry-k-harbor-01maas)
      - [Fix CNI: Cilium Bu When Disable Kube Proxy Bug](#fix-cni-cilium-bu-when-disable-kube-proxy-bug)
      - [Install RKE2 Systemd \& Start Systemd](#install-rke2-systemd--start-systemd)
      - [journalctl - log tracing](#journalctl---log-tracing)
    - [Install RKE2 - Agent](#install-rke2---agent)
      - [Download Air-Gapped Image from RKE2 Release to /var/lib/rancher/rke2/agent/images](#download-air-gapped-image-from-rke2-release-to-varlibrancherrke2agentimages-1)
      - [Create Token on RKE2 Server](#create-token-on-rke2-server)
      - [Create RKE2 Agent Configuration](#create-rke2-agent-configuration)
      - [Add Insecure Registry (k-harbor-01.maas)](#add-insecure-registry-k-harbor-01maas-1)
      - [Install RKE2 Systemd \& Start Systemd](#install-rke2-systemd--start-systemd-1)
      - [journalctl - log tracing](#journalctl---log-tracing-1)




# Limitation & References

[bug : kubeproxy skip & CNI Cilium](https://github.com/rancher/rke2/issues/4862)

[RKE2 Releases](https://github.com/rancher/rke2/releases/tag/v1.30.2%2Brke2r1)

[RKE2 Air Gapped](https://docs.rke2.io/install/airgap)

# Step

## Install RKE2 

### Install RKE2 - Server

#### Download Air-Gapped Image from RKE2 Release to /var/lib/rancher/rke2/agent/images

download tarball from https://github.com/rancher/rke2/releases/tag/v1.30.2%2Brke2r1

```shell
{
rm -rf /var/lib/rancher/rke2/agent/images/
mkdir -p /var/lib/rancher/rke2/agent/images/
cd /var/lib/rancher/rke2/agent/images/
wget http://10.10.54.4/rke2/rke2-images-cilium.linux-amd64.tar.gz
wget http://10.10.54.4/rke2/rke2-images-cilium.linux-amd64.txt
wget http://10.10.54.4/rke2/rke2-images-core.linux-amd64.tar.gz
wget http://10.10.54.4/rke2/rke2-images-core.linux-amd64.txt
wget http://10.10.54.4/rke2/rke2-images.linux-amd64.tar.gz
wget http://10.10.54.4/rke2/rke2-images.linux-amd64.txt
wget http://10.10.54.4/rke2/sha256sum-amd64.txt
wget http://10.10.54.4/rke2/rke2.linux-amd64.tar.gz
}
```

#### Create RKE2 Server - Configuration File

```shell
{
# Define variables
CNI_NETWORK=cilium
KUBECONF_FILE="/etc/rancher/rke2/kubeconfig"
# Generate config.yaml content
mkdir -p /etc/rancher/rke2/
cat <<EOF > /etc/rancher/rke2/config.yaml
write-kubeconfig: ${KUBECONF_FILE}
write-kubeconfig-mode: "0644"
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
cni: ${CNI_NETWORK}
#cni: none
disable-kube-proxy: true
#debug: true
EOF
}
```

#### Add Insecure Registry (k-harbor-01.maas)

```shell
{
echo 'configs:
  "k-harbor-01.maas":
    tls:
      insecure_skip_verify: true' > /etc/rancher/rke2/registries.yaml
}
```


#### Fix CNI: Cilium Bu When Disable Kube Proxy Bug

[bug : kubeproxy skip & CNI Cilium](https://github.com/rancher/rke2/issues/4862)

```shell
{
mkdir -p /var/lib/rancher/rke2/server/manifests/
echo 'apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-cilium
  namespace: kube-system
spec:
  valuesContent: |-
    kubeProxyReplacement: true
    k8sServiceHost: 127.0.0.1
    k8sServicePort: 6443
    cni:
      chainingMode: "none"' > /var/lib/rancher/rke2/server/manifests/rke2-kubeproxyreplacement.yaml
}
```


#### Install RKE2 Systemd & Start Systemd

download https://get.rke2.io/ and rename to rke2_install.sh

```shell
{
curl -sfL http://10.10.54.4/rke2/rke2_install.sh --output install.sh
INSTALL_RKE2_ARTIFACT_PATH=/var/lib/rancher/rke2/agent/images/ sh install.sh
sleep 2
systemctl enable rke2-server.service
sleep 1
systemctl start rke2-server.service
}
```

#### journalctl - log tracing

should start all component by 5 to 10 minutes, waiting etcd & api server

```shell
{
journalctl -f -u rke2-server.service
}
```


### Install RKE2 - Agent

#### Download Air-Gapped Image from RKE2 Release to /var/lib/rancher/rke2/agent/images

download tarball from https://github.com/rancher/rke2/releases/tag/v1.30.2%2Brke2r1

```shell
{
rm -rf /var/lib/rancher/rke2/agent/images/
mkdir -p /var/lib/rancher/rke2/agent/images/
cd /var/lib/rancher/rke2/agent/images/
wget http://10.10.54.4/rke2/rke2-images-cilium.linux-amd64.tar.gz
wget http://10.10.54.4/rke2/rke2-images-cilium.linux-amd64.txt
wget http://10.10.54.4/rke2/rke2-images-core.linux-amd64.tar.gz
wget http://10.10.54.4/rke2/rke2-images-core.linux-amd64.txt
wget http://10.10.54.4/rke2/rke2-images.linux-amd64.tar.gz
wget http://10.10.54.4/rke2/rke2-images.linux-amd64.txt
wget http://10.10.54.4/rke2/sha256sum-amd64.txt
wget http://10.10.54.4/rke2/rke2.linux-amd64.tar.gz
}
```

#### Create Token on RKE2 Server

```shell
{
rke2 token create
}
```

exmaple result

```shell
root@k-gong-kubeadm2-m-01:/etc/rancher/rke2# rke2 token create
K109ca28e9c24021cf11eff61ef8aeb58e5a1ade9f960b6796d10329895d17f31a5::9k1dr3.7vo53as0pwu9s3xg
root@k-gong-kubeadm2-m-01:/etc/rancher/rke2# 
```

#### Create RKE2 Agent Configuration

```shell
{
# Define variables
RKE2_SERVER_IP=10.10.52.11
RKE2_AGENT_TOKEN=K109ca28e9c24021cf11eff61ef8aeb58e5a1ade9f960b6796d10329895d17f31a5::9k1dr3.7vo53as0pwu9s3xg
# Generate config.yaml content
mkdir -p /etc/rancher/rke2/
cat <<EOF > /etc/rancher/rke2/config.yaml
server: https://${RKE2_SERVER_IP}:9345
token: ${RKE2_AGENT_TOKEN}
EOF
}
```

#### Add Insecure Registry (k-harbor-01.maas)

```shell
{
echo 'configs:
  "k-harbor-01.maas":
    tls:
      insecure_skip_verify: true' > /etc/rancher/rke2/registries.yaml
}
```

#### Install RKE2 Systemd & Start Systemd

download https://get.rke2.io/ and rename to rke2_install.sh

```shell
{
curl -sfL http://10.10.54.4/rke2/rke2_install.sh --output install.sh
INSTALL_RKE2_ARTIFACT_PATH=/var/lib/rancher/rke2/agent/images/ sh install.sh
sleep 2
systemctl enable rke2-agent.service
sleep 1
systemctl start rke2-agent.service
}
```

#### journalctl - log tracing

should start all component by 5 to 10 minutes

```shell
{
journalctl -f -u rke2-agent.service
}
```





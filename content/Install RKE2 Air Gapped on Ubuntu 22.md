
- [RKE2 Best Installation Guide](#rke2-best-installation-guide)
- [Limitation \& References](#limitation--references)
- [Step](#step)
  - [Install RKE2](#install-rke2)
    - [Install RKE2 - Server (Master-Node)](#install-rke2---server-master-node)
      - [Enable Firewall on port 6443](#enable-firewall-on-port-6443)
      - [Enable RBD](#enable-rbd)
      - [Increase File Watcher](#increase-file-watcher)
      - [Download Air-Gapped Image from RKE2 Release to /var/lib/rancher/rke2/agent/images](#download-air-gapped-image-from-rke2-release-to-varlibrancherrke2agentimages)
        - [Download v1.30](#download-v130)
      - [Add Insecure Registry (k-harbor-01.server.maas)](#add-insecure-registry-k-harbor-01servermaas)
        - [Step to rewrite docker.io to k-harbor-01.server.maas](#step-to-rewrite-dockerio-to-k-harbor-01servermaas)
          - [Step to Add SelfSign prevent containerd x509 cert error](#step-to-add-selfsign-prevent-containerd-x509-cert-error)
          - [Config Work ( Redirect Docker.io to k-harbor-01.maas)](#config-work--redirect-dockerio-to-k-harbor-01maas)
          - [Config Work ( Rewrite Docker.io to proxy)](#config-work--rewrite-dockerio-to-proxy)
      - [Create RKE2 Token](#create-rke2-token)
      - [Configuration File](#configuration-file)
        - [Create RKE2 Server - Configuration File ( Main )](#create-rke2-server---configuration-file--main-)
        - [Create RKE2 Server - Configuration File (disable rke2-ingress-nginx)](#create-rke2-server---configuration-file-disable-rke2-ingress-nginx)
      - [Fix CNI: Cilium Bug, When Disable KubeProxy](#fix-cni-cilium-bug-when-disable-kubeproxy)
      - [(Optional) Add Helm Chart](#optional-add-helm-chart)
        - [(Optional) Metallb](#optional-metallb)
          - [Add MetalLB HelmChart](#add-metallb-helmchart)
          - [Add publishService to Nginx Ingress Controller, ingress will use metalLB externalIP](#add-publishservice-to-nginx-ingress-controller-ingress-will-use-metallb-externalip)
          - [Add External IP Pool \& L2 Advertise to MatalLB](#add-external-ip-pool--l2-advertise-to-matallb)
        - [(Optional) Rook-Operator](#optional-rook-operator)
          - [Prepare Extra Disk on Master or Worker Node, Ceph Needed Disk](#prepare-extra-disk-on-master-or-worker-node-ceph-needed-disk)
          - [must enable modprobe rbd , If not rbdplugin will failed to strat](#must-enable-modprobe-rbd--if-not-rbdplugin-will-failed-to-strat)
          - [Add Rook-Operator HelmChart](#add-rook-operator-helmchart)
          - [Patch Finalizer , if cannot delete](#patch-finalizer--if-cannot-delete)
          - [Patch Ceph Image](#patch-ceph-image)
          - [Create Ceph Cluster](#create-ceph-cluster)
          - [Monitor Cluster Provisioning Status](#monitor-cluster-provisioning-status)
          - [Optional : Ceph toolbox use to run ceph -s](#optional--ceph-toolbox-use-to-run-ceph--s)
        - [(Optional) External-DNS](#optional-external-dns)
          - [Add External DNS HelmChart](#add-external-dns-helmchart)
      - [Install RKE2 Systemd \& Start Systemd](#install-rke2-systemd--start-systemd)
      - [Copy Kubectl](#copy-kubectl)
      - [journalctl - log tracing](#journalctl---log-tracing)
      - [Replace RKE2 Kubeconfig IP \& Copy Kubeconfig to default kubeconfig](#replace-rke2-kubeconfig-ip--copy-kubeconfig-to-default-kubeconfig)
      - [Get Node](#get-node)
      - [Get Token](#get-token)
        - [Ouput](#ouput)
    - [Install RKE2 - Agent ( Worker-Node )](#install-rke2---agent--worker-node-)
      - [Optional Step](#optional-step)
        - [(Optional) Set RBD parameter, If Rook-Ceph enabled](#optional-set-rbd-parameter-if-rook-ceph-enabled)
      - [Increase File Watcher ( this command is temporary fix)](#increase-file-watcher--this-command-is-temporary-fix)
      - [Download Air-Gapped Image from RKE2 Release to /var/lib/rancher/rke2/agent/images](#download-air-gapped-image-from-rke2-release-to-varlibrancherrke2agentimages-1)
        - [Download v1.30](#download-v130-1)
      - [Add Insecure Registry (k-harbor-01.server.maas)](#add-insecure-registry-k-harbor-01servermaas-1)
        - [Step to rewrite docker.io to k-harbor-01.server.maas](#step-to-rewrite-dockerio-to-k-harbor-01servermaas-1)
          - [Step to Add SelfSign prevent containerd x509 cert error](#step-to-add-selfsign-prevent-containerd-x509-cert-error-1)
          - [Config Work ( Rewrite Docker.io to proxy)](#config-work--rewrite-dockerio-to-proxy-1)
      - [Create RKE2 Agent Configuration](#create-rke2-agent-configuration)
        - [RKE2 - Default Worker (Main)](#rke2---default-worker-main)
        - [RKE2 - Default Worker (disable nginx)](#rke2---default-worker-disable-nginx)
        - [RKE2 - Ceph Worker](#rke2---ceph-worker)
        - [RKE2 - Ceph Worker](#rke2---ceph-worker-1)
      - [Install RKE2 Systemd \& Start Systemd](#install-rke2-systemd--start-systemd-1)
      - [journalctl - log tracing](#journalctl---log-tracing-1)
  - [Uninstall RKE2](#uninstall-rke2)


# RKE2 Best Installation Guide

[https://dev.to/arman-shafiei/install-rke2-with-cilium-and-metallb-48a4](https://dev.to/arman-shafiei/install-rke2-with-cilium-and-metallb-48a4)

# Limitation & References

[bug : kubeproxy skip & CNI Cilium](https://github.com/rancher/rke2/issues/4862)

[RKE2 Releases](https://github.com/rancher/rke2/releases/tag/v1.30.2%2Brke2r1)

[RKE2 Air Gapped](https://docs.rke2.io/install/airgap)

# Step




## Install RKE2 

### Install RKE2 - Server (Master-Node)

#### Enable Firewall on port 6443

```shell
{
sudo ufw allow 2379/tcp
sudo ufw allow 2380/tcp
sudo ufw allow 2381/tcp
sudo ufw allow 2382/tcp
sudo ufw allow 9345/tcp
sudo ufw allow 6443/tcp
}
```

#### Enable RBD

{
modprobe rbd
echo rbd | sudo tee -a /etc/modules
}

#### Increase File Watcher

Permanantly

```shell
{
sudo bash -c 'echo -e "fs.inotify.max_user_watches=2099999999\nfs.inotify.max_user_instances=2099999999\nfs.inotify.max_queued_events=2099999999" > /etc/sysctl.d/99-inotify.conf && sysctl --system'
cat /etc/sysctl.d/99-inotify.conf
sysctl fs.inotify.max_user_watches
sysctl fs.inotify.max_user_instances
sysctl fs.inotify.max_queued_events
}
```

Onetime

```shell
{
sudo sysctl -w fs.inotify.max_user_watches=2099999999
sudo sysctl -w fs.inotify.max_user_instances=2099999999
sudo sysctl -w fs.inotify.max_queued_events=2099999999
}
```

#### Download Air-Gapped Image from RKE2 Release to /var/lib/rancher/rke2/agent/images

##### Download v1.30

download tarball from https://github.com/rancher/rke2/releases/tag/v1.30.2%2Brke2r1

```shell
{
rm -rf /root/images
mkdir -p /root/images
cd /root/images
wget http://10.10.54.4/rke2_1.30.3/rke2-images-cilium.linux-amd64.tar.zst
wget http://10.10.54.4/rke2_1.30.3/rke2-images.linux-amd64.tar.zst
wget http://10.10.54.4/rke2_1.30.3/sha256sum-amd64.txt
wget http://10.10.54.4/rke2_1.30.3/rke2-images-core.linux-amd64.tar.zst
wget http://10.10.54.4/rke2_1.30.3/rke2.linux-amd64.tar.gz
}
```


#### Add Insecure Registry (k-harbor-01.server.maas)

```shell
{
mkdir -p /etc/rancher/rke2/
echo 'configs:
  "k-harbor-01.server.maas":
    tls:
      insecure_skip_verify: true' > /etc/rancher/rke2/registries.yaml
}
```

##### Step to rewrite docker.io to k-harbor-01.server.maas

###### Step to Add SelfSign prevent containerd x509 cert error

```shell
{
export HABOR_URL=k-harbor-01.server.maas:443
openssl s_client -connect ${HABOR_URL} -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM > 00-harbor-ca.crt 
sudo cp 00-harbor-ca.crt /usr/local/share/ca-certificates/00-harbor-ca.crt 
sudo update-ca-certificates
curl -k https://${HABOR_URL}
}
```

###### Config Work ( Redirect Docker.io to k-harbor-01.maas)

```shell
{
mkdir -p /etc/rancher/rke2/
echo 'mirrors:
  docker.io:
    endpoint:
      - "https://k-harbor-01.server.maas:443"' > /etc/rancher/rke2/registries.yaml
cat /etc/rancher/rke2/registries.yaml
}
```

###### Config Work ( Rewrite Docker.io to proxy)

```shell
{
mkdir -p /etc/rancher/rke2/
PRIVATE_REPO_URL="k-harbor-01.server.maas:443"
cat <<EOF > /etc/rancher/rke2/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-hub.docker.io/\$1"
  hub.docker.io:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-hub.docker.io/\$1"
  ghcr.io:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-ghcr.io/\$1"
  gcr.io:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-gcr.io/\$1"
  quay.io:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-quay.io/\$1"
  registry.k8s.io:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-registry.k8s.io/\$1"
  public.ecr.aws:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-public.ecr.aws/\$1"
  registry.gitlab.com:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-registry.gitlab.com/\$1"
EOF
cat /etc/rancher/rke2/registries.yaml
}
```

<!-- ###### Restart RKE Server

```shell
{
#systemctl restart rke2-agent
systemctl restart rke2-server
}
``` -->

#### Create RKE2 Token

cmd

```shell
{
openssl rand -hex 32
}
```

output

```shell
{
root@k-gong-kubeadm2-m-01:~# openssl rand -hex 32
18ad75ac20304b27050dd905bd36eb5a688a1a0ac84a9cd71c4c64ab6e92c018
root@k-gong-kubeadm2-m-01:~# 
}
```

#### Configuration File

##### Create RKE2 Server - Configuration File ( Main )

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
#token: ${SERVER_TOKEN}
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
  - "node-role.kubernetes.io/control-plane:NoSchedule"
# disable:
#   - rke2-ingress-nginx
EOF
}
```

##### Create RKE2 Server - Configuration File (disable rke2-ingress-nginx) 

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
#token: ${SERVER_TOKEN}
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
  - "node-role.kubernetes.io/control-plane:NoSchedule"
disable:
  - rke2-ingress-nginx
EOF
}
```

#### Fix CNI: Cilium Bug, When Disable KubeProxy

[bug : kubeproxy skip & CNI Cilium](https://github.com/rancher/rke2/issues/4862)

```shell
{
mkdir -p /var/lib/rancher/rke2/server/manifests/
# Define variables for k8sServiceHost and k8sServicePort
K8S_SERVICE_HOST="10.10.52.11"
K8S_SERVICE_PORT="6443"
# Use a here document (EOF) to create the YAML content
cat << EOF > /var/lib/rancher/rke2/server/manifests/rke2-kubeproxyreplacement.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-cilium
  namespace: kube-system
spec:
  valuesContent: |-
    kubeProxyReplacement: true
    k8sServiceHost: $K8S_SERVICE_HOST
    k8sServicePort: $K8S_SERVICE_PORT
EOF
}
```
#### (Optional) Add Helm Chart

##### (Optional) Metallb

###### Add MetalLB HelmChart

```shell
{
CHART_FILENAME=rke2-metallb.yaml
echo "
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: metallb
  namespace: kube-system
spec:
  insecureSkipTLSVerify: true
  chart: oci://k-harbor-01.server.maas/helm-chart/metallb
  version: 0.14.7
  targetNamespace: metallb-system
  createNamespace: true
  # authSecret:
  #   name: example-repo-auth
  repoCAConfigMap:
    name: k-harbor-01-maas-ca
  valuesContent: |-
    controller:
      image:
        repository: k-harbor-01.server.maas/metallb_image/controller
        tag: v0.14.7
        pullPolicy:

    speaker:
      image:
        repository: k-harbor-01.server.maas/metallb_image/speaker
        tag: v0.14.7
        pullPolicy:

      frr:
        image:
          repository: k-harbor-01.server.maas/metallb_image/frr
          tag: 9.1.0
          pullPolicy:
# ---
# apiVersion: v1
# kind: Secret
# metadata:
#   namespace: kube-system
#   name: example-repo-auth
# type: kubernetes.io/basic-auth
# stringData:
#   username: user
#   password: pass
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: kube-system
  name: k-harbor-01-maas-ca
data:
  ca.crt: |-
    -----BEGIN CERTIFICATE-----
    MIIDwDCCAqigAwIBAgIUZOdpWXJUvAQ8yLosFgrzOKPSBi8wDQYJKoZIhvcNAQEL
    BQAwbTELMAkGA1UEBhMCWFgxDDAKBgNVBAgMA04vQTEMMAoGA1UEBwwDTi9BMSAw
    HgYDVQQKDBdTZWxmLXNpZ25lZCBjZXJ0aWZpY2F0ZTEgMB4GA1UEAwwXay1oYXJi
    b3ItMDEuc2VydmVyLm1hYXMwHhcNMjQwODEwMTc1MjMxWhcNMzQwODA4MTc1MjMx
    WjBtMQswCQYDVQQGEwJYWDEMMAoGA1UECAwDTi9BMQwwCgYDVQQHDANOL0ExIDAe
    BgNVBAoMF1NlbGYtc2lnbmVkIGNlcnRpZmljYXRlMSAwHgYDVQQDDBdrLWhhcmJv
    ci0wMS5zZXJ2ZXIubWFhczCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
    AM/PP7jhDMNKlFhh/L4MIsdjRSzHZIUsT1h0QtqMRvupLRU2AA+KKDCWqeh7SjxT
    En4qiyh+NCj4sBX5Gy9ZOmjZ6MdFYlAwUkTJHiCbjVugmaQpzIjiBxNYRv0ogGSq
    /cy3Id4eKH9ZDYJq74lKJr8Ogz5F/uP/I+KDe1rUyffI/YuWF3wL3YDyT3A4Sa7D
    J+ywmA7+ZMqzjK2l3i1OObPlbgoyEojjUBel8cpxDie/ET0QfwlWy8KE5D7N5pNd
    irjXw68abEyckvM+Cb4XXGLXLC5zp0g5C3EvkK7W5X+zOYHNqI+oNUum4KtBumxD
    1XTkF+kRjj/8AuJ0YUGO520CAwEAAaNYMFYwNQYDVR0RBC4wLIIXay1oYXJib3It
    MDEuc2VydmVyLm1hYXOCC2staGFyYm9yLTAxhwQKCjbxMB0GA1UdDgQWBBSZ05BD
    mv0fQGmryWcYf0olCPhi7TANBgkqhkiG9w0BAQsFAAOCAQEAlCE6E6R5Hi586+TN
    /hrGRtTUemvY9BQjES0pJm4i14SOMRWglqCRvE5FAbuBXVyBb4YguFCh8w+V/Cjw
    oItAcGmRfTNHDti3aMLCSB7tcmws3A6MRUY0qIQ4TNyydCePOWEmajcpBbuVXrW0
    NBsibWd/7j+CUum3iExbgBdVf26P5B8oQ+k2Ed6cTVhxOR+UMvtbOazBE8zCEH/C
    sAVJbCn7Y9hR+/CkUOKgTD83G3r/c0dbG0YcN51eWWvztPXfjjGOkmmYGj2O7l46
    bHYo9kliH31IHj1QBml5fNlQN5A+X5pki/vTLep5mpAUdqB4x3XRpyo3+5UnKyN+
    9M87nA==
    -----END CERTIFICATE-----
  " > /var/lib/rancher/rke2/server/manifests/${CHART_FILENAME}
}
```

###### Add publishService to Nginx Ingress Controller, ingress will use metalLB externalIP

[References : RKE2 + MetalLB + PFsense](https://blog.monotok.org/posts/rke2-k8s-nginx-ingress-metallb-pfsense/)

```shell
{
CHART_FILENAME=rke2-nginx-ingress-config.yaml
echo "
---
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-ingress-nginx
  namespace: kube-system
spec:
  valuesContent: |-
    controller:
      publishService:
        enabled: true
      service:
        enabled: true
        type: LoadBalancer
" > /var/lib/rancher/rke2/server/manifests/${CHART_FILENAME}
}
```

###### Add External IP Pool & L2 Advertise to MatalLB

```shell
{
EXTERNAL_IP_RANGE=10.10.52.80-10.10.52.99
cat <<EOF | kubectl create -f -
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ippool-pxe
  namespace: metallb-system
spec:
  addresses:
  - ${EXTERNAL_IP_RANGE}
  avoidBuggyIPs: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-pool-pxe
  namespace: metallb-system
spec:
  ipAddressPools:
  - ippool-pxe
  # nodeSelectors:
  # - matchLabels:
  #     kubernetes.io/hostname: k-k8s-metallb-01
  # interfaces:
  # - ens224
EOF
}
```





##### (Optional) Rook-Operator 

###### Prepare Extra Disk on Master or Worker Node, Ceph Needed Disk

Add Extra disk to master node & worker node

######  must enable modprobe rbd , If not rbdplugin will failed to strat

https://github.com/rook/rook/issues/13490
<br>
fix

``` shell
{
modprobe rbd
echo rbd | sudo tee -a /etc/modules
}
```

###### Add Rook-Operator HelmChart

```shell
{
CHART_FILENAME=rke2-rook-operator.yaml
echo "
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: rook-ceph
  namespace: kube-system
spec:
  chart: oci://k-harbor-01.server.maas/helm-chart/rook-ceph
  version: v1.14.8
  targetNamespace: rook-ceph
  createNamespace: true
  # authSecret:
  #   name: example-repo-auth
  repoCAConfigMap:
    name: k-harbor-01-maas-ca
  valuesContent: |-
    image:
      # -- Image
      repository: k-harbor-01.server.maas/rook_image/ceph
# ---
# apiVersion: v1
# kind: Secret
# metadata:
#   namespace: kube-system
#   name: example-repo-auth
# type: kubernetes.io/basic-auth
# stringData:
#   username: user
#   password: pass
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: kube-system
  name: k-harbor-01-maas-ca
data:
  ca.crt: |-
    -----BEGIN CERTIFICATE-----
    MIIDwDCCAqigAwIBAgIUZOdpWXJUvAQ8yLosFgrzOKPSBi8wDQYJKoZIhvcNAQEL
    BQAwbTELMAkGA1UEBhMCWFgxDDAKBgNVBAgMA04vQTEMMAoGA1UEBwwDTi9BMSAw
    HgYDVQQKDBdTZWxmLXNpZ25lZCBjZXJ0aWZpY2F0ZTEgMB4GA1UEAwwXay1oYXJi
    b3ItMDEuc2VydmVyLm1hYXMwHhcNMjQwODEwMTc1MjMxWhcNMzQwODA4MTc1MjMx
    WjBtMQswCQYDVQQGEwJYWDEMMAoGA1UECAwDTi9BMQwwCgYDVQQHDANOL0ExIDAe
    BgNVBAoMF1NlbGYtc2lnbmVkIGNlcnRpZmljYXRlMSAwHgYDVQQDDBdrLWhhcmJv
    ci0wMS5zZXJ2ZXIubWFhczCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
    AM/PP7jhDMNKlFhh/L4MIsdjRSzHZIUsT1h0QtqMRvupLRU2AA+KKDCWqeh7SjxT
    En4qiyh+NCj4sBX5Gy9ZOmjZ6MdFYlAwUkTJHiCbjVugmaQpzIjiBxNYRv0ogGSq
    /cy3Id4eKH9ZDYJq74lKJr8Ogz5F/uP/I+KDe1rUyffI/YuWF3wL3YDyT3A4Sa7D
    J+ywmA7+ZMqzjK2l3i1OObPlbgoyEojjUBel8cpxDie/ET0QfwlWy8KE5D7N5pNd
    irjXw68abEyckvM+Cb4XXGLXLC5zp0g5C3EvkK7W5X+zOYHNqI+oNUum4KtBumxD
    1XTkF+kRjj/8AuJ0YUGO520CAwEAAaNYMFYwNQYDVR0RBC4wLIIXay1oYXJib3It
    MDEuc2VydmVyLm1hYXOCC2staGFyYm9yLTAxhwQKCjbxMB0GA1UdDgQWBBSZ05BD
    mv0fQGmryWcYf0olCPhi7TANBgkqhkiG9w0BAQsFAAOCAQEAlCE6E6R5Hi586+TN
    /hrGRtTUemvY9BQjES0pJm4i14SOMRWglqCRvE5FAbuBXVyBb4YguFCh8w+V/Cjw
    oItAcGmRfTNHDti3aMLCSB7tcmws3A6MRUY0qIQ4TNyydCePOWEmajcpBbuVXrW0
    NBsibWd/7j+CUum3iExbgBdVf26P5B8oQ+k2Ed6cTVhxOR+UMvtbOazBE8zCEH/C
    sAVJbCn7Y9hR+/CkUOKgTD83G3r/c0dbG0YcN51eWWvztPXfjjGOkmmYGj2O7l46
    bHYo9kliH31IHj1QBml5fNlQN5A+X5pki/vTLep5mpAUdqB4x3XRpyo3+5UnKyN+
    9M87nA==
    -----END CERTIFICATE-----
  " > /var/lib/rancher/rke2/server/manifests/${CHART_FILENAME}
}
```

###### Patch Finalizer , if cannot delete

```shell
{
kubectl -n rook-ceph patch cephblockpool replicapool --type=merge -p '{"metadata":{"finalizers":[]}}'
}
```

###### Patch Ceph Image

```shell
{
kubectl get configmaps rook-ceph-operator-config -o yaml | grep "IMAGE:"
echo "================================================="
cat << EOF > ./patch_ceph_image.yaml
data:
  ROOK_CSI_ATTACHER_IMAGE: k-harbor-01.server.maas/k8s-csi/csi-attacher:v4.5.1
  ROOK_CSI_PROVISIONER_IMAGE: k-harbor-01.server.maas/k8s-csi/csi-provisioner:v4.0.1
  ROOK_CSI_REGISTRAR_IMAGE: k-harbor-01.server.maas/k8s-csi/csi-node-driver-registrar:v2.10.1
  ROOK_CSI_RESIZER_IMAGE: k-harbor-01.server.maas/k8s-csi/csi-resizer:v1.10.1
  ROOK_CSI_SNAPSHOTTER_IMAGE: k-harbor-01.server.maas/k8s-csi/csi-snapshotter:v7.0.2
  ROOK_CSI_CEPH_IMAGE: k-harbor-01.server.maas/ceph_image/cephcsi:v3.11.0
  ROOK_CSIADDONS_IMAGE: k-harbor-01.server.maas/ceph_image/k8s-sidecar:v0.8.0
EOF
kubectl patch configmap rook-ceph-operator-config --patch "$(cat ./patch_ceph_image.yaml)"
sleep .5
kubectl get configmaps rook-ceph-operator-config -o yaml | grep "IMAGE:"
}
```


###### Create Ceph Cluster


```shell
{
cat << EOF > ./create_ceph_cluster.yaml
#################################################################################################################
# Define the settings for the rook-ceph cluster with common settings for a small test cluster.
# All nodes with available raw devices will be used for the Ceph cluster. One node is sufficient
# in this example.

# For example, to create the cluster:
#   kubectl create -f crds.yaml -f common.yaml -f operator.yaml
#   kubectl create -f cluster-test.yaml
#################################################################################################################
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: my-ceph
  namespace: rook-ceph # namespace:cluster
spec:
  dataDirHostPath: /var/lib/rook
  cephVersion:
    image: quay.io/ceph/ceph:v18.2.2
    allowUnsupported: true
  mon:
    count: 1
    allowMultiplePerNode: true
  mgr:
    count: 1
    allowMultiplePerNode: true
    modules:
      - name: rook
        enabled: true
  dashboard:
    enabled: true
  crashCollector:
    disable: true
  storage:
    useAllNodes: true
    useAllDevices: true
    #deviceFilter:
    #config:
    #  deviceClass: testclass
  monitoring:
    enabled: false
  healthCheck:
    daemonHealth:
      mon:
        interval: 45s
        timeout: 600s
  priorityClassNames:
    all: system-node-critical
    mgr: system-cluster-critical
  disruptionManagement:
    managePodBudgets: true
  cephConfig:
    global:
      osd_pool_default_size: "1"
      mon_warn_on_pool_no_redundancy: "false"
      bdev_flock_retry: "20"
      bluefs_buffered_io: "false"
      mon_data_avail_warn: "10"

---
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 1
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
   annotations:
     storageclass.kubernetes.io/is-default-class: "true"
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
    # clusterID is the namespace where the rook cluster is running
    clusterID: rook-ceph
    # Ceph pool into which the RBD image shall be created
    pool: replicapool

    # (optional) mapOptions is a comma-separated list of map options.
    # For krbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd/#kernel-rbd-krbd-options
    # For nbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd-nbd/#options
    # mapOptions: lock_on_read,queue_depth=1024

    # (optional) unmapOptions is a comma-separated list of unmap options.
    # For krbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd/#kernel-rbd-krbd-options
    # For nbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd-nbd/#options
    # unmapOptions: force

    # RBD image format. Defaults to "2".
    imageFormat: "2"
    imageFeatures: layering

    # The secrets contain Ceph admin credentials.
    csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
    csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
    csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
    csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

    csi.storage.k8s.io/fstype: ext4

# Delete the rbd volume when a PVC is deleted
reclaimPolicy: Delete

# Optional, if you want to add dynamic resize for PVC.
# For now only ext3, ext4, xfs resize support provided, like in Kubernetes itself.
allowVolumeExpansion: true
EOF
sleep .5
kubectl apply -f create_ceph_cluster.yaml
}
```

###### Monitor Cluster Provisioning Status

```shell
{
nattawat.ujj@k-k8s-deployment-01:~$ k get cephclusters.ceph.rook.io  -w
NAME      DATADIRHOSTPATH   MONCOUNT   AGE   PHASE         MESSAGE                 HEALTH   EXTERNAL   FSID
my-ceph   /var/lib/rook     1          91s   Progressing   Configuring Ceph Mons                       
my-ceph   /var/lib/rook     1          2m28s   Progressing   Configuring Ceph Mgr(s)                       
my-ceph   /var/lib/rook     1          2m54s   Progressing   Configuring Ceph OSDs                         
my-ceph   /var/lib/rook     1          3m21s   Progressing   Processing OSD 0 on node "monpf-workload-worker-02"                       
my-ceph   /var/lib/rook     1          6m22s   Progressing   Processing OSD 1 on node "monpf-workload-worker-01"                       
my-ceph   /var/lib/rook     1          6m26s   Progressing   Processing OSD 1 on node "monpf-workload-worker-01"                       
my-ceph   /var/lib/rook     1          6m27s   Ready         Cluster created successfully                          HEALTH_OK              ad9b2dfc-d273-404b-ae34-018768fa8d71
my-ceph   /var/lib/rook     1          7m31s   Ready         Cluster created successfully                          HEALTH_OK              ad9b2dfc-d273-404b-ae34-018768fa8d71
}
```

###### Optional : Ceph toolbox use to run ceph -s

[Ceph Toolbox - yaml](https://github.com/rook/rook/blob/master/deploy/examples/toolbox.yaml)

```shell
{
cat << EOF > ./create_ceph_toolbox.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rook-ceph-tools
  namespace: rook-ceph # namespace:cluster
  labels:
    app: rook-ceph-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rook-ceph-tools
  template:
    metadata:
      labels:
        app: rook-ceph-tools
    spec:
      dnsPolicy: ClusterFirstWithHostNet
      serviceAccountName: rook-ceph-default
      containers:
        - name: rook-ceph-tools
          image: k-harbor-01.server.maas/ceph_image/ceph:v18.2.2
          command:
            - /bin/bash
            - -c
            - |
              # Replicate the script from toolbox.sh inline so the ceph image
              # can be run directly, instead of requiring the rook toolbox
              CEPH_CONFIG="/etc/ceph/ceph.conf"
              MON_CONFIG="/etc/rook/mon-endpoints"
              KEYRING_FILE="/etc/ceph/keyring"

              # create a ceph config file in its default location so ceph/rados tools can be used
              # without specifying any arguments
              write_endpoints() {
                endpoints=$(cat ${MON_CONFIG})

                # filter out the mon names
                # external cluster can have numbers or hyphens in mon names, handling them in regex
                # shellcheck disable=SC2001
                mon_endpoints=$(echo "${endpoints}"| sed 's/[a-z0-9_-]\+=//g')

                DATE=$(date)
                echo "$DATE writing mon endpoints to ${CEPH_CONFIG}: ${endpoints}"
                  cat <<EOF > ${CEPH_CONFIG}
              [global]
              mon_host = ${mon_endpoints}

              [client.admin]
              keyring = ${KEYRING_FILE}
              EOF
              }

              # watch the endpoints config file and update if the mon endpoints ever change
              watch_endpoints() {
                # get the timestamp for the target of the soft link
                real_path=$(realpath ${MON_CONFIG})
                initial_time=$(stat -c %Z "${real_path}")
                while true; do
                  real_path=$(realpath ${MON_CONFIG})
                  latest_time=$(stat -c %Z "${real_path}")

                  if [[ "${latest_time}" != "${initial_time}" ]]; then
                    write_endpoints
                    initial_time=${latest_time}
                  fi

                  sleep 10
                done
              }

              # read the secret from an env var (for backward compatibility), or from the secret file
              ceph_secret=${ROOK_CEPH_SECRET}
              if [[ "$ceph_secret" == "" ]]; then
                ceph_secret=$(cat /var/lib/rook-ceph-mon/secret.keyring)
              fi

              # create the keyring file
              cat <<EOF > ${KEYRING_FILE}
              [${ROOK_CEPH_USERNAME}]
              key = ${ceph_secret}
              EOF

              # write the initial config file
              write_endpoints

              # continuously update the mon endpoints if they fail over
              watch_endpoints
          imagePullPolicy: IfNotPresent
          tty: true
          securityContext:
            runAsNonRoot: true
            runAsUser: 2016
            runAsGroup: 2016
            capabilities:
              drop: ["ALL"]
          env:
            - name: ROOK_CEPH_USERNAME
              valueFrom:
                secretKeyRef:
                  name: rook-ceph-mon
                  key: ceph-username
          volumeMounts:
            - mountPath: /etc/ceph
              name: ceph-config
            - name: mon-endpoint-volume
              mountPath: /etc/rook
            - name: ceph-admin-secret
              mountPath: /var/lib/rook-ceph-mon
              readOnly: true
      volumes:
        - name: ceph-admin-secret
          secret:
            secretName: rook-ceph-mon
            optional: false
            items:
              - key: ceph-secret
                path: secret.keyring
        - name: mon-endpoint-volume
          configMap:
            name: rook-ceph-mon-endpoints
            items:
              - key: data
                path: mon-endpoints
        - name: ceph-config
          emptyDir: {}
      tolerations:
        - key: "node.kubernetes.io/unreachable"
          operator: "Exists"
          effect: "NoExecute"
          tolerationSeconds: 5
EOF
sleep .5
kubectl apply -f ./create_ceph_toolbox.yaml
}
```

##### (Optional) External-DNS

###### Add External DNS HelmChart

you need to adjust the parameter to suit your dns

```shell
{
CHART_FILENAME=rke2-external-dns.yaml
ZONE_NAME=gong.k8s.maas
DNS_SERVER_IP=10.10.0.8
DNS_KEYNAME="externaldns-key"
DNS_KEYPASSWORD="GH6RDefrrZngKZZs57eTVf/tGWiqLAUSoPOPaZVFFLk="
echo "
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: external-dns
  namespace: kube-system
spec:
  chart: oci://k-harbor-01.server.maas/helm-chart/external-dns
  version: 1.14.5
  targetNamespace: external-dns
  createNamespace: true
  # authSecret:
  #   name: example-repo-auth
  repoCAConfigMap:
    name: k-harbor-01-maas-ca
  valuesContent: |-
    # The external-dns image to use.
    image:
      repository: k-harbor-01.server.maas/externaldns_image/external-dns
      tag: v0.14.2
    # The DNS provider to use. This example uses RFC2136.
    provider: rfc2136
    # Configuration specific to the RFC2136 provider.
    rfc2136:
      # The URL of the DNS server to use.
      server: "${DNS_SERVER_IP}"
      # The port of the DNS server to use.
      port: 53
      # The name of the zone to manage.
      zone: "${ZONE_NAME}"
    # List of sources from which ExternalDNS should read the records.
    sources:
      - service
      - ingress
    # The interval at which to synchronize the records.
    interval: "1m"
    # The domain filters to use. This restricts external-dns to only manage records
    # within these domains.
    domainFilters:
      - ${ZONE_NAME}
    # The policy for managing DNS records. Possible values are "upsert-only" and "sync".
    policy: "sync"
    # If true, the external-dns controller will create the records it manages even if they do not exist.
    createRecord: true
    # The log level for ExternalDNS. Possible values are "debug", "info", "warn", "error".
    logLevel: "info"
    # The service account to use for ExternalDNS.
    serviceAccount:
      create: true
      name: ""
    # Resources requests and limits for the ExternalDNS pod.
    resources: {}
    extraArgs:
      - --rfc2136-host=${DNS_SERVER_IP}
      - --rfc2136-port=53
      - --rfc2136-zone=${ZONE_NAME}
      - --rfc2136-tsig-keyname=${DNS_KEYNAME}
      - --rfc2136-tsig-secret=${DNS_KEYPASSWORD}
      - --rfc2136-tsig-secret-alg=hmac-sha256
      - --rfc2136-min-ttl=90s
      - --rfc2136-tsig-axfr
# ---
# apiVersion: v1
# kind: Secret
# metadata:
#   namespace: kube-system
#   name: example-repo-auth
# type: kubernetes.io/basic-auth
# stringData:
#   username: user
#   password: pass
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: kube-system
  name: k-harbor-01-maas-ca
data:
  ca.crt: |-
    -----BEGIN CERTIFICATE-----
    MIIDwDCCAqigAwIBAgIUZOdpWXJUvAQ8yLosFgrzOKPSBi8wDQYJKoZIhvcNAQEL
    BQAwbTELMAkGA1UEBhMCWFgxDDAKBgNVBAgMA04vQTEMMAoGA1UEBwwDTi9BMSAw
    HgYDVQQKDBdTZWxmLXNpZ25lZCBjZXJ0aWZpY2F0ZTEgMB4GA1UEAwwXay1oYXJi
    b3ItMDEuc2VydmVyLm1hYXMwHhcNMjQwODEwMTc1MjMxWhcNMzQwODA4MTc1MjMx
    WjBtMQswCQYDVQQGEwJYWDEMMAoGA1UECAwDTi9BMQwwCgYDVQQHDANOL0ExIDAe
    BgNVBAoMF1NlbGYtc2lnbmVkIGNlcnRpZmljYXRlMSAwHgYDVQQDDBdrLWhhcmJv
    ci0wMS5zZXJ2ZXIubWFhczCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
    AM/PP7jhDMNKlFhh/L4MIsdjRSzHZIUsT1h0QtqMRvupLRU2AA+KKDCWqeh7SjxT
    En4qiyh+NCj4sBX5Gy9ZOmjZ6MdFYlAwUkTJHiCbjVugmaQpzIjiBxNYRv0ogGSq
    /cy3Id4eKH9ZDYJq74lKJr8Ogz5F/uP/I+KDe1rUyffI/YuWF3wL3YDyT3A4Sa7D
    J+ywmA7+ZMqzjK2l3i1OObPlbgoyEojjUBel8cpxDie/ET0QfwlWy8KE5D7N5pNd
    irjXw68abEyckvM+Cb4XXGLXLC5zp0g5C3EvkK7W5X+zOYHNqI+oNUum4KtBumxD
    1XTkF+kRjj/8AuJ0YUGO520CAwEAAaNYMFYwNQYDVR0RBC4wLIIXay1oYXJib3It
    MDEuc2VydmVyLm1hYXOCC2staGFyYm9yLTAxhwQKCjbxMB0GA1UdDgQWBBSZ05BD
    mv0fQGmryWcYf0olCPhi7TANBgkqhkiG9w0BAQsFAAOCAQEAlCE6E6R5Hi586+TN
    /hrGRtTUemvY9BQjES0pJm4i14SOMRWglqCRvE5FAbuBXVyBb4YguFCh8w+V/Cjw
    oItAcGmRfTNHDti3aMLCSB7tcmws3A6MRUY0qIQ4TNyydCePOWEmajcpBbuVXrW0
    NBsibWd/7j+CUum3iExbgBdVf26P5B8oQ+k2Ed6cTVhxOR+UMvtbOazBE8zCEH/C
    sAVJbCn7Y9hR+/CkUOKgTD83G3r/c0dbG0YcN51eWWvztPXfjjGOkmmYGj2O7l46
    bHYo9kliH31IHj1QBml5fNlQN5A+X5pki/vTLep5mpAUdqB4x3XRpyo3+5UnKyN+
    9M87nA==
    -----END CERTIFICATE-----
  " > /var/lib/rancher/rke2/server/manifests/${CHART_FILENAME}
}
```

#### Install RKE2 Systemd & Start Systemd

download https://get.rke2.io/ and rename to rke2_install.sh

```shell
{
export TARGET_RKE_VERSION='v1.30.3+rke2r1'
curl -sfL http://10.10.54.4/rke2_1.30.3/get.rke2.io --output install.sh
chmod 755 ./install.sh
INSTALL_RKE2_ARTIFACT_PATH=/root/images INSTALL_RKE2_VERSION="${TARGET_RKE_VERSION}" INSTALL_RKE2_TYPE="server" ./install.sh
sleep 2
systemctl enable rke2-server.service
sleep 1
systemctl start rke2-server.service
sleep 10
rm -f /var/lib/rancher/rke2/agent/images/*
}
```


#### Copy Kubectl

```shell
{
echo 'PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc
source ~/.bashrc
cd ~
mkdir ~/.kube
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
}
```

#### journalctl - log tracing

should start all component by 5 to 10 minutes, waiting etcd & api server

```shell
{
journalctl -f -u rke2-server.service
}
```


#### Replace RKE2 Kubeconfig IP & Copy Kubeconfig to default kubeconfig

```shell
{
K8S_VIP_IP="10.10.52.11"
sed -i "s/127.0.0.1/${K8S_VIP_IP}/g" /etc/rancher/rke2/rke2.yaml
mkdir -p $HOME/.kube
sudo cp -f /etc/rancher/rke2/rke2.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
sudo cp $HOME/.kube/config $HOME/.kube/local-kubernetes.conf
cd /var/lib/rancher/rke2/bin
cat /etc/rancher/rke2/rke2.yaml
}
```

#### Get Node

```shell
{
/var/lib/rancher/rke2/bin/kubectl get nodes \
  --kubeconfig /etc/rancher/rke2/rke2.yaml 
/var/lib/rancher/rke2/bin/kubectl get pod -n kube-system \
  --kubeconfig /etc/rancher/rke2/rke2.yaml 
}
```


#### Get Token

```shell
{
cat /var/lib/rancher/rke2/server/node-token
}
```

##### Ouput

```shell
{
mkdir -p /var/lib/rancher/rke2/server/
echo 'K10de6805a2203c992b68139334341f700b68afd399c081c78751de4fc7404a9a91::server:18ad75ac20304b27050dd905bd36eb5a688a1a0ac84a9cd71c4c64ab6e92c018' > /var/lib/rancher/rke2/server/node-token
}
```

```shell
root@k-gong-kubeadm2-m-01:/var/lib/rancher/rke2/bin# cat /var/lib/rancher/rke2/server/node-token
K10de6805a2203c992b68139334341f700b68afd399c081c78751de4fc7404a9a91::server:18ad75ac20304b27050dd905bd36eb5a688a1a0ac84a9cd71c4c64ab6e92c018
```



### Install RKE2 - Agent ( Worker-Node )


#### Optional Step
##### (Optional) Set RBD parameter, If Rook-Ceph enabled


```shell
{
modprobe rbd
echo rbd | sudo tee -a /etc/modules
}
```

#### Increase File Watcher ( this command is temporary fix)

failed to create fsnotify watcher: too many open files

```shell
{
sudo sysctl -w fs.inotify.max_user_watches=2099999999
sudo sysctl -w fs.inotify.max_user_instances=2099999999
sudo sysctl -w fs.inotify.max_queued_events=2099999999
}
```

#### Download Air-Gapped Image from RKE2 Release to /var/lib/rancher/rke2/agent/images

##### Download v1.30

download tarball from https://github.com/rancher/rke2/releases/tag/v1.30.2%2Brke2r1

```shell
{
rm -rf /root/images
mkdir -p /root/images
cd /root/images
wget http://10.10.54.4/rke2_1.30.3/rke2-images-cilium.linux-amd64.tar.zst
wget http://10.10.54.4/rke2_1.30.3/rke2-images.linux-amd64.tar.zst
wget http://10.10.54.4/rke2_1.30.3/sha256sum-amd64.txt
wget http://10.10.54.4/rke2_1.30.3/rke2-images-core.linux-amd64.tar.zst
wget http://10.10.54.4/rke2_1.30.3/rke2.linux-amd64.tar.gz
}
```

#### Add Insecure Registry (k-harbor-01.server.maas)

```shell
{
mkdir -p /etc/rancher/rke2
echo 'configs:
  "k-harbor-01.server.maas":
    tls:
      insecure_skip_verify: true' > /etc/rancher/rke2/registries.yaml
}
```


##### Step to rewrite docker.io to k-harbor-01.server.maas

###### Step to Add SelfSign prevent containerd x509 cert error

```shell
{
export HABOR_URL=k-harbor-01.server.maas:443
openssl s_client -connect ${HABOR_URL} -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM > 00-harbor-ca.crt 
sudo cp 00-harbor-ca.crt /usr/local/share/ca-certificates/00-harbor-ca.crt 
sudo update-ca-certificates
curl -k https://${HABOR_URL}
}
```


###### Config Work ( Rewrite Docker.io to proxy)

```shell
{
mkdir -p /etc/rancher/rke2/
PRIVATE_REPO_URL="k-harbor-01.server.maas:443"
cat <<EOF > /etc/rancher/rke2/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-hub.docker.io/\$1"
  hub.docker.io:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-hub.docker.io/\$1"
  ghcr.io:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-ghcr.io/\$1"
  gcr.io:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-gcr.io/\$1"
  quay.io:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-quay.io/\$1"
  registry.k8s.io:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-registry.k8s.io/\$1"
  public.ecr.aws:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-public.ecr.aws/\$1"
  registry.gitlab.com:
    endpoint:
      - "https://${PRIVATE_REPO_URL}"
    rewrite:
      "(.*)": "proxy-registry.gitlab.com/\$1"
EOF
cat /etc/rancher/rke2/registries.yaml
}
```

<!-- ###### Config Work ( Rewrite Docker.io to proxy)

```shell
{
mkdir -p /etc/rancher/rke2/
echo 'mirrors:
  docker.io:
    endpoint:
      - "https://k-harbor-01.server.maas:443"
    rewrite:
      "(.*)": "proxy-hub.docker.io/$1"' > /etc/rancher/rke2/registries.yaml
cat /etc/rancher/rke2/registries.yaml
}
``` -->

#### Create RKE2 Agent Configuration

##### RKE2 - Default Worker (Main)
```shell
{
# Define variables
RKE2_SERVER_IP=10.10.52.11
RKE2_AGENT_TOKEN==K10412177778a68f82194eaf1b6b1a3ff0abd4d1dac88cd8da9746f4757f527aeeb::server:564e6b1c9d7a18f7e0b1cfa1ac7cdfe2
# Generate config.yaml content
mkdir -p /etc/rancher/rke2/
cat <<EOF > /etc/rancher/rke2/config.yaml
server: https://${RKE2_SERVER_IP}:9345
token: ${RKE2_AGENT_TOKEN}
disable:
  - rke2-ingress-nginx
EOF
}
```

##### RKE2 - Default Worker (disable nginx)
```shell
{
# Define variables
RKE2_SERVER_IP=10.10.52.11
RKE2_AGENT_TOKEN=K10412177778a68f82194eaf1b6b1a3ff0abd4d1dac88cd8da9746f4757f527aeeb::server:564e6b1c9d7a18f7e0b1cfa1ac7cdfe2
# Generate config.yaml content
mkdir -p /etc/rancher/rke2/
cat <<EOF > /etc/rancher/rke2/config.yaml
server: https://${RKE2_SERVER_IP}:9345
token: ${RKE2_AGENT_TOKEN}
disable:
  - rke2-ingress-nginx
EOF
}
```

##### RKE2 - Ceph Worker
```shell
{
# Define variables
RKE2_SERVER_IP=10.10.52.11
RKE2_AGENT_TOKEN=K10412177778a68f82194eaf1b6b1a3ff0abd4d1dac88cd8da9746f4757f527aeeb::server:564e6b1c9d7a18f7e0b1cfa1ac7cdfe2
# Generate config.yaml content
mkdir -p /etc/rancher/rke2/
cat <<EOF > /etc/rancher/rke2/config.yaml
server: https://${RKE2_SERVER_IP}:9345
token: ${RKE2_AGENT_TOKEN}
node-label:
  - "storage-node=true"
# disable:
#   - rke2-ingress-nginx
EOF
}
```

##### RKE2 - Ceph Worker
```shell
{
# Define variables
RKE2_SERVER_IP=10.10.52.11
RKE2_AGENT_TOKEN=K10412177778a68f82194eaf1b6b1a3ff0abd4d1dac88cd8da9746f4757f527aeeb::server:564e6b1c9d7a18f7e0b1cfa1ac7cdfe2
# Generate config.yaml content
mkdir -p /etc/rancher/rke2/
cat <<EOF > /etc/rancher/rke2/config.yaml
server: https://${RKE2_SERVER_IP}:9345
token: ${RKE2_AGENT_TOKEN}
node-label:
  - "storage-node=true"
disable:
  - rke2-ingress-nginx
EOF
}
```





#### Install RKE2 Systemd & Start Systemd

download https://get.rke2.io/ and rename to rke2_install.sh

```shell
{
export TARGET_RKE_VERSION='v1.30.3+rke2r1'
curl -sfL http://10.10.54.4/rke2_1.30.3/get.rke2.io --output install.sh
sleep 1
chmod 755 ./install.sh
sleep 1
INSTALL_RKE2_ARTIFACT_PATH=/root/images INSTALL_RKE2_VERSION="${TARGET_RKE_VERSION}" INSTALL_RKE2_TYPE="agent" ./install.sh
sleep 2
systemctl enable rke2-agent.service
sleep 1
systemctl start rke2-agent.service
sleep 10 
rm -f /var/lib/rancher/rke2/agent/images/*
}
```


#### journalctl - log tracing

should start all component by 5 to 10 minutes

```shell
{
journalctl -f -u rke2-agent.service
}
```
## Uninstall RKE2

```shell
{
/usr/local/bin/rke2-killall.sh
sleep 1
/usr/local/bin/rke2-uninstall.sh
sleep 1
reboot
}
```

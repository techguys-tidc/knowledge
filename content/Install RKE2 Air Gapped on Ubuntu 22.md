
- [Limitation \& References](#limitation--references)
- [Step](#step)
  - [Install RKE2](#install-rke2)
    - [Install RKE2 - Server (Master-Node)](#install-rke2---server-master-node)
      - [Enable Firewall on port 6443](#enable-firewall-on-port-6443)
      - [Download Air-Gapped Image from RKE2 Release to /var/lib/rancher/rke2/agent/images](#download-air-gapped-image-from-rke2-release-to-varlibrancherrke2agentimages)
        - [Download v1.30](#download-v130)
        - [Download v1.29](#download-v129)
      - [Create RKE2 Server - Configuration File](#create-rke2-server---configuration-file)
      - [Add Insecure Registry (k-harbor-01.maas)](#add-insecure-registry-k-harbor-01maas)
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
          - [Patch Ceph Image](#patch-ceph-image)
          - [Create Ceph Cluster](#create-ceph-cluster)
          - [Ceph toolbox](#ceph-toolbox)
        - [(Optional) External-DNS](#optional-external-dns)
          - [Add External DNS HelmChart](#add-external-dns-helmchart)
      - [Install RKE2 Systemd \& Start Systemd](#install-rke2-systemd--start-systemd)
      - [journalctl - log tracing](#journalctl---log-tracing)
      - [Replace RKE2 Kubeconfig IP \& Copy Kubeconfig to default kubeconfig](#replace-rke2-kubeconfig-ip--copy-kubeconfig-to-default-kubeconfig)
    - [Install RKE2 - Agent ( Worker-Node )](#install-rke2---agent--worker-node-)
      - [Optional Step](#optional-step)
        - [(Optional) Set RBD parameter, If Rook-Ceph enabled](#optional-set-rbd-parameter-if-rook-ceph-enabled)
      - [Download Air-Gapped Image from RKE2 Release to /var/lib/rancher/rke2/agent/images](#download-air-gapped-image-from-rke2-release-to-varlibrancherrke2agentimages-1)
        - [Download v1.30](#download-v130-1)
        - [Download v1.29](#download-v129-1)
      - [Create Token on RKE2 Server](#create-token-on-rke2-server)
      - [Create RKE2 Agent Configuration](#create-rke2-agent-configuration)
      - [Add Insecure Registry (k-harbor-01.maas)](#add-insecure-registry-k-harbor-01maas-1)
      - [Install RKE2 Systemd \& Start Systemd](#install-rke2-systemd--start-systemd-1)
      - [journalctl - log tracing](#journalctl---log-tracing-1)
    - [Test](#test)
    - [Test](#test-1)
      - [Add Network Param](#add-network-param)




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
sudo ufw allow 6443/tcp
}
```



#### Download Air-Gapped Image from RKE2 Release to /var/lib/rancher/rke2/agent/images

##### Download v1.30

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

##### Download v1.29

download tarball from https://github.com/rancher/rke2/releases/tag/v1.30.2%2Brke2r1

```shell
{
rm -rf /var/lib/rancher/rke2/agent/images/
mkdir -p /var/lib/rancher/rke2/agent/images/
cd /var/lib/rancher/rke2/agent/images/
wget http://10.10.54.4/rke2_1.29/rke2-images-cilium.linux-amd64.tar.gz
wget http://10.10.54.4/rke2_1.29/rke2-images-core.linux-amd64.tar.gz
wget http://10.10.54.4/rke2_1.29/rke2-images.linux-amd64.tar.gz
wget http://10.10.54.4/rke2_1.29/rke2.linux-amd64.tar.gz
wget http://10.10.54.4/rke2_1.29/sha256sum-amd64.txt
######### OPTIONAL INSTALL CANAL
wget http://10.10.54.4/rke2_1.29/rke2-images-canal.linux-amd64.tar.gz
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
  chart: oci://k-harbor-01.maas/helm-chart/metallb
  version: 0.14.5
  targetNamespace: metallb-system
  createNamespace: true
  # authSecret:
  #   name: example-repo-auth
  repoCAConfigMap:
    name: k-harbor-01-maas-ca
  valuesContent: |-
    controller:
      image:
        repository: k-harbor-01.maas/metallb_image/controller
        tag: v0.14.5
        pullPolicy:

    speaker:
      image:
        repository: k-harbor-01.maas/metallb_image/speaker
        tag: v0.14.5
        pullPolicy:

      frr:
        image:
          repository: k-harbor-01.maas/metallb_image/frr
          tag: 9.0.2
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
    MIIDqzCCApOgAwIBAgIUOHgHCowkVQLwCsWcU9czYagU2CIwDQYJKoZIhvcNAQEL
    BQAwZjELMAkGA1UEBhMCWFgxDDAKBgNVBAgMA04vQTEMMAoGA1UEBwwDTi9BMSAw
    HgYDVQQKDBdTZWxmLXNpZ25lZCBjZXJ0aWZpY2F0ZTEZMBcGA1UEAwwQay1oYXJi
    b3ItMDEubWFhczAeFw0yMzExMTYwNTU5NDlaFw0zMzExMTMwNTU5NDlaMGYxCzAJ
    BgNVBAYTAlhYMQwwCgYDVQQIDANOL0ExDDAKBgNVBAcMA04vQTEgMB4GA1UECgwX
    U2VsZi1zaWduZWQgY2VydGlmaWNhdGUxGTAXBgNVBAMMEGstaGFyYm9yLTAxLm1h
    YXMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC770WZ9giux/veOx/X
    EDiCEnuT/6Bm4pSxTjrlpVl4o4RRQ4O3ylQ2pP7ZiXkfMmInP5rDkAGmZTDFbKhZ
    lMy+zjv65cKOyIgfYKVneEprwEpNA/htDrbuDw1o0JnglO8o+NIGYrNqBHQie0nA
    vIvTTkBq1p+h0u1LdQOhC6YZKm5su20ynRD+Htty/4goA0y1DF+/WCIIPSQa8qkf
    Vg5+jYUogYkIay1twqGb+988ar7VOb9gRjkiq86Qtp+AvpNHbT01BMmksXcjsHZ5
    zDaLeFc49flD16t6lVkK0LHxUoUzwp/t//GmilAptUWMT24yrmmKsSvJMfOUKeMg
    sQXNAgMBAAGjUTBPMC4GA1UdEQQnMCWCEGstaGFyYm9yLTAxLm1hYXOCC2staGFy
    Ym9yLTAxhwQKCjbxMB0GA1UdDgQWBBR84z7v0eKpOZRfl3y7Fwiby+WruDANBgkq
    hkiG9w0BAQsFAAOCAQEAfu2nurgBVhUdjXg6sLR4POyQ22I70BoD8LlrcrYBtpJ4
    7jORP/j/6y4elBjy8BljE9NpCyRuNnvMAQpKwuqWSyS+QZlEZud0H9ym4RvRi6dW
    B1Ko9ELUws+5HA0LmMza2pBIEedwJp9GVv6cjCm6eH2thr143QeUK6SyKIDFtG+L
    GZNhtlkVt6g66UP+bYqdUWsRQ3NsnjX3fq0fFB3LPI5wZT1Gy8qseFH6Y/oe0bDs
    mlBKPyoPJEnRzreoTs9qiDIoIe/ehxTzqeBh9bn81OdVRW54F/XO79G+wv+05Bko
    bVu0od+aCjVCdaRmO7Tzq9MGEovZt8d7sUM/kHuN1g==
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
  chart: oci://k-harbor-01.maas/helm-chart/rook-ceph
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
      repository: k-harbor-01.maas/rook_image/ceph
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
    MIIDqzCCApOgAwIBAgIUOHgHCowkVQLwCsWcU9czYagU2CIwDQYJKoZIhvcNAQEL
    BQAwZjELMAkGA1UEBhMCWFgxDDAKBgNVBAgMA04vQTEMMAoGA1UEBwwDTi9BMSAw
    HgYDVQQKDBdTZWxmLXNpZ25lZCBjZXJ0aWZpY2F0ZTEZMBcGA1UEAwwQay1oYXJi
    b3ItMDEubWFhczAeFw0yMzExMTYwNTU5NDlaFw0zMzExMTMwNTU5NDlaMGYxCzAJ
    BgNVBAYTAlhYMQwwCgYDVQQIDANOL0ExDDAKBgNVBAcMA04vQTEgMB4GA1UECgwX
    U2VsZi1zaWduZWQgY2VydGlmaWNhdGUxGTAXBgNVBAMMEGstaGFyYm9yLTAxLm1h
    YXMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC770WZ9giux/veOx/X
    EDiCEnuT/6Bm4pSxTjrlpVl4o4RRQ4O3ylQ2pP7ZiXkfMmInP5rDkAGmZTDFbKhZ
    lMy+zjv65cKOyIgfYKVneEprwEpNA/htDrbuDw1o0JnglO8o+NIGYrNqBHQie0nA
    vIvTTkBq1p+h0u1LdQOhC6YZKm5su20ynRD+Htty/4goA0y1DF+/WCIIPSQa8qkf
    Vg5+jYUogYkIay1twqGb+988ar7VOb9gRjkiq86Qtp+AvpNHbT01BMmksXcjsHZ5
    zDaLeFc49flD16t6lVkK0LHxUoUzwp/t//GmilAptUWMT24yrmmKsSvJMfOUKeMg
    sQXNAgMBAAGjUTBPMC4GA1UdEQQnMCWCEGstaGFyYm9yLTAxLm1hYXOCC2staGFy
    Ym9yLTAxhwQKCjbxMB0GA1UdDgQWBBR84z7v0eKpOZRfl3y7Fwiby+WruDANBgkq
    hkiG9w0BAQsFAAOCAQEAfu2nurgBVhUdjXg6sLR4POyQ22I70BoD8LlrcrYBtpJ4
    7jORP/j/6y4elBjy8BljE9NpCyRuNnvMAQpKwuqWSyS+QZlEZud0H9ym4RvRi6dW
    B1Ko9ELUws+5HA0LmMza2pBIEedwJp9GVv6cjCm6eH2thr143QeUK6SyKIDFtG+L
    GZNhtlkVt6g66UP+bYqdUWsRQ3NsnjX3fq0fFB3LPI5wZT1Gy8qseFH6Y/oe0bDs
    mlBKPyoPJEnRzreoTs9qiDIoIe/ehxTzqeBh9bn81OdVRW54F/XO79G+wv+05Bko
    bVu0od+aCjVCdaRmO7Tzq9MGEovZt8d7sUM/kHuN1g==
    -----END CERTIFICATE-----
  " > /var/lib/rancher/rke2/server/manifests/${CHART_FILENAME}
}
```

###### Patch Ceph Image

```shell
{
kubectl get configmaps rook-ceph-operator-config -o yaml | grep "IMAGE:"
echo "================================================="
cat << EOF > ./patch_ceph_image.yaml
data:
  ROOK_CSI_ATTACHER_IMAGE: k-harbor-01.maas/k8s-csi/csi-attacher:v4.5.1
  ROOK_CSI_PROVISIONER_IMAGE: k-harbor-01.maas/k8s-csi/csi-provisioner:v4.0.1
  ROOK_CSI_REGISTRAR_IMAGE: k-harbor-01.maas/k8s-csi/csi-node-driver-registrar:v2.10.1
  ROOK_CSI_RESIZER_IMAGE: k-harbor-01.maas/k8s-csi/csi-resizer:v1.10.1
  ROOK_CSI_SNAPSHOTTER_IMAGE: k-harbor-01.maas/k8s-csi/csi-snapshotter:v7.0.2
  ROOK_CSI_CEPH_IMAGE: k-harbor-01.maas/ceph_image/cephcsi:v3.11.0
  ROOK_CSIADDONS_IMAGE: k-harbor-01.maas/ceph_image/k8s-sidecar:v0.8.0
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
  name: my-cluster
  namespace: rook-ceph # namespace:cluster
spec:
  dataDirHostPath: /var/lib/rook
  cephVersion:
    image: k-harbor-01.maas/ceph_image/ceph:v18.2.2
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

###### Ceph toolbox

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
          image: k-harbor-01.maas/ceph_image/ceph:v18.2.2
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
echo "
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: external-dns
  namespace: kube-system
spec:
  chart: oci://k-harbor-01.maas/helm-chart/external-dns
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
      repository: k-harbor-01.maas/externaldns_image/external-dns
      tag: v0.14.2
    # The DNS provider to use. This example uses RFC2136.
    provider: rfc2136
    # Configuration specific to the RFC2136 provider.
    rfc2136:
      # The URL of the DNS server to use.
      server: "10.10.0.8"
      # The port of the DNS server to use.
      port: 53
      # The name of the zone to manage.
      zone: "ingress.maas"
    # List of sources from which ExternalDNS should read the records.
    sources:
      - service
      - ingress
    # The interval at which to synchronize the records.
    interval: "1m"
    # The domain filters to use. This restricts external-dns to only manage records
    # within these domains.
    domainFilters:
      - ingress.maas
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
      - --rfc2136-host=10.10.0.8
      - --rfc2136-port=53
      - --rfc2136-zone=ingress.maas
      - --rfc2136-tsig-secret=GH6RDefrrZngKZZs57eTVf/tGWiqLAUSoPOPaZVFFLk=
      - --rfc2136-tsig-keyname=externaldns-key
      - --rfc2136-tsig-secret-alg=hmac-sha256
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
    MIIDqzCCApOgAwIBAgIUOHgHCowkVQLwCsWcU9czYagU2CIwDQYJKoZIhvcNAQEL
    BQAwZjELMAkGA1UEBhMCWFgxDDAKBgNVBAgMA04vQTEMMAoGA1UEBwwDTi9BMSAw
    HgYDVQQKDBdTZWxmLXNpZ25lZCBjZXJ0aWZpY2F0ZTEZMBcGA1UEAwwQay1oYXJi
    b3ItMDEubWFhczAeFw0yMzExMTYwNTU5NDlaFw0zMzExMTMwNTU5NDlaMGYxCzAJ
    BgNVBAYTAlhYMQwwCgYDVQQIDANOL0ExDDAKBgNVBAcMA04vQTEgMB4GA1UECgwX
    U2VsZi1zaWduZWQgY2VydGlmaWNhdGUxGTAXBgNVBAMMEGstaGFyYm9yLTAxLm1h
    YXMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC770WZ9giux/veOx/X
    EDiCEnuT/6Bm4pSxTjrlpVl4o4RRQ4O3ylQ2pP7ZiXkfMmInP5rDkAGmZTDFbKhZ
    lMy+zjv65cKOyIgfYKVneEprwEpNA/htDrbuDw1o0JnglO8o+NIGYrNqBHQie0nA
    vIvTTkBq1p+h0u1LdQOhC6YZKm5su20ynRD+Htty/4goA0y1DF+/WCIIPSQa8qkf
    Vg5+jYUogYkIay1twqGb+988ar7VOb9gRjkiq86Qtp+AvpNHbT01BMmksXcjsHZ5
    zDaLeFc49flD16t6lVkK0LHxUoUzwp/t//GmilAptUWMT24yrmmKsSvJMfOUKeMg
    sQXNAgMBAAGjUTBPMC4GA1UdEQQnMCWCEGstaGFyYm9yLTAxLm1hYXOCC2staGFy
    Ym9yLTAxhwQKCjbxMB0GA1UdDgQWBBR84z7v0eKpOZRfl3y7Fwiby+WruDANBgkq
    hkiG9w0BAQsFAAOCAQEAfu2nurgBVhUdjXg6sLR4POyQ22I70BoD8LlrcrYBtpJ4
    7jORP/j/6y4elBjy8BljE9NpCyRuNnvMAQpKwuqWSyS+QZlEZud0H9ym4RvRi6dW
    B1Ko9ELUws+5HA0LmMza2pBIEedwJp9GVv6cjCm6eH2thr143QeUK6SyKIDFtG+L
    GZNhtlkVt6g66UP+bYqdUWsRQ3NsnjX3fq0fFB3LPI5wZT1Gy8qseFH6Y/oe0bDs
    mlBKPyoPJEnRzreoTs9qiDIoIe/ehxTzqeBh9bn81OdVRW54F/XO79G+wv+05Bko
    bVu0od+aCjVCdaRmO7Tzq9MGEovZt8d7sUM/kHuN1g==
    -----END CERTIFICATE-----
  " > /var/lib/rancher/rke2/server/manifests/${CHART_FILENAME}
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


### Install RKE2 - Agent ( Worker-Node )


#### Optional Step
##### (Optional) Set RBD parameter, If Rook-Ceph enabled


```shell
{
modprobe rbd
}
```

#### Download Air-Gapped Image from RKE2 Release to /var/lib/rancher/rke2/agent/images

##### Download v1.30

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

##### Download v1.29

download tarball from https://github.com/rancher/rke2/releases/tag/v1.30.2%2Brke2r1

```shell
{
rm -rf /var/lib/rancher/rke2/agent/images/
mkdir -p /var/lib/rancher/rke2/agent/images/
cd /var/lib/rancher/rke2/agent/images/
wget http://10.10.54.4/rke2_1.29/rke2-images-cilium.linux-amd64.tar.gz
wget http://10.10.54.4/rke2_1.29/rke2-images-core.linux-amd64.tar.gz
wget http://10.10.54.4/rke2_1.29/rke2-images.linux-amd64.tar.gz
wget http://10.10.54.4/rke2_1.29/rke2.linux-amd64.tar.gz
wget http://10.10.54.4/rke2_1.29/sha256sum-amd64.txt
######### OPTIONAL INSTALL CANAL
wget http://10.10.54.4/rke2_1.29/rke2-images-canal.linux-amd64.tar.gz
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
K10e4e3316ff062b417c3f19b31838f15650bbca5e480de1d08b147fa5173c866dc::h0b3in.bom7edr4s00fvb6w
root@k-gong-kubeadm2-m-01:/etc/rancher/rke2# 
```

#### Create RKE2 Agent Configuration

```shell
{
# Define variables
RKE2_SERVER_IP=10.10.52.11
RKE2_AGENT_TOKEN=K10e4e3316ff062b417c3f19b31838f15650bbca5e480de1d08b147fa5173c866dc::h0b3in.bom7edr4s00fvb6w
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


### Test


  

echo "
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: grafana
  namespace: kube-system
spec:
  createNamespace: true
  chart: grafana
  repo: https://grafana.github.io/helm-charts
  targetNamespace: monitoring
  set:
    adminPassword: "NotVerySafePassword"
  valuesContent: |-
    image:
      tag: master
    env:
      GF_EXPLORE_ENABLED: true
    adminUser: admin
    sidecar:
      datasources:
        enabled: true" > /var/lib/rancher/rke2/server/manifests/rke2-grafana.yaml




### Test


docker pull --insecure-registry=k-harbor-01.maas k-harbor-01.maas/rancher_image/rancher:latest
sudo docker run --privileged -d --restart=unless-stopped -p 443:443 k-harbor-01.maas/rancher_image/rancher



docker logs $(docker ps  | grep rancher | awk '{print $1}') 2>&1 | grep "Bootstrap Password:"

2024/07/19 06:50:18 [INFO] Bootstrap Password: xnrj44k2mfxf2cj7k79xrtf8wpqrssfchnjphklz5xfvcdkwrm8nbs

admin password : tamJ0uIQ6yJETfOW



#### Add Network Param

```shell
{
modprobe br_netfilter
modprobe overlay
cat <<EOF | tee /etc/modules-load.d/k8s.conf
br_netfilter
overlay
EOF
}
```

```shell
{
cat <<EOF | tee /etc/sysctl.conf
net.ipv4.ip_forward=1
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.default.send_redirects=0
net.ipv4.conf.default.accept_source_route=0
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.default.accept_redirects=0
net.ipv4.conf.all.log_martians=1
net.ipv4.conf.default.log_martians=1
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.default.rp_filter=1
net.ipv6.conf.all.accept_ra=0
net.ipv6.conf.default.accept_ra=0
net.ipv6.conf.all.accept_redirects=0
net.ipv6.conf.default.accept_redirects=0
kernel.keys.root_maxbytes=25000000
kernel.keys.root_maxkeys=1000000
kernel.panic=10
kernel.panic_on_oops=1
vm.overcommit_memory=1
vm.panic_on_oom=0
net.ipv4.ip_local_reserved_ports=30000-32767
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-arptables=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
sleep 2
sysctl --system
}
```

```shell
{
echo '
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
  chain input {
    type filter hook input priority filter; policy accept;
    tcp dport 22 accept;
    ct state established,related accept;
    iifname "lo" accept;
    ip protocol icmp accept;
    ip daddr 8.8.8.8 tcp dport 53 accept;
    ip daddr 8.8.8.8 udp dport 53 accept;
    ip daddr 8.8.4.4 tcp dport 53 accept;
    ip daddr 8.8.4.4 udp dport 53 accept;
    ip daddr 1.1.1.1 tcp dport 53 accept;
    ip daddr 1.1.1.1 udp dport 53 accept;
    ip saddr 192.168.100.11 accept;
    ip saddr 192.168.100.12 accept;
    ip saddr 192.168.100.13 accept;
    ip saddr 192.168.100.14 accept;
    ip saddr 192.168.100.15 accept;
    ip saddr 192.168.100.16 accept;
    ip saddr 192.168.100.100 tcp dport {9345,6443,443,80} accept;
    ip saddr 192.168.100.100 udp dport {9345,6443,443,80} accept;
    counter packets 0 bytes 0 drop;
  }

  chain forward {
    type filter hook forward priority filter; policy accept;
  }

  chain output {
    type filter hook output priority filter; policy accept;
  }
}' > /etc/nftables.conf
sleep 1
systemctl enable nftables
sleep 1
systemctl restart nftables
}
```
- [Revision : 2024-08-16](#revision--2024-08-16)
- [Prepare Node](#prepare-node)
    - [Install Kubelet Kubeadm and Crio 1.30](#install-kubelet-kubeadm-and-crio-130)
    - [Turn off Swap](#turn-off-swap)
    - [Add Parameter #1](#add-parameter-1)
    - [Add Parameter #2](#add-parameter-2)
    - [Add Insecure Registry](#add-insecure-registry)
    - [Add PauseImage](#add-pauseimage)
    - [start crio service](#start-crio-service)
    - [Create Kubeadm.yaml](#create-kubeadmyaml)
    - [Prepare Habor](#prepare-habor)
    - [Pull Image](#pull-image)
    - [Init Cluster](#init-cluster)
    - [Copy Kubeconfig](#copy-kubeconfig)
    - [Install CNI : Cillium](#install-cni--cillium)
      - [Install Cillium CLI](#install-cillium-cli)
    - [Install rancher local-path Storage Classs](#install-rancher-local-path-storage-classs)
    - [Install Instio Operator](#install-instio-operator)
      - [install istioctl](#install-istioctl)
      - [istioctl precheck](#istioctl-precheck)
      - [istioctl dry-run](#istioctl-dry-run)
      - [istioctl operator install](#istioctl-operator-install)
      - [Install istio](#install-istio)
      - [Optional - Install MetalLB](#optional---install-metallb)
        - [Prepare Helm Custom values.yaml](#prepare-helm-custom-valuesyaml)
          - [Install Metallb](#install-metallb)
        - [l2 advertisement](#l2-advertisement)
          - [create ip poll \& l2-advertisement](#create-ip-poll--l2-advertisement)
      - [Optional - Deploy Istio - Grafana Addon](#optional---deploy-istio---grafana-addon)
          - [Deploy Grafana Addon](#deploy-grafana-addon)
          - [Deploy Grafana Ingress GATEWAY](#deploy-grafana-ingress-gateway)
        - [Deploy Grafana Virtual Service](#deploy-grafana-virtual-service)
        - [Client Map /etc/hosts](#client-map-etchosts)
          - [Deploy Prometheus Addon](#deploy-prometheus-addon)
      - [Optional - Loadtest using K6](#optional---loadtest-using-k6)
        - [Ref - How to write K6 Script](#ref---how-to-write-k6-script)
        - [Enable Istio Injection](#enable-istio-injection)
        - [K6 - Deploy](#k6---deploy)
        - [K6 Files](#k6-files)
        - [Deploy Echo Server](#deploy-echo-server)
      - [Optional - Step Install httpbin](#optional---step-install-httpbin)
          - [Deploy HTTPBIN](#deploy-httpbin)
          - [Deploy ISTIO GATEWAY](#deploy-istio-gateway)
        - [Deploy Istio Virtual Service](#deploy-istio-virtual-service)
        - [Client Map /etc/hosts](#client-map-etchosts-1)
        - [Test Ingress Gateway](#test-ingress-gateway)
      - [Optional - Step Install BookInfo](#optional---step-install-bookinfo)
          - [Deploy BookInfo](#deploy-bookinfo)
        - [Deploy Bookinfo Istio Gateway](#deploy-bookinfo-istio-gateway)
      - [Reset KubeADM](#reset-kubeadm)
          - [Reset KubeADM](#reset-kubeadm-1)


# Revision : 2024-08-16


# Prepare Node

### Install Kubelet Kubeadm and Crio 1.30





```shell
{
KUBERNETES_VERSION=v1.30
PROJECT_PATH=prerelease:/main
apt-get update
apt-get install -y software-properties-common curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list
apt-get update
apt-get install -y cri-o kubelet kubeadm kubectl
}
```


### Turn off Swap

```
{
sudo swapoff -a
sudo sed -i.bak -r 's/(.+swap.+)/#\1/' /etc/fstab
}
```

### Add Parameter #1

```
{
# Enable kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter
# Add some settings to sysctl
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
# Reload sysctl
sleep 2
sudo sysctl --system
}
```

### Add Parameter #2

```
{
# Configure persistent loading of modules
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
sleep 2
sudo sysctl --system
}
```





### Add Insecure Registry

```shell
{
config_file="/etc/crio/crio.conf.d/10-crio.conf"
registry1="k-harbor-01.maas"
registry2="10.10.54.241"
sed -i "/^\[crio\.image\]/a insecure_registries = \[\"$registry1\",\"$registry2\"\]" "$config_file"
sleep 1
grep crio.image -A1 ${config_file}
}
# Use sed to add the parameter under crio.image section
```

### Add PauseImage

```shell
{
pause_image="k-harbor-01.maas/kubeadm_image/pause:3.9"
# Define the configuration file
config_file="/etc/crio/crio.conf.d/10-crio.conf"
# Single sed command to add or update pause_image property
sed -i "/^\[crio\.image\]/ { /pause_image/ { s|pause_image.*=.*|pause_image = \"$pause_image\"|; b }; s|\[crio\.image\]|&\npause_image = \"$pause_image\"| }" "$config_file"
sleep 1
grep crio.image -A1 ${config_file}
}
```


output

```
[crio.image]
 insecure_registries = \["k-harbor-01.maas","10.10.54.241"\]
pause_image="k-harbor-01.maas/kubeadm-proxy/pause:3.9"
```


### start crio service

```shell
{
systemctl stop crio.service
sleep 1
systemctl start crio.service
sleep 2
systemctl status crio.service
}
```

### Create Kubeadm.yaml


```shell
{
COREDNS_IMAGE_REPOSITORY="k-harbor-01.maas/kubeadm_image"
K8S_IMAGE_REPOSITORY="k-harbor-01.maas/kubeadm_image"
MASTER_1_IP="10.10.51.51"
MASTER_1_HOSTNAME="k-gong-kubeadm2-m-01"
cat <<EOF > cluster-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: ${MASTER_1_IP}
  bindPort: 6443
skipPhases:
  - addon/kube-proxy
nodeRegistration:
  criSocket: unix:///var/run/crio/crio.sock
  imagePullPolicy: IfNotPresent
  name: ${MASTER_1_HOSTNAME}
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {imageRepository: ${COREDNS_IMAGE_REPOSITORY}}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: "${K8S_IMAGE_REPOSITORY}"
kind: ClusterConfiguration
kubernetesVersion: 1.30.2
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 192.168.0.0/16
scheduler: {}
#---
#apiVersion: kubeproxy.config.k8s.io/v1alpha1
#kind: KubeProxyConfiguration
#mode: ipvs
EOF
}
```

### Prepare Habor


### Pull Image

pull image from custom config

```shell
{
kubeadm config images pull --config=./cluster-config.yaml
}
```

pull image from default configuration

```shell
{
kubeadm config images pull
}
```

output

```shell
root@k-gong-kubeadm2-m-01:~# kubeadm config images pull
[config/images] Pulled registry.k8s.io/kube-apiserver:v1.30.2
[config/images] Pulled registry.k8s.io/kube-controller-manager:v1.30.2
[config/images] Pulled registry.k8s.io/kube-scheduler:v1.30.2
[config/images] Pulled registry.k8s.io/kube-proxy:v1.30.2
[config/images] Pulled registry.k8s.io/coredns/coredns:v1.11.1
[config/images] Pulled registry.k8s.io/pause:3.9
[config/images] Pulled registry.k8s.io/etcd:3.5.12-0
root@k-gong-kubeadm2-m-01:~# 
root@k-gong-kubeadm2-m-01:~# kubeadm config images pull --config=./cluster-config.yaml
[config/images] Pulled k-harbor-01.maas/kubeadm_image/kube-apiserver:v1.30.2
[config/images] Pulled k-harbor-01.maas/kubeadm_image/kube-controller-manager:v1.30.2
[config/images] Pulled k-harbor-01.maas/kubeadm_image/kube-scheduler:v1.30.2
[config/images] Pulled k-harbor-01.maas/kubeadm_image/kube-proxy:v1.30.2
[config/images] Pulled k-harbor-01.maas/kubeadm_image/coredns:v1.11.1
[config/images] Pulled k-harbor-01.maas/kubeadm_image/pause:3.9
[config/images] Pulled k-harbor-01.maas/kubeadm_image/etcd:3.5.12-0
root@k-gong-kubeadm2-m-01:~# 
```

### Init Cluster

```shell
{
sudo kubeadm init --config=./cluster-config.yaml --upload-certs
}
```

```shell
kubeadm join 10.10.51.51:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:faf390926238898c609f451da09385547816beff40f197ed9d69573b5e7e0833
```

### Copy Kubeconfig

```shell
{
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
sudo cp $HOME/.kube/config $HOME/.kube/local-kubernetes.conf
}
```

### Install CNI : Cillium

#### Install Cillium CLI


```shell
{
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
}
```

cilium values.yaml

```
image:
  repository: "k-harbor-01.maas/cilium_image/cilium"
  useDigest: false
operator:
  image:
    repository: "k-harbor-01.maas/cilium_image/operator"
    useDigest: false
```

cilium install -f values.yaml --version 1.15.7


```
root@k-gong-kubeadm2-m-01:/home/nattawat.ujj/cilium# cilium install --version 1.15.7 --dry-run | grep -i image:
  sidecar-istio-proxy-image: "cilium/istio_proxy"
        image: "quay.io/cilium/cilium:v1.15.6@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "quay.io/cilium/cilium:v1.15.6@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "quay.io/cilium/cilium:v1.15.6@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "quay.io/cilium/cilium:v1.15.6@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "quay.io/cilium/cilium:v1.15.6@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "quay.io/cilium/cilium:v1.15.6@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "quay.io/cilium/cilium:v1.15.6@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "quay.io/cilium/operator-generic:v1.15.6@sha256:5789f0935eef96ad571e4f5565a8800d3a8fbb05265cf6909300cd82fd513c3d"
root@k-gong-kubeadm2-m-01:/home/nattawat.ujj/cilium# cilium install -f values.yaml --version 1.15.7 --dry-run | grep -i image:
  sidecar-istio-proxy-image: "cilium/istio_proxy"
        image: "k-harbor-01.maas/cilium_image/cilium:v1.15.7@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "k-harbor-01.maas/cilium_image/cilium:v1.15.7@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "k-harbor-01.maas/cilium_image/cilium:v1.15.7@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "k-harbor-01.maas/cilium_image/cilium:v1.15.7@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "k-harbor-01.maas/cilium_image/cilium:v1.15.7@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "k-harbor-01.maas/cilium_image/cilium:v1.15.7@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "k-harbor-01.maas/cilium_image/cilium:v1.15.7@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def"
        image: "k-harbor-01.maas/cilium_image/operator-generic:v1.15.7@sha256:5789f0935eef96ad571e4f5565a8800d3a8fbb05265cf6909300cd82fd513c3d"
root@k-gong-kubeadm2-m-01:/home/nattawat.ujj/cilium# 
```

### Install rancher local-path Storage Classs

```shell
{
RANCHER_LOCALPATH_VERSION="v0.0.28"
IMAGE_RANCHER_LOCALPATHSTORAGE="k-harbor-01.maas/rancher_image/local-path-provisioner:${RANCHER_LOCALPATH_VERSION}"
IMAGE_BUSYBOX="k-harbor-01.maas/rancher_image/busybox"
wget -O local-path-storage.yaml https://raw.githubusercontent.com/rancher/local-path-provisioner/${RANCHER_LOCALPATH_VERSION}/deploy/local-path-storage.yaml 2>/dev/null
sleep .3
sed -i -E "s|rancher/local-path-provisioner:v0\.0\.28|$IMAGE_RANCHER_LOCALPATHSTORAGE|g" local-path-storage.yaml
sleep .3
sed -i -E "s|busybox|$IMAGE_BUSYBOX|g" local-path-storage.yaml
sleep .5
kubectl apply -f ./local-path-storage.yaml
sleep .2
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
rm -f ./local-path-storage.yaml
}
```

### Install Instio Operator


#### install istioctl

```shell
{
curl -sL https://istio.io/downloadIstioctl | sh -
sleep .5
echo 'export PATH=$HOME/.istioctl/bin:$PATH'>> ~/.bashrc
}
```


#### istioctl precheck

reload ssh session, you will be able to use istioctl

```shell
{
istioctl x precheck
}
```


#### istioctl dry-run

```shell
{
export ISTIO_IMAGE_REPO="k-harbor-01.maas/istio_image"
istioctl operator init --hub ${ISTIO_IMAGE_REPO} --dry-run
}
```

#### istioctl operator install

```shell
{
export ISTIO_IMAGE_REPO="k-harbor-01.maas/istio_image"
istioctl operator init --hub ${ISTIO_IMAGE_REPO}
}
```

#### Install istio

[Istio Operator - Customize Options](https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/)

```shell
{
export ISTIO_IMAGE_REPO="k-harbor-01.maas/istio_image"
export ISTIO_PROFILE"demo"
kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: ${ISTIO_PROFILE}
  hub: ${ISTIO_IMAGE_REPO}
EOF
}
```

#### Optional - Install MetalLB


##### Prepare Helm Custom values.yaml

```shell
echo "---
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
" > ./values.yaml
```

###### Install Metallb

```shell
{
helm repo add metallb https://metallb.github.io/metallb
#helm install metallb --create-namespace --namespace metallb-system metallb/metallb -f values.yaml --dry-run
helm install metallb --create-namespace --namespace metallb-system metallb/metallb -f values.yaml
}
```

##### l2 advertisement

###### create ip poll & l2-advertisement

```shell
cat <<EOF | kubectl create -f -
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ippool-pxe
  namespace: metallb-system
spec:
  addresses:
  - 10.10.51.80-10.10.51.99
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
```


#### Optional - Deploy Istio - Grafana Addon

###### Deploy Grafana Addon
```shell
{
IMAGE_GRAFANA="k-harbor-01.maas/istio_image/grafana:10.4.0"
wget -O grafana.yaml https://raw.githubusercontent.com/istio/istio/release-1.22/samples/addons/grafana.yaml 2>/dev/null
sleep .2
sed -i -E "s|docker.io/grafana/grafana:10\.4\.0|$IMAGE_GRAFANA|g" grafana.yaml
sleep .5
kubectl apply -f ./grafana.yaml
#sleep .2
#rm -f ./bookinfo.yaml
}
```


###### Deploy Grafana Ingress GATEWAY
```shell
{
HTTPBIN_URL="grafana.labtt1.com"
GATEWAY_NAME="istio-monitoring-gateway"
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ${GATEWAY_NAME}
  namespace: istio-system
spec:
  # The selector matches the ingress gateway pod labels.
  # If you installed Istio using Helm following the standard documentation, this would be "istio=ingress"
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "${HTTPBIN_URL}"
EOF
}
```

##### Deploy Grafana Virtual Service

```shell
{
VIRTUAL_SERVICE_NAME=grafana
HTTPBIN_URL="grafana.labtt1.com"
GATEWAY_NAME="istio-monitoring-gateway"
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ${VIRTUAL_SERVICE_NAME}
  namespace: istio-system
spec:
  hosts:
  - "${HTTPBIN_URL}"
  gateways:
  - ${GATEWAY_NAME}
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        port:
          number: 3000
        host: grafana
EOF
}
```

##### Client Map /etc/hosts

client need to map /etc/hosts. if dns cannot reach

```
# INGRESS Gateway Loagbalancer IP  is 10.10.51.80
10.10.51.80	grafana.labtt1.com
```



###### Deploy Prometheus Addon
```shell
{
IMAGE_PROMETHEUS="k-harbor-01.maas/istio_image/prometheus:v2.51.1"
IMAGE_PROMETHEUS_CONFIG_RELOAD="k-harbor-01.maas/istio_image/prometheus-config-reloader:v0.72.0"
wget -O prometheus.yaml https://raw.githubusercontent.com/istio/istio/release-1.22/samples/addons/prometheus.yaml 2>/dev/null
sleep .2
sed -i -E "s|ghcr.io/prometheus-operator/prometheus-config-reloader:v0\.72\.0|$IMAGE_PROMETHEUS_CONFIG_RELOAD|g" prometheus.yaml
sleep .2
sed -i -E "s|prom/prometheus:v2\.51\.1|$IMAGE_PROMETHEUS|g" prometheus.yaml
sleep .5
kubectl apply -f ./prometheus.yaml
}
```

#### Optional - Loadtest using K6

##### Ref - How to write K6 Script

[K6 Tutorials](https://medium.com/@anshita.bhasin/k6-exploring-the-terms-and-terminologies-698b5271c7be)

[Complex K6](https://stackoverflow.com/questions/75511450/how-to-send-multiple-requests-for-a-duration-in-k6)

##### Enable Istio Injection

```shell
{
kubectl label namespace k6-load-test istio-injection=enabled
}
```

##### K6 - Deploy

```shell
{
IMAGE_K6="k-harbor-01.maas/grafana_image/k6:latest"
kubectl create namespace k6-load-test
sleep .3
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-k6-loadtest-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: k6-load-test
  namespace: k6-load-test
spec:
  containers:
    - name: k6-container
      image: ${IMAGE_K6}
      command: ["sleep", "infinity"]
      volumeMounts:
      - mountPath: "/data"
        name: data
      resources:
        limits:
          cpu: "1"
          memory: "1Gi"
        requests:
          cpu: "0.5"
          memory: "512Mi"
  volumes:
      - name: data
        persistentVolumeClaim:
          claimName: pvc-k6-loadtest-data
  restartPolicy: Never
EOF
# Make It Retain
PV_ID=$(kubectl get pv | grep k6-load-test/pvc-k6-loadtest-data | awk '{print $1}')
kubectl patch pv ${PV_ID} -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
}
```


##### K6 Files

```js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    scenarios: {
        status_200: {
            executor: 'constant-arrival-rate',
            rate: 100, // 100 requests per second
            duration: '5m',
            preAllocatedVUs: 1,
            exec: 'first_endpoint', // Assigning the first_endpoint function as the scenario
        },
        status_401: {
            executor: 'constant-arrival-rate',
            rate: 50, // 50 requests per second
            duration: '5m',
            preAllocatedVUs: 1,
            exec: 'second_endpoint', // Assigning the second_endpoint function as the scenario
        },
    }
};

export function first_endpoint() {
    const url = 'http://log-debug:8080/register/step_02'; // 75%
    const response = http.get(url, { retries: 0 }); // Disable retries

    check(response, {
        'first_endpoint status is 200': (r) => r.status === 200,
        'first_endpoint response time < 200ms': (r) => r.timings.duration < 200,
    });
}

export function second_endpoint() {
    const url = 'http://log-debug:8080/register/step_02?x-set-response-status-code=401'; // Define your second endpoint URL here
    const response = http.get(url, { retries: 0 }); // Disable retries
}
```

##### Deploy Echo Server

```shell
{
IMAGE_ECHOSERVER="k-harbor-01.maas/grafana_image/echo-server:latest"
#kubectl create namespace k6-load-test
sleep .3
kubectl apply -f - <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver
  namespace: k6-load-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echoserver
  template:
    metadata:
      labels:
        app: echoserver
    spec:
      containers:
      - image: ${IMAGE_ECHOSERVER}
        imagePullPolicy: IfNotPresent
        name: echoserver
        ports:
        - containerPort: 80
        env:
        - name: PORT
          value: "80"
---
apiVersion: v1
kind: Service
metadata:
  name: echoserver
  namespace: k6-load-test
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: ClusterIP
  selector:
    app: echoserver
EOF
}
```


#### Optional - Step Install httpbin


###### Deploy HTTPBIN
```shell
{
IMAGE_HTTPBIN="k-harbor-01.maas/istio_image/httpbin"
wget -O httpbin.yaml https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml 2>/dev/null
sleep .2
sed -i -E "s|docker.io/kong/httpbin|$IMAGE_HTTPBIN|g" httpbin.yaml
sleep .5
kubectl create namespace istio-httpbin
sleep .5
kubectl apply -n istio-httpbin -f ./httpbin.yaml
}
```

###### Deploy ISTIO GATEWAY
```shell
{
HTTPBIN_URL="httpbin.example.com"
GATEWAY_NAME="httpbin-gateway"
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ${GATEWAY_NAME}
spec:
  # The selector matches the ingress gateway pod labels.
  # If you installed Istio using Helm following the standard documentation, this would be "istio=ingress"
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "${HTTPBIN_URL}"
EOF
}
```

##### Deploy Istio Virtual Service

```shell
{
HTTPBIN_URL="httpbin.example.com"
GATEWAY_NAME="httpbin-gateway"
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "${HTTPBIN_URL}"
  gateways:
  - ${GATEWAY_NAME}
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    - uri:
        prefix: /headers
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
}
```

##### Client Map /etc/hosts

client need to map /etc/hosts. if dns cannot reach

```
# INGRESS Gateway Loagbalancer IP  is 10.10.51.80
httpbin.example.com 10.10.51.80
```

##### Test Ingress Gateway

```shell
{
curl -v http://httpbin.example.com/headers
curl -v http://httpbin.example.com/status/200
curl -v http://httpbin.example.com/status/503
curl -v http://httpbin.example.com/delay/5
}
```





https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/#before-you-begin

https://github.com/istio/istio/tree/master/samples/httpbin

#### Optional - Step Install BookInfo



###### Deploy BookInfo
```shell
{
IMAGE_BOOKINFO_DETAILS_V1="k-harbor-01.maas/istiobookinfo_image/examples-bookinfo-details-v1:1.20.1"
IMAGE_BOOKINFO_RATINGS_V1="k-harbor-01.maas/istiobookinfo_image/examples-bookinfo-ratings-v1:1.20.1"
IMAGE_BOOKINFO_REVIEWS_V1="k-harbor-01.maas/istiobookinfo_image/examples-bookinfo-reviews-v1:1.20.1"
IMAGE_BOOKINFO_REVIEWS_V2="k-harbor-01.maas/istiobookinfo_image/examples-bookinfo-reviews-v2:1.20.1"
IMAGE_BOOKINFO_REVIEWS_V3="k-harbor-01.maas/istiobookinfo_image/examples-bookinfo-reviews-v3:1.20.1"
IMAGE_BOOKINFO_PRODUCTPAGE_V1="k-harbor-01.maas/istiobookinfo_image/examples-bookinfo-productpage-v1:1.20.1"
wget -O bookinfo.yaml https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml 2>/dev/null
sleep .2
sed -i -E "s|docker.io/istio/examples-bookinfo-details-v1:1\.20\.1|$IMAGE_BOOKINFO_DETAILS_V1|g" bookinfo.yaml
sleep .2
sed -i -E "s|docker.io/istio/examples-bookinfo-ratings-v1:1\.20\.1|$IMAGE_BOOKINFO_RATINGS_V1|g" bookinfo.yaml
sleep .2
sed -i -E "s|docker.io/istio/examples-bookinfo-reviews-v1:1\.20\.1|$IMAGE_BOOKINFO_REVIEWS_V1|g" bookinfo.yaml
sleep .2
sed -i -E "s|docker.io/istio/examples-bookinfo-reviews-v2:1\.20\.1|$IMAGE_BOOKINFO_REVIEWS_V2|g" bookinfo.yaml
sleep .2
sed -i -E "s|docker.io/istio/examples-bookinfo-reviews-v3:1\.20\.1|$IMAGE_BOOKINFO_REVIEWS_V3|g" bookinfo.yaml
sleep .2
sed -i -E "s|docker.io/istio/examples-bookinfo-productpage-v1:1\.20\.1|$IMAGE_BOOKINFO_PRODUCTPAGE_V1|g" bookinfo.yaml
sleep .5
kubectl create namespace istio-bookinfo
sleep .2
kubectl label namespace istio-bookinfo istio-injection=enabled
sleep .2
kubectl apply -n istio-bookinfo -f ./bookinfo.yaml
#sleep .2
#rm -f ./bookinfo.yaml
}
```

##### Deploy Bookinfo Istio Gateway

```shell
{
wget -O bookinfo-gateway.yaml https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/bookinfo-gateway.yaml
sleep .2
kubectl apply -n istio-bookinfo -f ./bookinfo-gateway.yaml
}
```




#### Reset KubeADM
###### Reset KubeADM

```
{
sudo kubeadm reset
echo "sleep 10 seconds"
sleep 10
sudo rm -rf /etc/cni /etc/kubernetes /var/lib/dockershim /var/lib/etcd /var/lib/kubelet /var/lib/etcd2/ /var/run/kubernetes ~/.kube/*
#sudo crictl rmi --all
#sudo crictl status
sudo reboot
}
```
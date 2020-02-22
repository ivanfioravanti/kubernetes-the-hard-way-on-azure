# 쿠버네티스 워커 노드 부트스트랩

이 실습에서는 3개의 쿠버네티스 워커 노드를 부트스트랩합니다. [runc](https://github.com/opencontainers/runc), [컨테이너 네트워킹 플러그인](https://github.com/containernetworking/cni), [cri-containerd](https://github.com/kubernetes-incubator/cri-containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet) 및 [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies) 구성 요소가 각 노드에 설치됩니다.

## 전제 조건

각 워커 인스턴스 (`worker-0`, `worker-1` 및 `worker-2`)에 로그인해서 이 실습에 포함된 내용을 실행해야합니다.
Azure 메타데이터 인스턴스 서비스는 사용자 지정 속성을 설정하는데 사용할 수 없으므로, 대신 각 워커 VM에서 *태그* 를 사용하여 나중에 사용되는 POD-CIDR을 정의했습니다.

새로 만든 컴퓨팅 인스턴스의 POD CIDR 범위를 확인하고, 다른 곳에 메모해둡니다.

```shell
az vm show -g kubernetes --name worker-0 --query "tags" -o tsv
```

> 출력

```shell
10.200.0.0/24
```

각 워커 인스턴스에 로그인하기 위해서는 {code3}az{/code3} 명령을 사용하여 각 컨트롤러 인스턴스의 퍼블릭 IP를 찾아야 하며, 아래와 같이 실행하여 찾을 수 있습니다. 예:

```shell
WORKER="worker-0"
PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
  -n ${WORKER}-pip --query "ipAddress" -otsv)

ssh kuberoot@${PUBLIC_IP_ADDRESS}
```

### tmux로 동시에 여러 명령 실행하기

[tmux](https://github.com/tmux/tmux/wiki)를 사용하여 여러 컴퓨팅 인스턴스에서 동시에 명령을 실행할 수 있습니다. 앞 단계의 [tmux로 동시에 여러 명령 실행하기](01-prerequisites.md#running-commands-in-parallel-with-tmux) 섹션에서 자세한 내용을 확인할 수 있습니다.

## 쿠버네티스 워커 노드 프로비저닝

OS 종속성을 설치합니다.

```shell
{
  sudo apt-get update
  sudo apt-get -y install socat conntrack ipset
}
```

> socat 바이너리는 `kubectl port-forward` 명령을 지원합니다.

### 워커 바이너리 다운로드 및 설치

```shell
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.14.0/crictl-v1.14.0-linux-amd64.tar.gz \
  https://storage.googleapis.com/gvisor/releases/nightly/latest/runsc \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc8/runc.amd64\
  https://github.com/containernetworking/plugins/releases/download/v0.8.1/cni-plugins-linux-amd64-v0.8.1.tgz \
  https://github.com/containerd/containerd/releases/download/v1.2.7/containerd-1.2.7.linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kubelet
```

설치 디렉토리를 만듭니다.

```shell
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

워커 바이너리를 설치합니다.

```shell
{
  sudo mv runc.amd64 runc
  chmod +x kubectl kube-proxy kubelet runc runsc
  sudo mv kubectl kube-proxy kubelet runc runsc /usr/local/bin/
  sudo tar -xvf crictl-v1.14.0-linux-amd64.tar.gz -C /usr/local/bin/
  sudo tar -xvf cni-plugins-linux-amd64-v0.8.1.tgz -C /opt/cni/bin/
  sudo tar -xvf containerd-1.2.7.linux-amd64.tar.gz -C /
}
```

### CNI 네트워킹 구성

POD_CIDR을 Azure VM 태그에서 처음 검색된 주소로 바꾸는 `bridge` 네트워크 구성 파일을 만듭니다. (참고: Azure Metadata Instance Service는 각 작업자의 POD_CIDR 태그를 검색하는 데 사용됩니다. https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/virtual-machines/windows/instance-metadata-service.md)

```shell
POD_CIDR="$(echo $(curl --silent -H Metadata:true "http://169.254.169.254/metadata/instance/compute/tags?api-version=2017-08-01&format=text") | cut -d : -f2)"
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.2.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

`loopback` 네트워크 구성 파일을 만듭니다.

```shell
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.2.0",
    "name": "lo",
    "type": "loopback"
}
EOF
```

### containerd 구성

`containerd` 구성 파일을 만듭니다.

```shell
sudo mkdir -p /etc/containerd/
```

```shell
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
    [plugins.cri.containerd.untrusted_workload_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
    [plugins.cri.containerd.gvisor]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
EOF
```

> 신뢰할 수 없는 워크로드는 gVisor (runsc) 런타임을 사용하여 실행됩니다.

`containerd.service` 시스템 유닛 파일을 만듭니다.

```shell
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd

Delegate=yes
KillMode=process
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### Kubelet 구성

```shell
{
  sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
  sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.pem /var/lib/kubernetes/
}
```

`kubelet-config.yaml` 구성 파일을 만듭니다.

```shell
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

> `resolvConf` 구성은 `systemd-resolved` 실행하는 시스템에서 서비스 발견을 위해 CoreDNS를 사용할 때 루프를 피하기 위해 사용됩니다.

`kubelet.service` 시스템 유닛 파일을 만듭니다.

```shell
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 쿠버네티스 프록시 구성

```shell
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

`kube-proxy-config.yaml` 구성 파일을 만듭니다.

```shell
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

`kube-proxy.service` 시스템 유닛 파일을 만듭니다.

```shell
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 워커 서비스 시작

```shell
{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd kubelet kube-proxy
  sudo systemctl start containerd kubelet kube-proxy
}
```

> 각 워커 노드 (`worker-0`, `worker-1` 및 `worker-2`) 에서 위의 명령을 실행해야 합니다.

## 확인

컨트롤러 노드 중 하나에 로그인합니다.

```shell
CONTROLLER="controller-0"
PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
  -n ${CONTROLLER}-pip --query "ipAddress" -otsv)

ssh kuberoot@${PUBLIC_IP_ADDRESS}
```

등록된 쿠버네티스 노드를 확인합니다.

```shell
kubectl get nodes
```

> 출력

```shell
NAME       STATUS    AGE       VERSION
worker-0   Ready    <none>   32s   v1.15.0
worker-1   Ready    <none>   32s   v1.15.0
worker-2   Ready    <none>   37s   v1.15.0
```

다음: [원격 액세스를위한 kubectl 구성](10-configuring-kubectl.md)

# 쿠버네티스 컨트롤 플레인 부트스트랩

이 실습에서는 3개의 컴퓨팅 인스턴스에서 쿠버네티스 컨트롤 플레인을 부트스트랩하고 고 가용성을 위해 구성합니다. 또한 쿠버네티스 API 서버를 원격 클라이언트에 노출시키는 외부 로드 밸런서를 생성합니다. 쿠버네티스 API 서버, 스케줄러 및 컨트롤러 관리자와 같은 구성 요소가 각 노드에 설치됩니다.

## 전제 조건

각 컨트롤러 인스턴스 (`controller-0`, `controller-1` 및 `controller-2`)에 로그인해서 이 실습에 포함된 내용을 실행해야합니다. 각 인스턴스에 로그인하기 위해서는 `az` 명령을 사용하여 각 컨트롤러 인스턴스의 퍼블릭 IP를 찾아야 하며, 아래와 같이 실행하여 찾을 수 있습니다.

```shell
CONTROLLER="controller-0"
PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
  -n ${CONTROLLER}-pip --query "ipAddress" -otsv)

ssh kuberoot@${PUBLIC_IP_ADDRESS}
```

### tmux로 동시에 여러 명령 실행하기

[tmux](https://github.com/tmux/tmux/wiki)를 사용하여 여러 컴퓨팅 인스턴스에서 동시에 명령을 실행할 수 있습니다. 앞 단계의 [tmux로 동시에 여러 명령 실행하기](01-prerequisites.md#running-commands-in-parallel-with-tmux) 섹션에서 자세한 내용을 확인할 수 있습니다.

## 쿠버네티스 컨트롤 플레인 프로비저닝

쿠버네티스 바이너리 파일들을 설치할 디렉터리를 만듭니다.

```shell
sudo mkdir -p /etc/kubernetes/config
```

### 쿠버네티스 컨트롤러 바이너리 다운로드 및 설치

공식 쿠버네티스 릴리스 바이너리를 다운로드합니다.

```shell
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kubectl"
```

쿠버네티스 바이너리를 설치합니다.

```shell
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

### 쿠버네티스 API 서버 구성

```shell
{
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
}
```

인스턴스 내부 IP 주소가 사용되어 API 서버를 클러스터 구성원에게 알립니다. 지금 작업 중인 컴퓨팅 인스턴스의 내부 IP 주소를 확인하여 변수로 저장합니다.

```shell
INTERNAL_IP=$(ip addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
```

`kube-apiserver.service` 시스템 유닛 파일을 만듭니다.

```shell
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,TaintNodesByCondition,Priority,DefaultTolerationSeconds,DefaultStorageClass,PersistentVolumeClaimResize,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 쿠버네티스 컨트롤러 매니저 구성

`kube-controller-manager` kubeconfig 파일을 다음 위치로 옮깁니다.

```shell
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

`kube-controller-manager.service` 시스템 유닛 파일을 만듭니다.

```shell
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 쿠버네티스 스케줄러 구성

`kube-scheduler` kubeconfig 파일을 다음 위치로 옮깁니다.

```shell
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

`kube-scheduler.yaml` 구성 파일을 만듭니다.

```shell
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

`kube-scheduler.service` 시스템 유닛 파일을 만듭니다.

```shell
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 컨트롤러 서비스 시작

```shell
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

> 쿠버네티스 API 서버가 완전히 초기화 될 때까지 최대 10초 정도 소요됩니다.

### 확인

```shell
kubectl get componentstatuses
```

```shell
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-2               Healthy   {"health": "true"}
```

> 각 컨트롤러 노드 (`controller-0`, `controller-1` 및 `controller-2`) 에서 명령을 실행하는 것을 놓치지 마세요.

## Kubelet 인증을 위한 RBAC

이 섹션에서는 쿠버네티스 API 서버가 각 작업자 노드의 Kubelet API에 액세스 할 수 있도록 RBAC 권한을 구성합니다. 파드에서 메트릭, 로그 및 명령을 검색하려면 Kubelet API에 액세스해야합니다.

> 이 튜토리얼은 Kubelet `--authorization-mode` 플래그를 `Webhook`으로 설정합니다. 웹 후크 모드는 [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API를 사용하여 권한을 결정합니다.

```shell
CONTROLLER="controller-0"
PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
  -n ${CONTROLLER}-pip --query "ipAddress" -otsv)

ssh kuberoot@${PUBLIC_IP_ADDRESS}
```

Kubelet API에 액세스 할 수있는 권한으로 `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole)을 만들고 파드 관리와 관련된 가장 일반적인 작업을 수행합니다.

```shell
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

쿠버네티스 API 서버는 Kubelet에 `kubernetes` 사용자로서 `--kubelet-client-certificate` 플래그로 정의된 클라이언트 인증서를 사용하여 인증합니다.

`system:kube-apiserver-to-kubelet` ClusterRole을 `kubernetes` 사용자에게 바인딩합니다:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## 쿠버네티스 프론트 엔드 로드 밸런서

이 섹션에서는 쿠버네티스 API 서버를 위해 외부 로드 밸런서를 프로비저닝합니다. `kubernetes-the-hard-way` 고정 IP 주소는 새로 만들어지는 로드 밸런서와 연결합니다.

> 이 자습서를 진행하면서 새로 만든 컴퓨팅 인스턴스에서는 이 섹션을 완료하기 위해 필요한 권한이 없습니다. 컴퓨팅 인스턴스를 작성하는 데 사용된 여러분의 컴퓨터에서 계속 진행해야 합니다.

로드 밸런서를 만들기 위하여 필요한 로드 밸런서 상태 검사기를 만듭니다.

```shell
az network lb probe create -g kubernetes \
  --lb-name kubernetes-lb \
  --name kubernetes-apiserver-probe \
  --port 6443 \
  --protocol tcp
```

외부 로드 밸런서 네트워크 리소스를 만듭니다.

```shell
az network lb rule create -g kubernetes \
  -n kubernetes-apiserver-rule \
  --protocol tcp \
  --lb-name kubernetes-lb \
  --frontend-ip-name LoadBalancerFrontEnd \
  --frontend-port 6443 \
  --backend-pool-name kubernetes-lb-pool \
  --backend-port 6443 \
  --probe-name kubernetes-apiserver-probe
```

### 확인

`kubernetes-the-hard-way` 고정 IP 주소를 확인합니다.

```shell
KUBERNETES_PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
  -n kubernetes-pip --query ipAddress -otsv)
```

Kubernetes 버전 정보를 확인하는 HTTP 요청을 보냅니다.

```shell
curl --cacert ca.pem https://$KUBERNETES_PUBLIC_IP_ADDRESS:6443/version
```

> 출력

```shell
{
  "major": "1",
  "minor": "14",
  "gitVersion": "v1.15.0",
  "gitCommit": "5e53fd6bc17c0dec8434817e69b04a25d8ae0ff0",
  "gitTreeState": "clean",
  "buildDate": "2019-06-06T01:36:19Z",
  "goVersion": "go1.12.5",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

다음: [쿠버네티스 워커 노드 부트스트랩](09-bootstrapping-kubernetes-workers.md)

# etcd 클러스터 부트스트랩

Kubernetes 구성 요소는 상태 독립적이며 클러스터 상태를 [etcd](https://github.com/etcd-io/etcd)에 저장합니다. 이 실습에서는 3개의 작업자 노드로 구성된 클러스터를 부트스트랩하고 고 가용성 및 안전한 원격 액세스를 위해 구성합니다.

## 전제 조건

각 컨트롤러 인스턴스 (`controller-0`, `controller-1` 및 `controller-2`)에 로그인해서 이 실습에 포함된 내용을 실행해야합니다. 각 인스턴스에 로그인하기 위해서는 `az` 명령을 사용하여 각 컨트롤러 인스턴스의 퍼블릭 IP를 찾아야 하며, 아래와 같이 실행하여 찾을 수 있습니다.

```shell
CONTROLLER="controller-0"
PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
  -n ${CONTROLLER}-pip --query "ipAddress" -otsv)

ssh kuberoot@${PUBLIC_IP_ADDRESS}
```

## etcd 클러스터 멤버 부트스트랩

### etcd 바이너리 다운로드 및 설치

[etcd-io/etcd](https://github.com/etcd-io/etcd) GitHub 프로젝트에서 공식 etcd 릴리스 바이너리를 다운로드하십시오.

```shell
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.3.13/etcd-v3.3.13-linux-amd64.tar.gz"
```

`etcd` 서버와 `etcdctl` CLI 유틸리티를 추출하여 설치하십시오.

```shell
{
  tar -xvf etcd-v3.3.13-linux-amd64.tar.gz
  sudo mv etcd-v3.3.13-linux-amd64/etcd* /usr/local/bin/
}
```

### etcd 서버 구성

```shell
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
}
```

인스턴스 내부 IP 주소는 클라이언트 요청을 처리하고 etcd 클러스터 피어와 통신하는 데 사용됩니다. 현재 인스턴스의 내부 IP 주소를 확인합니다.

```shell
INTERNAL_IP=$(ip addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
```

각 etcd 멤버는 etcd 클러스터 내에서 고유한 이름을 가져야합니다. etcd 이름을 현재 계산 인스턴스의 호스트 이름과 일치하도록 설정합니다.

```shell
ETCD_NAME=$(hostname -s)
```

`etcd.service` 시스템 유닛 파일을 작성합니다.

```shell
cat > etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### etcd 서버 시작하기

```shell
sudo mv etcd.service /etc/systemd/system/
```

```shell
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

> 각 컨트롤러 노드에서 `controller-0`, `controller-1` 및 `controller-2` 명령을 실행해야 한다는 것을 놓치지 마세요.

## 확인

etcd 클러스터 멤버를 확인합니다.

```shell
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://${INTERNAL_IP}:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> 출력

```shell
3a57933972cb5131, started, controller-2, https://10.240.0.12:2380, https://10.240.0.12:2379
f98dc20bce6225a0, started, controller-0, https://10.240.0.10:2380, https://10.240.0.10:2379
ffed16798470cab5, started, controller-1, https://10.240.0.11:2380, https://10.240.0.11:2379
```

다음: [쿠버네티스 컨트롤 플레인 부트스트랩](08-bootstrapping-kubernetes-controllers.md)

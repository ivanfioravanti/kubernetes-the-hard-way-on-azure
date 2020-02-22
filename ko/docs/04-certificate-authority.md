# CA 프로비저닝 및 TLS 인증서 생성

이 실습에서는 CloudFlare의 PKI 도구인 [cfssl](https://en.wikipedia.org/wiki/Public_key_infrastructure)을 사용하여 [PKI 인프라](https://github.com/cloudflare/cfssl)를 프로비저닝 한 다음, 이를 사용하여 인증 기관을 부트 스트랩하고 etcd, kube-apiserver, kubelet 및 kube-proxy와 같은 구성 요소에 대한 TLS 인증서를 생성합니다.

## 인증 주체 (CA)

이 섹션에서는 추가 TLS 인증서를 생성하는 데 사용할 수 있는 인증 주체를 새로 만듭니다.

CA 구성 파일을 만듭니다.

```shell
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

CA 인증서 서명 요청을 만듭니다.

```shell
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IT",
      "L": "Milan",
      "O": "Kubernetes",
      "OU": "MI",
      "ST": "Italy"
    }
  ]
}
EOF
```

CA 인증서 및 개인 키를 만듭니다.

```shell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

다음과 같이 파일이 만들어져야 합니다.

```shell
ca-key.pem
ca.pem
```

## 클라이언트 및 서버 인증서

이 섹션에서는 각 쿠버네티스 구성 요소에 대한 클라이언트 및 서버 인증서와 쿠버네티스 `admin` 사용자에 대한 클라이언트 인증서를 생성합니다.

### 관리 클라이언트 인증서

`admin` 클라이언트 인증서 서명 요청을 작성합니다.

```shell
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IT",
      "L": "Milan",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Italy"
    }
  ]
}
EOF
```

`admin` 클라이언트 인증서 및 개인 키를 생성합니다.

```shell
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

다음과 같이 파일이 만들어져야 합니다.

```shell
admin-key.pem
admin.pem
```

### Kubelet 클라이언트 인증서

쿠버네티스는 노드 인증자라는 [특수 목적의 권한 부여 모드를](https://kubernetes.io/docs/admin/authorization/node/) 사용합니다. 이 [모드](https://kubernetes.io/docs/concepts/overview/components/#kubelet)는 {a2}Kubelets의{/a2} API 요청을 구체적으로 승인합니다. 노드 인증자를 통해 권한을 부여하기 위해 새로 등록하는 Kubelet은 `system:node:<nodeName>` 이라는 사용자 이름이 {code4}system:nodes{/code4} 그룹에 속하는 것으로 식별되는 자격 증명을 사용해야합니다. 이 섹션에서는 노드 권한 부여자 요구 사항을 충족하는 각 쿠버네티스 작업자 노드에 대한 인증서를 만듭니다.

각 쿠버네티스 작업자 노드에 대한 인증서 및 개인 키를 생성합니다.

```shell
for instance in worker-0 worker-1 worker-2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IT",
      "L": "Milan",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Italy"
    }
  ]
}
EOF

EXTERNAL_IP=$(az network public-ip show -g kubernetes \
  -n kubernetes-pip --query ipAddress -otsv)

INTERNAL_IP=$(az vm show -d -n ${instance} -g kubernetes --query privateIps -otsv)

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

다음과 같이 파일이 만들어져야 합니다.

```shell
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

### 컨트롤러 관리자 클라이언트 인증서

`kube-controller-manager` 클라이언트 인증서 및 개인 키를 생성합니다.

```shell
{

cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IT",
      "L": "Milan",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Italy"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

다음과 같이 파일이 만들어져야 합니다.

```shell
kube-controller-manager-key.pem
kube-controller-manager.pem
```

### Kube Proxy 클라이언트 인증서

`kube-proxy` 클라이언트 인증서 서명 요청을 작성합니다.

```shell
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IT",
      "L": "Milano",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Italy"
    }
  ]
}
EOF
```

`kube-proxy` 클라이언트 인증서 및 개인 키를 생성합니다.

```shell
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

다음과 같이 파일이 만들어져야 합니다.

```shell
kube-proxy-key.pem
kube-proxy.pem
```

### 스케줄러 클라이언트 인증서

`kube-scheduler` 클라이언트 인증서 및 개인 키를 생성합니다.

```shell
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IT",
      "L": "Milan",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Italy"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```

다음과 같이 파일이 만들어져야 합니다.

```shell
kube-scheduler-key.pem
kube-scheduler.pem
```

### 쿠버네티스 API 서버 인증서

`kubernetes-the-hard-way` 고정 IP 주소는 쿠버네티스 API 서버 인증서의 주체 대체 이름 목록에 포함됩니다. 이렇게하면 원격 클라이언트가 인증서를 확인할 수 있습니다.

`kubernetes-the-hard-way` 고정 IP 주소를 확인합니다.

```shell
KUBERNETES_PUBLIC_ADDRESS=$(az network public-ip show -g kubernetes \
  -n kubernetes-pip --query "ipAddress" -otsv)
```

쿠버네티스 API 서버 인증서 서명 요청을 작성합니다.

```shell
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IT",
      "L": "Milam",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Italy"
    }
  ]
}
EOF
```

쿠버네티스 API 서버 인증서 및 개인 키를 생성하십시오.

```shell
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

다음과 같이 파일이 만들어져야 합니다.

```shell
kubernetes-key.pem
kubernetes.pem
```

## 서비스 계정 키 페어

쿠버네티스 컨트롤러 매니저는 서비스 계정 [관리](https://kubernetes.io/docs/admin/service-accounts-admin/) 문서에 설명 된대로 키 페어를 사용하여 서비스 계정 토큰을 생성하고 서명합니다.

`service-account` 인증서 및 개인 키를 만듭니다.

```shell
{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IT",
      "L": "Milan",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Italy"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

다음과 같이 파일이 만들어져야 합니다.

```shell
service-account-key.pem
service-account.pem
```

## 클라이언트 및 서버 인증서 배포

## 이전 단계를 따르는 경우 리눅스 VM을 만드는 데 사용 된 사용자 이름은 kuberoot입니다.

적절한 인증서와 개인 키를 각 작업자 인스턴스에 복사합니다.

```shell
for instance in worker-0 worker-1 worker-2; do
  PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
    -n ${instance}-pip --query "ipAddress" -otsv)

  scp -o StrictHostKeyChecking=no ca.pem ${instance}-key.pem ${instance}.pem kuberoot@${PUBLIC_IP_ADDRESS}:~/
done
```

적절한 인증서와 개인 키를 각 컨트롤러 인스턴스에 복사합니다.

```shell
for instance in controller-0 controller-1 controller-2; do
  PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
    -n ${instance}-pip --query "ipAddress" -otsv)

  scp -o StrictHostKeyChecking=no ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem kuberoot@${PUBLIC_IP_ADDRESS}:~/
done
```

> `kube-proxy`, `kube-controller-manager`, `kube-scheduler` 및 `kubelet` 클라이언트 인증서는 다음 실습에서 클라이언트 인증 구성 파일을 생성하는 데 사용됩니다.

다음: [인증을 위한 쿠버네티스 구성 파일 생성](05-kubernetes-configuration-files.md)

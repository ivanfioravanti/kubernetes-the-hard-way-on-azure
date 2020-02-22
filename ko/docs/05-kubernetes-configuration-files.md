# 인증을 위한 쿠버네티스 구성 파일 생성

이 실습에서는 kubeconfigs 라고 불리는 [쿠버네티스 구성 파일](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) 을 생성하여 쿠버네티스 클라이언트가 쿠버네티스 API 서버를 찾고 인증 할 수 있도록 합니다.

## 클라이언트 인증 구성

이 섹션에서는 `controller manager`, `kubelet`, `kube-proxy` 및 `scheduler` 클라이언트와 `admin` 사용자에 대한 kubeconfig 파일을 생성합니다.

### 쿠버네티스 퍼블릭 IP 주소

각 kubeconfig에는 쿠버네티스 API 서버가 지정되어 있어야합니다. 고 가용성을 지원하기 위해 쿠버네티스 API 서버 앞에 있는 외부로드 밸런서에 할당된 IP 주소가 사용됩니다.

`kubernetes-the-hard-way` 고정 IP 주소를 확인합니다.

```shell
KUBERNETES_PUBLIC_ADDRESS=$(az network public-ip show -g kubernetes \
  -n kubernetes-pip --query "ipAddress" -otsv)
```

### kubelet 쿠버네티스 구성 파일

Kubelets에 대한 kubeconfig 파일을 생성 할 때 Kubelet의 노드 이름과 일치하는 클라이언트 인증서를 사용해야합니다. 이를 통해 Kubelet이 쿠버네티스 [노드 인증자](https://kubernetes.io/docs/admin/authorization/node/)에 의해 올바르게 승인됩니다.

각 작업자 노드에 대한 kubeconfig 파일을 만듭니다.

```shell
for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

다음과 같이 파일이 만들어져야 합니다.

```shell
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```

### kube-proxy 쿠버네티스 구성 파일

`kube-proxy` 서비스에 대한 kubeconfig 파일을 만듭니다.

```shell
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=kube-proxy.kubeconfig
```

```shell
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
```

```shell
kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
```

```shell
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

다음과 같이 파일이 만들어져야 합니다.

```shell
kube-proxy.kubeconfig
```

### kube-controller-manager 쿠버네티스 구성 파일

`kube-controller-manager` 서비스를 위한 kubeconfig 파일을 만듭니다.

```shell
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

다음과 같이 파일이 만들어져야 합니다.

```shell
kube-controller-manager.kubeconfig
```

### kube-scheduler 쿠버네티스 구성 파일

`kube-scheduler` 서비스에 대한 kubeconfig 파일을 만듭니다.

```shell
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

다음과 같이 파일이 만들어져야 합니다.

```shell
kube-scheduler.kubeconfig
```

### 관리자 쿠버네티스 구성 파일

`admin` 사용자를 위한 kubeconfig 파일을 만듭니다.

```shell
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

다음과 같이 파일이 만들어져야 합니다.

```shell
admin.kubeconfig
```

## 쿠버네티스 구성 파일 배포

적절한 `kubelet` 및 `kube-proxy` kubeconfig 파일을 각 작업자 인스턴스에 복사합니다.

```shell
for instance in worker-0 worker-1 worker-2; do
  PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
    -n ${instance}-pip --query "ipAddress" -otsv)

  scp ${instance}.kubeconfig kube-proxy.kubeconfig kuberoot@${PUBLIC_IP_ADDRESS}:~/
done
```

적절한 `kube-controller-manager` 및 `kube-scheduler` kubeconfig 파일을 각 컨트롤러 인스턴스에 복사합니다.

```shell
for instance in controller-0 controller-1 controller-2; do
  PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
    -n ${instance}-pip --query "ipAddress" -otsv)

  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig kuberoot@${PUBLIC_IP_ADDRESS}:~/
done
```

다음: [데이터 암호화 구성 및 키 생성](06-data-encryption-keys.md)

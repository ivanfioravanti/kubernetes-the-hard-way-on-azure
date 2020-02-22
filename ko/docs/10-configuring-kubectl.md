# 원격 액세스를 위한 kubectl 구성

이 실습에서는 `admin` 사용자 자격 증명을 기반으로 `kubectl` CLI 유틸리티에 대한 kubeconfig 파일을 생성합니다.

> 관리 클라이언트 인증서를 생성하는 데 사용된 것과 동일한 디렉토리에서 아래 명령들을 실행해야 합니다.

## 관리자 쿠버네티스 구성 파일

각 kubeconfig에는 쿠버네티스 API 서버가 지정되어야합니다. 고 가용성을 지원하기 위해 쿠버네티스 API 서버 앞에있는 외부 로드 밸런서에 할당 된 IP 주소가 사용됩니다.

`kubernetes-the-hard-way` 고정 IP 주소를 확인합니다.

```shell
KUBERNETES_PUBLIC_ADDRESS=$(az network public-ip show -g kubernetes \
  -n kubernetes-pip --query ipAddress -otsv)
```

`admin`으로 인증하기에 적합한 kubeconfig 파일을 생성합니다.

```shell
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443
```

```shell
kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem
```

```shell
kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin
```

```shell
kubectl config use-context kubernetes-the-hard-way
```

## 확인

원격 쿠버네티스 클러스터 컴포넌트들의 상태를 확인합니다.

```shell
kubectl get componentstatuses
```

> 출력

```shell
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-2               Healthy   {"health": "true"}
```

원격 쿠버네티스 클러스터의 노드 목록을 확인합니다.

```shell
kubectl get nodes
```

> 출력

```shell
NAME       STATUS    AGE       VERSION
worker-0   Ready    <none>   75s   v1.15.0
worker-1   Ready    <none>   73s   v1.15.0
worker-2   Ready    <none>   72s   v1.15.0
```

다음: [파드 네트워크 경로 프로비저닝](11-pod-network-routes.md)

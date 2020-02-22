# DNS 클러스터 애드온 배포

이 실습에서는 [CoreDNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) 에 의해 지원되는 DNS 기반 서비스 검색을 쿠버네티스 클러스터 내에서 실행되는 응용 프로그램에 제공하는 [DNS 애드온](https://coredns.io/) 을 배포합니다.

## DNS 클러스터 애드온

`coredns` 클러스터 애드온을 배포합니다.

```
kubectl apply -f https://raw.githubusercontent.com/ivanfioravanti/kubernetes-the-hard-way-on-azure/master/deployments/coredns.yaml
```

> 출력

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.extensions/coredns created
service/kube-dns created
```

`kube-dns` 배포로 생성된 파드를 나열합니다.

```
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> 출력

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-699f8ddd77-94qv9   1/1     Running   0          20s
coredns-699f8ddd77-gtcgb   1/1     Running   0          20s
```

## 확인

`busybox` 배포를 만듭니다.

```
kubectl run --generator=run-pod/v1 busybox --image=busybox:1.28 --command -- sleep 3600
```

`busybox` 배포에서 생성한 파드를 확인합니다.

```
kubectl get pods -l run=busybox
```

> 출력

```
NAME                      READY   STATUS    RESTARTS   AGE
busybox                   1/1     Running   0          10s
```

`busybox` 파드의 전체 이름을 확인합니다.

```
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

`busybox` 파드 내에서 `kubernetes` 서비스에 대한 DNS 조회를 요청합니다.

```
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> 출력

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

다음: [스모크 테스트](13-smoke-test.md)

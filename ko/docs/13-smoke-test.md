# 스모크 테스트

이 실습에서는 쿠버네티스 클러스터가 올바르게 작동하는지 확인하기위한 일련의 작업을 진행합니다.

## 데이터 암호화

이 섹션에서는 [데이터를 암호화](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted)하는 기능을 확인합니다.

일반적인 시크릿을 만듭니다.

```shell
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

etcd에 저장된 `kubernetes-the-hard-way` 시크릿의 16 진 덤프를 출력해봅니다.

```shell
CONTROLLER="controller-0"
PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
  -n ${CONTROLLER}-pip --query "ipAddress" -otsv)

ssh kuberoot@${PUBLIC_IP_ADDRESS} \
  "ETCDCTL_API=3 etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> 출력

```shell
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 65 c3 db a8 fb ae 9b  |:v1:key1:e......|
00000050  f9 09 59 0b 12 fa 4f 5d  4c 6c c5 35 28 d8 72 08  |..Y...O]Ll.5(.r.|
00000060  f7 9e 4b 0a 6e 1d 6b 27  8f d2 7f 36 2b 11 6b 61  |..K.n.k'...6+.ka|
00000070  53 6a a7 24 56 e2 19 ee  e7 04 94 ee b3 9c d3 c3  |Sj.$V...........|
00000080  68 b5 b8 51 8b 01 4e d9  f0 ce 40 9a 73 5c 10 28  |h..Q..N...@.s\.(|
00000090  18 bc ff 3a 51 4d bc 0c  6d 27 97 5c c6 bd a2 35  |...:QM..m'.\...5|
000000a0  88 18 56 16 c7 10 12 a1  e2 cf c5 62 6c 50 7e 67  |..V........blP~g|
000000b0  89 0c 42 56 73 69 48 bf  24 5e 91 91 56 2d 64 2f  |..BVsiH.$^..V-d/|
000000c0  3a 35 b9 c9 08 41 d6 95  62 e8 1b 35 80 c9 8e 74  |:5...A..b..5...t|
000000d0  79 34 bc 5b 7c 68 cd 0c  bc 11 21 c0 48 bc 92 a6  |y4.[|h....!.H...|
000000e0  2f b5 ef 18 5c f1 00 16  19 22 e8 9c c1 8c 3c 35  |/...\...."....<5|
000000f0  fa b3 87 51 85 bf f0 cd  0e 0a                    |...Q......|
000000fa
```

etcd 키는 반드시 `k8s:enc:aescbc:v1:key1` 접두사를 사용해야 하며, 이것은 `aescbc` 공급자가 `key1` 암호화 키로 데이터를 암호화하기 위하여 사용되었음을 나타냅니다.

## 배포

이 섹션에서는 [배포](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)를 만들고 관리하는 기능을 확인합니다.

[nginx](https://nginx.org/en/) 웹 서버에 대한 배포를 만듭니다.

```shell
kubectl run --generator=run-pod/v1 nginx --image=nginx
```

`nginx` 배포로 생성 된 파드를 확인합니다.

```shell
kubectl get pods -l run=nginx
```

> 출력

```shell
NAME                     READY     STATUS    RESTARTS   AGE
nginx                    1/1       Running   0          15s
```

### 포트 포워딩

이 섹션에서는 [포트 포워딩](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)을 사용하여 원격으로 응용 프로그램에 액세스하는 기능을 확인합니다.

`nginx` 파드의 전체 이름을 확인합니다.

```shell
POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")
```

로컬 시스템의 포트 `8080`을 `nginx` 파드의 포트 `80`으로 연결합니다.

```shell
kubectl port-forward $POD_NAME 8080:80
```

> 출력

```shell
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

새 터미널에서 포워딩된 주소를 사용하여 HTTP 요청을 보내어 연결 상태를 확인합니다.

```shell
curl --head http://127.0.0.1:8080
```

> 출력

```shell
HTTP/1.1 200 OK
Server: nginx/1.17.0
Date: Sun, 16 Jun 2019 08:51:13 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 21 May 2019 14:23:57 GMT
Connection: keep-alive
ETag: "5ce409fd-264"
Accept-Ranges: bytes
```

이전 터미널로 다시 전환하고 `nginx` 파드와의 포트 포워딩을 종료합니다.

```shell
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### 로그

이 섹션에서는 [컨테이너 로그](https://kubernetes.io/docs/concepts/cluster-administration/logging/)를 출력하는 기능을 확인합니다.

`nginx` 파드 로그를 확인합니다.

```shell
kubectl logs $POD_NAME
```

> 출력

```shell
127.0.0.1 - - [16/Jun/2019:08:51:13 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.54.0" "-"
```

### 실행

이 섹션에서는 [컨테이너에서 명령을 실행](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container) 하는 기능을 확인합니다.

`nginx` 컨테이너에서 `nginx -v` 명령을 실행하여 nginx 버전을 확인합니다.

```shell
kubectl exec -ti $POD_NAME -- nginx -v
```

> 출력

```shell
nginx version: nginx/1.17.0
```

## 서비스

이 섹션에서는 [서비스](https://kubernetes.io/docs/concepts/services-networking/service/)를 사용하여 응용 프로그램을 노출하는 기능을 확인합니다.

{a0}NodePort{/a0} 서비스를 사용하여 {code1}nginx{/code1} 배포를 외부에서 접속할 수 있게 공개합니다.

```shell
kubectl expose pod nginx --port 80 --type NodePort
```

> 이 자습서에서는 클러스터가 [클라우드 제공자 통합으로](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider) 구성되지 않았으므로 LoadBalancer 서비스 유형을 사용할 수 없습니다. 클라우드 제공자 통합 설정은 이 학습서의 범위를 벗어나므로 다루지 않습니다.

`nginx` 서비스에 지정된 노드 포트를 확인합니다.

```shell
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

`nginx` 노드 포트에 대한 원격 액세스를 허용하는 방화벽 규칙 만듭니다.

```shell
az network nsg rule create -g kubernetes \
  -n kubernetes-allow-nginx \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range ${NODE_PORT} \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1002
```

워커 인스턴스의 외부 IP 주소를 확인합니다.

```shell
EXTERNAL_IP=$(az network public-ip show -g kubernetes \
  -n worker-0-pip --query "ipAddress" -otsv)
```

외부 IP 주소와 `nginx` 노드 포트를 사용하여 HTTP 요청을 보냅니다.

```shell
curl -I http://$EXTERNAL_IP:$NODE_PORT
```

> 출력

```shell
HTTP/1.1 200 OK
Server: nginx/1.17.0
Date: Sun, 16 Jun 2019 08:53:18 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 21 May 2019 14:23:57 GMT
Connection: keep-alive
ETag: "5ce409fd-264"
Accept-Ranges: bytes
```

다음: [정리하기](14-cleanup.md)

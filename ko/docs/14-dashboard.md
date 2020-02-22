# 대시보드 구성

이 실습에서는 쿠버네티스 대시보드를 설치합니다.

## 설치

다음 명령을 실행하여 대시 보드를 배포합니다.

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

대시보드 접속을 위하여 프록시 연결을 만듭니다.

```shell
kubectl proxy
```

그런 다음 브라우저에서 대시 보드에 액세스하려면 컴퓨터 브라우저에서 다음 주소로 이동하십시오. [대시 보드 로컬 URL](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/)

> 출력
> 
> 접속하면 kubeconfig 파일이나 토큰을 묻는 로그인 페이지가 나타납니다.

로그인하려면 서비스 계정 및 해당 자격 증명이 필요합니다.

## 서비스 계정 생성

다음 명령을 사용하여 서비스 계정을 만듭니다.

```shell
kubectl create serviceaccount dashboard -n default
```

다음 명령으로 클러스터 바인딩 규칙을 추가합니다.

```shell
kubectl create clusterrolebinding dashboard-admin -n default --clusterrole=cluster-admin --serviceaccount=default:dashboard
```

아래 명령을 사용하여 대시보드 로그인에 필요한 비밀 토큰을 복사합니다.

```shell
kubectl get secret $(kubectl get serviceaccount dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
```

마지막으로 토큰 옵션 (두 번째 라디오 박스)을 선택하여 비밀 토큰을 복사하여 대시 보드 로그인 인터페이스에 붙여 넣기만 하면 됩니다. 로그인하면 쿠버네티스 대시 보드 홈페이지로 리디렉션됩니다.

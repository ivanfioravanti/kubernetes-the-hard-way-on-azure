# 클라이언트 도구 설치

이 모듈에서는 이 자습서를 완료하는 데 필요한 CLI 유틸리티 인 [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl) 및 [kubectl을 설치](https://kubernetes.io/docs/tasks/tools/install-kubectl) 합니다.

## CFSSL 설치

`cfssl` 및 `cfssljson` CLI 유틸리티는 [PKI 인프라](https://en.wikipedia.org/wiki/Public_key_infrastructure)를 프로비저닝하고, TLS 인증서를 생성하는 데 사용됩니다.

`cfssl`과 `cfssljson`을 [cfssl 패키지 저장소](https://pkg.cfssl.org/)에서 다운로드하여 설치합니다.

### OS X

OS X 사용자는 사전 빌드 된 바이너리를 사용하는 데 문제가 있을 수 있습니다. 번거로움을 피하기 위해 [Homebrew](https://brew.sh)를 사용하는 것을 권장합니다.

```
brew install cfssl
```

### 리눅스

```shell
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
```

```shell
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
```

```shell
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
```

```shell
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

### Windows

적절한 버전의 cfss_windows-386.exe 또는 cfssl_windows-amd64.exe 파일을 다운로드하세요. 관리자 권한을 획득하여 PowerShell을 실행한 후 32비트 버전의 경우 아래 명령어를 실행하세요.

```shell
PS C:\Windows\system32>Invoke-WebRequest -Uri https://pkg.cfssl.org/R1.2/cfssl_windows-386.exe -OutFile cfssl.exe
PS C:\Windows\system32>Invoke-WebRequest -Uri https://pkg.cfssl.org/R1.2/cfssljson_windows-386.exe -OutFile cfssljson.exe
```

관리자 권한을 획득하여 PowerShell을 실행한 후, 64비트 시스템인 경우 아래 명령어를 실행하세요.

```shell
PS C:\Windows\system32>Invoke-WebRequest -Uri https://pkg.cfssl.org/R1.2/cfssl_windows-amd64.exe -OutFile cfssl.exe
PS C:\Windows\system32>Invoke-WebRequest -Uri https://pkg.cfssl.org/R1.2/cfssljson_windows-amd64.exe -OutFile cfssljson.exe
```

### 확인

`cfssl` 버전 1.2.0 이상이 설치되었는지 확인합니다.

```shell
cfssl version
```

이 단계가 런타임 오류로 실패하면 [CloudFlare의 리포지토리](https://github.com/cloudflare/cfssl#installation)에 있는 지침에 따라 cfssl을 설치하세요.

> 출력

```shell
Version: 1.3.3
Revision: dev
Runtime: go1.12.3
```

```shell
cfssljson -version
```

> 출력

```shell
Version: 1.3.3
Revision: dev
Runtime: go1.12.3
```

## kubectl 설치

`kubectl` CLI 유틸리티는 쿠버네티스 API 서버와 상호 작용하는 데 사용됩니다. 공식 릴리스 바이너리에서 `kubectl` 을 다운로드하여 설치합니다.

### OS X

```shell
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/darwin/amd64/kubectl
```

```shell
chmod +x kubectl
```

```shell
sudo mv kubectl /usr/local/bin/
```

### 리눅스

```shell
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubectl
```

```shell
chmod +x kubectl
```

```shell
sudo mv kubectl /usr/local/bin/
```

### Windows

먼저 초콜릿 패키지 관리자를 설치해야합니다 (https://chocolatey.org/)

```shell
PS C:\Windows\system32>choco install kubernetes-cli
```

### 확인

`kubectl` 버전 1.13.0 이상이 설치되었는지 확인합니다.

```shell
kubectl version --client
```

> 출력

```shell
Client Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.0", GitCommit:"ddf47ac13c1a9483ea035a79cd7c10005ff21a6d", GitTreeState:"clean", BuildDate:"2018-12-03T21:04:45Z", GoVersion:"go1.11.2", Compiler:"gc", Platform:"darwin/amd64"}
```

kubectl 버전을 빠르게 확인하려면 다음 명령을 사용할 수도 있습니다.

```shell
kubectl version --short
```

> 출력

```shell
Client Version: v1.15.0
```

다음: [컴퓨팅 리소스 프로비저닝](03-compute-resources.md)

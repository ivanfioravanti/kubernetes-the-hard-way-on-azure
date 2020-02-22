# 전제 조건

## Microsoft Azure

이 자습서는 [Microsoft Azure](https://azure.microsoft.com)를 활용하여 쿠버네티스 클러스터를 처음부터 부트 스트랩하는 데 필요한 컴퓨팅 인프라 프로비저닝을 간소화합니다. 신규 가입하면 무료로 $200 어치의 크레딧을 [받으실 수 있습니다](https://azure.microsoft.com/en-us/free/). 다만 Azure 무료 평가판에는 사용 가능한 코어 수가 4개로 제한되어 있으므로 자습서 지침을 변경하여 6개 대신 4개의 노드 (2개의 컨트롤러 및 2개의 작업자)를 만드는 것으로 대체해야 합니다.

이 자습서를 실행하는 데 [소요되는 예상 비용](https://azure.microsoft.com/en-us/pricing/calculator/)은 시간당 약 500원 (하루 약 12,000원) 정도입니다.

> 이 자습서에 필요한 컴퓨팅 리소스는 Microsoft Azure 프리 티어를 초과하지 않습니다.

## Microsoft Azure 클라우드 플랫폼 SDK

### Microsoft Azure CLI 2.0 설치

Microsoft Azure CLI 2.0 [설명서](https://github.com/azure/azure-cli#installation)에 따라 `az` CLI 유틸리티를 설치하고 구성합니다.

Microsoft Azure CLI 2.0 버전이 2.0.66 이상인지 확인합니다.

```shell
az --version
```

### 특정 리전에 기본 리소스 그룹 생성

이 가이드는 이미 여러분이 [Azure CLI 2.0](https://github.com/azure/azure-cli#installation)을 설치했다고 가정하며, `koreacentral` 지역에 `kubernetes` 라는 이름의 리소스 그룹 아래에 새 리소스들을 만들 것입니다. 새로운 리소스 그룹을 만들려면, 다음과 같이 명령을 실행하면 됩니다:

```shell
az group create -n kubernetes -l koreacentral
```

> 사용 가능한 다른 지역 코드를 보려면 `az account list-locations` 명령을 사용하세요.

## tmux로 동시에 여러 명령 실행하기

[tmux](https://github.com/tmux/tmux/wiki) 를 사용하여 여러 컴퓨팅 인스턴스에서 동시에 명령을 실행할 수 있습니다. 이 자습서의 실습에서는 여러 컴퓨팅 인스턴스에서 동일한 명령을 여러번 실행해야 할 수 있습니다. 이러한 경우 tmux를 사용하고 프로비저닝 프로세스 속도를 높이기 위해 `synchronize-panes` 사용하여 창을 여러 창으로 분할하는 것을 추천합니다.

> tmux를 사용할 것인지의 여부는 선택 사항이며, 이 학습을 완료하는 데 꼭 필요하지는 않습니다.

![tmux screenshot](../../docs/images/tmux-screenshot.png)

> `synchronize-panes` 활성화하기: 
> `ctrl+b` 키 그리고 `shift :` 키를 누릅니다. 그런 다음 프롬프트에서 `set synchronize-panes on` 명령어를 입력하세요. 동기화를 비활성화하려면: `set synchronize-panes off` 명령어를 입력하세요.

다음: [클라이언트 도구 설치](02-client-tools.md)

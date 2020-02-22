# 컴퓨팅 리소스 프로비저닝

쿠버네티스는 쿠버네티스 컨트롤 플레인과 컨테이너가 궁극적으로 실행되는 작업자 노드를 호스팅하기 위해 일련의 컴퓨터가 필요합니다. 이 실습에서는 단일 [리전](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-overview#resource-groups) 의 단일 [리소스 그룹](https://azure.microsoft.com/en-us/regions/) 내에서 안전하고 가용성이 높은 쿠버네티스 클러스터를 실행하는 데 필요한 컴퓨팅 리소스를 프로비저닝합니다.

> [전제 조건](01-prerequisites.md#create-a-deafult-resource-group-in-a-region) 모듈에서 설명한대로 리소스 그룹이 준비되었는지 확인합니다.

## 네트워킹

쿠버네티스 [네트워킹 모델](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) 은 컨테이너와 노드가 서로 통신 할 수있는 플랫 네트워크를 가정합니다. 만약 이 모델을 사용할 수 없는 경우 [네트워크 정책](https://kubernetes.io/docs/concepts/services-networking/network-policies/)은 컨테이너 그룹이 서로 및 외부 네트워크 엔드 포인트와 통신하는 방법을 제한할 수 있습니다.

> 네트워크 정책 설정은 이 자습서에서 다루지 않습니다.

### 가상 네트워크

이 섹션에서는 Kubernetes 클러스터를 호스팅하기 위해 전용 [가상 네트워크](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview) (VNet)를 설정합니다.

쿠버네티스 클러스터의 각 노드에 개인 IP 주소를 할당하기에 충분히 큰 IP 주소 범위로 프로비저닝 된 서브넷 `kubernetes` 사용하여 `kubernetes-vnet` 사용자 정의 가상 네트워크를 만듭니다.

```shell
az network vnet create -g kubernetes \
  -n kubernetes-vnet \
  --address-prefix 10.240.0.0/24 \
  --subnet-name kubernetes-subnet
```

> `10.240.0.0/24` IP 주소 범위는 최대 254 개의 컴퓨팅 인스턴스를 호스팅 할 수 있습니다.

### 방화벽 규칙

방화벽 ([네트워크 보안 그룹, NSG](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-nsg))을 만들어 서브넷에 지정합니다.

```shell
az network nsg create -g kubernetes -n kubernetes-nsg
```

```shell
az network vnet subnet update -g kubernetes \
  -n kubernetes-subnet \
  --vnet-name kubernetes-vnet \
  --network-security-group kubernetes-nsg
```

외부 SSH 및 HTTPS를 허용하는 방화벽 규칙을 추가합니다.

```shell
az network nsg rule create -g kubernetes \
  -n kubernetes-allow-ssh \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range 22 \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1000
```

```shell
az network nsg rule create -g kubernetes \
  -n kubernetes-allow-api-server \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range 6443 \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1001
```

> 쿠버네티스 API 서버를 원격 클라이언트에 노출시키기 위하여 [외부 로드 밸런서](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview)가 사용됩니다.

`kubernetes-vnet` 가상 네트워크의 방화벽 규칙을 확인해봅니다.

```shell
az network nsg rule list -g kubernetes --nsg-name kubernetes-nsg --query "[].{Name:name, \
  Direction:direction, Priority:priority, Port:destinationPortRange}" -o table
```

> 출력

```shell
Name                         Direction      Priority    Port
---------------------------  -----------  ----------  ------
kubernetes-allow-ssh         Inbound            1000      22
kubernetes-allow-api-server  Inbound            1001    6443
```

### 쿠버네티스 퍼블릭 IP 주소

쿠버네티스 API 서버를 향한 외부로드 밸런서에 연결될 고정 IP 주소를 할당합니다.

```shell
az network lb create -g kubernetes \
  -n kubernetes-lb \
  --backend-pool-name kubernetes-lb-pool \
  --public-ip-address kubernetes-pip \
  --public-ip-address-allocation static
```

`kubernetes-pip` 고정 IP 주소가 `kubernetes` 리소스 그룹 및 선택한 리전에 정확하게 만들어졌는지 확인합니다.

```shell
az network public-ip  list --query="[?name=='kubernetes-pip'].{ResourceGroup:resourceGroup, \
  Region:location,Allocation:publicIpAllocationMethod,IP:ipAddress}" -o table
```

> 출력

```shell
ResourceGroup    Region    Allocation    IP
---------------  --------  ------------  --------------
kubernetes       eastus2   Static        XX.XXX.XXX.XXX
```

## 가상 머신

이 실습의 컴퓨팅 인스턴스는 [우분투 서버](https://www.ubuntu.com/server) 18.04를 사용하여 프로비저닝되며, [cri-containerd](https://github.com/kubernetes-incubator/cri-containerd) 런타임을 잘 지원합니다. 각 컴퓨팅 인스턴스에는 쿠버네티스 부트 스트랩 과정을 단순화하기 위해 고정 프라이빗 IP 주소가 제공됩니다.

안정적인 최신 Ubuntu Server 릴리스를 선택하려면, 아래의 UBUNTULTS 변수의 내용을 아래 명령을 실행하여 나오는 결과 표 상의 가장 최신 버전으로 설정합니다.

```shell
az vm image list --location eastus2 --publisher Canonical --offer UbuntuServer --sku 18.04-LTS --all -o table
```

```shell
UBUNTULTS="Canonical:UbuntuServer:18.04-LTS:18.04.201906170"
```

### 쿠버네티스 컨트롤러

`controller-as` [가용성 집합](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/regions-and-availability#availability-sets) 안에 3개의 컴퓨터 인스턴스를 만들어서 컨트롤러 노드를 호스팅하도록 합니다.

```shell
az vm availability-set create -g kubernetes -n controller-as
```

```shell
for i in 0 1 2; do
    echo "[Controller ${i}] Creating public IP..."
    az network public-ip create -n controller-${i}-pip -g kubernetes > /dev/null

    echo "[Controller ${i}] Creating NIC..."
    az network nic create -g kubernetes \
        -n controller-${i}-nic \
        --private-ip-address 10.240.0.1${i} \
        --public-ip-address controller-${i}-pip \
        --vnet kubernetes-vnet \
        --subnet kubernetes-subnet \
        --ip-forwarding \
        --lb-name kubernetes-lb \
        --lb-address-pools kubernetes-lb-pool > /dev/null

    echo "[Controller ${i}] Creating VM..."
    az vm create -g kubernetes \
        -n controller-${i} \
        --image ${UBUNTULTS} \
        --generate-ssh-keys \
        --nics controller-${i}-nic \
        --availability-set controller-as \
        --nsg '' \
        --admin-username 'kuberoot' > /dev/null \
        --generate-ssh-keys
done
```

### 쿠버네티스 워커 노드

각 작업자 인스턴스에는 Kubernetes 클러스터 CIDR 범위의 파드 서브넷 할당이 필요합니다. 포드 서브넷 할당은 나중에 실습에서 컨테이너 네트워킹을 구성하는 데 사용됩니다. `pod-cidr` 인스턴스 메타 데이터는 포드 서브넷 할당을 노출시 런타임에 인스턴스를 계산하는 데 사용됩니다.

> Kubernetes 클러스터 CIDR 범위는 Controller Manager의 `--cluster-cidr` 플래그로 정의됩니다. 이 자습서에서 클러스터 CIDR 범위는 `10.240.0.0/16` 으로 설정되며 254 개의 서브넷을 지원합니다.

`worker-as` 가용성 집합에서 Kubernetes 작업자 노드를 호스팅 할 계산 인스턴스를 세 개 만듭니다.

```shell
az vm availability-set create -g kubernetes -n worker-as
```

```shell
for i in 0 1 2; do
    echo "[Worker ${i}] Creating public IP..."
    az network public-ip create -n worker-${i}-pip -g kubernetes > /dev/null

    echo "[Worker ${i}] Creating NIC..."
    az network nic create -g kubernetes \
        -n worker-${i}-nic \
        --private-ip-address 10.240.0.2${i} \
        --public-ip-address worker-${i}-pip \
        --vnet kubernetes-vnet \
        --subnet kubernetes-subnet \
        --ip-forwarding > /dev/null

    echo "[Worker ${i}] Creating VM..."
    az vm create -g kubernetes \
        -n worker-${i} \
        --image ${UBUNTULTS} \
        --nics worker-${i}-nic \
        --tags pod-cidr=10.200.${i}.0/24 \
        --availability-set worker-as \
        --nsg '' \
        --admin-username 'kuberoot' > /dev/null \
        --generate-ssh-keys
done
```

### 확인

만들어진 가상 컴퓨터 인스턴스들을 확인합니다.

```shell
az vm list -d -g kubernetes -o table
```

> 출력

```shell
Name          ResourceGroup    PowerState    PublicIps       Location
------------  ---------------  ------------  --------------  ----------
controller-0  kubernetes       VM running    XX.XXX.XXX.XXX  westus2
controller-1  kubernetes       VM running    XX.XXX.XXX.XXX  westus2
controller-2  kubernetes       VM running    XX.XXX.XXX.XXX  westus2
worker-0      kubernetes       VM running    XX.XXX.XXX.XXX  westus2
worker-1      kubernetes       VM running    XX.XXX.XXX.XXX  westus2
worker-2      kubernetes       VM running    XX.XXX.XXX.XXX  westus2
```

다음: [CA 프로비저닝 및 TLS 인증서 생성](04-certificate-authority.md)

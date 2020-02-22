# 파드 네트워크 경로 프로비저닝

노드에 예약된 파드는 노드의 파드 CIDR 범위에서 IP 주소를 할당받습니다. 이 시점에서 파드는 [라우팅 테이블 항목](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview) 누락으로 인해 다른 노드에서 실행중인 다른 파드와 통신 할 수 없습니다.

이 실습에서는 노드의 파드 CIDR 범위를 노드의 내부 IP 주소에 매핑하는 각 작업자 노드에 대한 경로를 만듭니다.

> 쿠버네티스 네트워킹 모델을 구현하는 [방법들](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this)은 여러 가지가 있습니다.

## 라우팅 테이블

이 섹션에서는 kubernetes-vnet에서 경로를 만드는 데 필요한 정보를 수집합니다.

각 워커 인스턴스의 내부 IP 주소 및 포드 CIDR 범위를 인쇄합니다.

```shell
for instance in worker-0 worker-1 worker-2; do
  PRIVATE_IP_ADDRESS=$(az vm show -d -g kubernetes -n ${instance} --query "privateIps" -otsv)
  POD_CIDR=$(az vm show -g kubernetes --name ${instance} --query "tags" -o tsv)
  echo $PRIVATE_IP_ADDRESS $POD_CIDR
done
```

> 출력

```shell
10.240.0.20 10.200.0.0/24
10.240.0.21 10.200.1.0/24
10.240.0.22 10.200.2.0/24
```

## 라우팅 테이블

워커 인스턴스간의 네트워크 경로를 만듭니다.

```shell
az network route-table create -g kubernetes -n kubernetes-routes
```

```shell
az network vnet subnet update -g kubernetes \
  -n kubernetes-subnet \
  --vnet-name kubernetes-vnet \
  --route-table kubernetes-routes
```

```shell
for i in 0 1 2; do
az network route-table route create -g kubernetes \
  -n kubernetes-route-10-200-${i}-0-24 \
  --route-table-name kubernetes-routes \
  --address-prefix 10.200.${i}.0/24 \
  --next-hop-ip-address 10.240.0.2${i} \
  --next-hop-type VirtualAppliance
done
```

`kubernetes-vnet` 라우팅 테이블을 확인합니다.

```shell
az network route-table route list -g kubernetes --route-table-name kubernetes-routes -o table
```

> 출력

```shell
AddressPrefix    Name                            NextHopIpAddress    NextHopType       ProvisioningState    ResourceGroup
---------------  ------------------------------  ------------------  ----------------  -------------------  ---------------
10.200.0.0/24    kubernetes-route-10-200-0-0-24  10.240.0.20         VirtualAppliance  Succeeded            kubernetes
10.200.1.0/24    kubernetes-route-10-200-1-0-24  10.240.0.21         VirtualAppliance  Succeeded            kubernetes
10.200.2.0/24    kubernetes-route-10-200-2-0-24  10.240.0.22         VirtualAppliance  Succeeded            kubernetes
```

다음: [DNS 클러스터 애드온 배포](12-dns-addon.md)

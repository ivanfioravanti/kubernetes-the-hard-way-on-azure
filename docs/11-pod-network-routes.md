# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-vnet` Virtual Network.

Print the internal IP address and Pod CIDR range for each worker instance:

```shell
for instance in worker-0; do
  PRIVATE_IP_ADDRESS=$(az vm show -d -g kubernetes -n ${instance} --query "privateIps" -otsv)
  POD_CIDR=$(az vm show -g kubernetes --name ${instance} --query "tags" -o tsv)
  echo $PRIVATE_IP_ADDRESS $POD_CIDR
done
```

> output

```shell
10.240.0.20 10.200.0.0/24
```

## Routes

Create network routes for worker instance:

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

List the routes in the `kubernetes-vnet` VPC network:

```shell
az network route-table route list -g kubernetes --route-table-name kubernetes-routes -o table
```

> output

```shell
AddressPrefix    Name                            NextHopIpAddress    NextHopType       ProvisioningState    ResourceGroup
---------------  ------------------------------  ------------------  ----------------  -------------------  ---------------
10.200.0.0/24    kubernetes-route-10-200-0-0-24  10.240.0.20         VirtualAppliance  Succeeded            kubernetes
10.200.1.0/24    kubernetes-route-10-200-1-0-24  10.240.0.21         VirtualAppliance  Succeeded            kubernetes
10.200.2.0/24    kubernetes-route-10-200-2-0-24  10.240.0.22         VirtualAppliance  Succeeded            kubernetes
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)

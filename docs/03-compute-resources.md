# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster within a single [Resource Group](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-overview#resource-groups) in a single [region](https://azure.microsoft.com/en-us/regions/)
Create a default Resource Group in a region
> Ensure a resource group has been created as described in the [Prerequisites](01-prerequisites.md#create-a-deafult-resource-group-in-a-region) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Network

In this section a dedicated [Virtual Network](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview) (VNet) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-vnet` custom VNet network with a subnet `kubernetes` provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.:

```shell
az network vnet create -g kubernetes \
  -n kubernetes-vnet \
  --address-prefix 10.240.0.0/24 \
  --subnet-name kubernetes-subnet
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules

Create a firewall ([Network Security Group](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-nsg)) and assign it to the subnet:

```shell
az network nsg create -g kubernetes -n kubernetes-nsg
```

```shell
az network vnet subnet update -g kubernetes \
  -n kubernetes-subnet \
  --vnet-name kubernetes-vnet \
  --network-security-group kubernetes-nsg
```

Create a firewall rule that allows external SSH and HTTPS:

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

> An [external load balancer](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `kubernetes-vnet` VNet network:

```shell
az network nsg rule list -g kubernetes --nsg-name kubernetes-nsg --query "[].{Name:name, \
  Direction:direction, Priority:priority, Port:destinationPortRange}" -o table
```

> output

```shell
Name                         Direction      Priority    Port
---------------------------  -----------  ----------  ------
kubernetes-allow-ssh         Inbound            1000      22
kubernetes-allow-api-server  Inbound            1001    6443
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```shell
az network lb create -g kubernetes \
  -n kubernetes-lb \
  --backend-pool-name kubernetes-lb-pool \
  --public-ip-address kubernetes-pip \
  --public-ip-address-allocation static
```

Verify the `kubernetes-pip` static IP address was created correctly in the `kubernetes` Resource Group and chosen region:

```shell
az network public-ip  list --query="[?name=='kubernetes-pip'].{ResourceGroup:resourceGroup, \
  Region:location,Allocation:publicIpAllocationMethod,IP:ipAddress}" -o table
```

> output

```shell
ResourceGroup    Region    Allocation    IP
---------------  --------  ------------  --------------
kubernetes       westus2   Static        XX.XXX.XXX.XXX
```

## Virtual Machines

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [cri-containerd container runtime](https://github.com/kubernetes-incubator/cri-containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

To select latest stable Ubuntu Server release run following command and replace UBUNTULTS variable below with latest row in the table.

```shell
az vm image list --location westus2 --publisher Canonical --offer UbuntuServer --sku 18.04-LTS --all -o table
```

```shell
UBUNTULTS="Canonical:UbuntuServer:18.04-LTS:18.04.201812060"
```

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane in `controller-as` [Availability Set](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/regions-and-availability#availability-sets):

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
        --admin-username 'kuberoot' > /dev/null
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.240.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes in `worker-as` Availability Set:

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
        --admin-username 'kuberoot' > /dev/null
done
```

### Verification

List the compute instances in your default compute zone:

```shell
az vm list -d -g kubernetes -o table
```

> output

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

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)

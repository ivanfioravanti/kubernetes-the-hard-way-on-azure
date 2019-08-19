# Smoke Test

In this lab you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.

## Data Encryption

In this section you will verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:

```shell
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:

```shell
CONTROLLER="controller-0"
PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
  -n ${CONTROLLER}-pip --query "ipAddress" -otsv)

ssh kuberoot@${PUBLIC_IP_ADDRESS} \
  "ETCDCTL_API=3 etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> output

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

The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

## Deployments

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```shell
kubectl run --generator=run-pod/v1 nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```shell
kubectl get pods -l run=nginx
```

> output

```shell
NAME                     READY     STATUS    RESTARTS   AGE
nginx                    1/1       Running   0          15s
```

### Port Forwarding

In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:

```shell
POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```shell
kubectl port-forward $POD_NAME 8080:80
```

> output

```shell
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```shell
curl --head http://127.0.0.1:8080
```

> output

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

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```shell
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

In this section you will verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` pod logs:

```shell
kubectl logs $POD_NAME
```

> output

```shell
127.0.0.1 - - [16/Jun/2019:08:51:13 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.54.0" "-"
```

### Exec

In this section you will verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```shell
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```shell
nginx version: nginx/1.17.0
```

## Services

In this section you will verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```shell
kubectl expose pod nginx --port 80 --type NodePort
```

> The LoadBalancer service type can not be used because your cluster is not configured with [cloud provider integration](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider). Setting up cloud provider integration is out of scope for this tutorial.

Retrieve the node port assigned to the `nginx` service:

```shell
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Create a firewall rule that allows remote access to the `nginx` node port:

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

Retrieve the external IP address of a worker instance:

```shell
EXTERNAL_IP=$(az network public-ip show -g kubernetes \
  -n worker-0-pip --query "ipAddress" -otsv)
```

Make an HTTP request using the external IP address and the `nginx` node port:

```shell
curl -I http://$EXTERNAL_IP:$NODE_PORT
```

> output

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

Next: [Dashboard configuration](14-dashboard.md)

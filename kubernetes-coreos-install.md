
# CoreOS Way to install kubernetes

single-master/multi-worker Kubernetes cluster on CoreOS.

* configure an etcd cluster for Kubernetes to use
* create the certificates for communication between Kubernetes components
* deploy a master node
* deploy worker nodes
* configure kubectl to work with the cluster.
* deploy the DNS add-on

- use etcd for data storage and for cluster consensus.
- use Fleet for cluster management. Fleet is from CoreOS.
  . Fleet builds on top of systemd. 
  . systemd provides system and service initialization for a machine.
  . Fleet does the same thing for a cluster of machines.
  . Fleet supports "socket activation" - a container spins up when 
    the connection is on a given port.
- components
  . Pods: groups of containers
  . Flat Networking Space: all pods can talk to each other without NAT.
  . Labels: key-value pair attached to objects like pods "type":"redis"
  . Services: a group of pods. can be connected to pods with labels
    ex) "cache" service connect to several "redis" pods 
    with label "type":"redis". 
    Then, the service will automatically round-robin requests between pods.
  . Replication Controllers: control and monitor the # of running pods for
    the service.

## Deployment Options

* MASTER_HOST=192.168.24.11 (coreos1)
  - run Kubernetes API endpoint
* ETCD_ENDPOINTS=http://192.168.24.11:2379,...
* POD_NETWORK=10.2.0.0/16
  - used for pod IPs
* SERVICE_IP_RANGE=10.3.0.0/24
  - use for service cluster VIPs (local kube-proxy is handling routing to VIP)
* K8S_SERVICE_IP=10.3.0.1
  - VIP of Kubernetes API service (must be the first IP of SERVICE_IP_RANGE)
* DNS_SERVICE_IP=10.3.0.10
  - VIP of cluster DNS service (must be in the range of SERVICE_IP_RANGE)
  - all worker nodes should be configured the same DNS_SERVICE_IP.

## Deploy etcd cluster

## Generate Kubernetes TLS Assets

API server use certificate auth for validating clients.

### Deployment Options

* WORKER_IP=192.168.24.{12,13,14}
* WORKER_FQDN=coreos{2,3,4}
  - IP and fqdn of all workers are needed.

### Create a cluster root CA

```
$ mkdir k8s_tls_assets
$ cd k8s_tls_assets
$ openssl genrsa -out ca-key.pem 2048
$ openssl req -x509 -new -nodes -key ca-key.pem -days 10000 -out ca.pem \
  -subj "/CN=kube-ca"
```

### Kubernetes API Server Keypair

* creating openssl config file
```
$ vi openssl.cnf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = ${K8S_SERVICE_IP}
IP.2 = ${MASTER_HOST}
```

* generating API server Keypair
```
$ openssl genrsa -out apiserver-key.pem 2048
$ openssl req -new -key apiserver-key.pem -out apiserver.csr -subj
"/CN=kube-apiserver" -config openssl.cnf
$ openssl x509 -req -in apiserver.csr -CA ca.pem -CAkey ca-key.pem
-CAcreateserial -out apiserver.pem -days 365 -extensions v3_req -extfile
openssl.cnf
```

### Kubernetes Worker Keypairs

* creating common worker openssl config file.
```
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = $ENV::WORKER_IP
```
* Generating kubernetes worker keypair
```
$ openssl genrsa -out ${WORKER_FQDN}-worker-key.pem 2048
$ WORKER_IP=${WORKER_IP} openssl req -new -key ${WORKER_FQDN}-worker-key.pem
-out ${WORKER_FQDN}-worker.csr -subj "/CN=${WORKER_FQDN}" -config
worker-openssl.cnf
$ WORKER_IP=${WORKER_IP} openssl x509 -req -in ${WORKER_FQDN}-worker.csr -CA
ca.pem -CAkey ca-key.pem -CAcreateserial -out ${WORKER_FQDN}-worker.pem -days
365 -extensions v3_req -extfile worker-openssl.cnf
```

### Cluster administrator keypair
```
$ openssl genrsa -out admin-key.pem 2048
$ openssl req -new -key admin-key.pem -out admin.csr -subj "/CN=kube-admin"
$ openssl x509 -req -in admin.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial
-out admin.pem -days 365
```



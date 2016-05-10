
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

## Deploy Kubernetes Master Node.

CoreOS uses kubelet-wrapper. 
It's there since CoreOS version 962.0.0
So it is not in current stable release (899.x.x)
Let's install it manually.

* Log into coreos1 which is a master node.
* Create /opt/bin/
  - sudo mkdir -p /opt/bin/
* Get and save a copy of kubelet-wrapper bash script in /opt/bin/.
  - https://github.com/coreos/coreos-overlay/blob/master/app-admin/kubelet-wrapper/files/kubelet-wrapper
* Make the script executable
  - sudo chmod +x /opt/bin/kubelet-wrapper

### Configure Service Components

#### TLS Assets
```
$ sudo mkdir -p /etc/kubernetes/ssl
$ sudo cp ~/k8s_tls_assets/{ca,apiserver-key,apiserver}.pem /etc/kubernetes/ssl/
$ sudo chmod 600 /etc/kubernetes/ssl/*-key.pem
$ sudo chown root:root /etc/kubernetes/ssl/*-key.pem
```

#### Network Configuration

Network components are Flannel and Calico.
* flannel: software-defined overlay network for routing traffic of the pods
* calico: restricting traffic of pods based on fine-grained network policy
```
$ sudo mkdir /etc/flannel
$ sudo vi /etc/flannel/options.env
FLANNELD_IFACE=192.168.24.11
FLANNELD_ETCD_ENDPOINTS=http://192.168.24.11:2379,http://192.168.24.12:2379,http://192.168.24.13:2379,http://192.168.24.14:2379
$ sudo mkdir -p /etc/systemd/system/flanneld.service.d
$ sudo vi /etc/systemd/system/flanneld.service.d/40-ExecStartPre-symlink.conf
[Service]
ExecStartPre=/usr/bin/ln -sf /etc/flannel/options.env /run/flannel/options.env
```

#### Docker configuration
Docker uses flannel to manage the pod network in the cluster.
```
$ sudo mkdir -p /etc/systemd/system/docker.service.d
$ sudo vi /etc/systemd/system/docker.service.d/40-flannel.conf
[Unit]
Requires=flanneld.service
After=flanneld.service
```

#### Create the kubelet Unit

kubelet is the agent on every master and node.
* It starts/stops Pods and does other machine-level tasks.
* It communicates with the API server with TLS certificates.
* On master node, it does not register for cluster work
  (--register-schedulable=false)
* It is configured to use Container Networking Interface(CNI) for networking.
```
$ sudo vi /etc/systemd/system/kubelet.service
[Service]
ExecStartPre=/usr/bin/mkdir -p /etc/kubernetes/manifests

Environment=KUBELET_VERSION=v1.2.3_coreos.cni.1
ExecStart=/opt/bin/kubelet-wrapper \
  --api-servers=http://127.0.0.1:8080 \
  --network-plugin-dir=/etc/kubernetes/cni/net.d \
  --network-plugin=cni \
  --register-schedulable=false \
  --allow-privileged=true \
  --config=/etc/kubernetes/manifests \
  --hostname-override=192.168.24.11 \
  --cluster-dns=10.3.0.10 \
  --cluster-domain=cluster.local
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

#### Set up the kube-apiserver Pod
kube-apiserver 
* is steteless.
* takes in API requests.
* processes requests.
* stores the result in etcd.
* returns the result.
```
$ sudo mkdir -p /etc/kubernetes/manifests
$ sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-apiserver
    image: quay.io/coreos/hyperkube:v1.2.3_coreos.0
    command:
    - /hyperkube
    - apiserver
    - --bind-address=0.0.0.0
    - --etcd-servers=http://192.168.24.11:2379,http://192.168.24.12:2379,http://192.168.24.13:2379,http://192.168.24.14:2379
    - --allow-privileged=true
    - --service-cluster-ip-range=10.3.0.0/24
    - --secure-port=443
    - --advertise-address=192.168.24.11
    - --admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota
    - --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem
    - --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --client-ca-file=/etc/kubernetes/ssl/ca.pem
    - --service-account-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --runtime-config=extensions/v1beta1=true,extensions/v1beta1/thirdpartyresources=true
    ports:
    - containerPort: 443
      hostPort: 443
      name: https
    - containerPort: 8080
      hostPort: 8080
      name: local
    volumeMounts:
    - mountPath: /etc/kubernetes/ssl
      name: ssl-certs-kubernetes
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/ssl
    name: ssl-certs-kubernetes
  - hostPath:
      path: /usr/share/ca-certificates
    name: ssl-certs-host
```

#### Set up the kube-proxy Pod
kube-proxy is
* directing traffic for specific services and pods to the correct location.
* communicating with the API server periodically to keep up to date.
* running on both master and worker nodes.
```
$ sudo vi /etc/kubernetes/manifests/kube-proxy.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-proxy
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-proxy
    image: quay.io/coreos/hyperkube:v1.2.3_coreos.0
    command:
    - /hyperkube
    - proxy
    - --master=http://127.0.0.1:8080
    - --proxy-mode=iptables
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
  volumes:
  - hostPath:
      path: /usr/share/ca-certificates
    name: ssl-certs-host
```

#### Set up the kube-controller-manager Pod
kube-controller-manager
* takes any required action based on changes to Replication Controllers.
```
$ sudo vi /etc/kubernetes/manifests/kube-controller-manager.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-controller-manager
    image: quay.io/coreos/hyperkube:v1.2.3_coreos.0
    command:
    - /hyperkube
    - controller-manager
    - --master=http://127.0.0.1:8080
    - --leader-elect=true 
    - --service-account-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --root-ca-file=/etc/kubernetes/ssl/ca.pem
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10252
      initialDelaySeconds: 15
      timeoutSeconds: 1
    volumeMounts:
    - mountPath: /etc/kubernetes/ssl
      name: ssl-certs-kubernetes
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/ssl
    name: ssl-certs-kubernetes
  - hostPath:
      path: /usr/share/ca-certificates
    name: ssl-certs-host
```

#### Set up the kube-scheduler Pod
kube-scheduler
* monitors the API for unscheduled pods.
* finds a machine to run on for them.
* communicates the decision back to the API.
```
$ sudo vi /etc/kubernetes/manifests/kube-scheduler.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-scheduler
    image: quay.io/coreos/hyperkube:v1.2.3_coreos.0
    command:
    - /hyperkube
    - scheduler
    - --master=http://127.0.0.1:8080
    - --leader-elect=true
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10251
      initialDelaySeconds: 15
      timeoutSeconds: 1
```

#### Set up Calico Node Container
Calico node container
* runs on both master and worker nodes.
* 2 functions
  - connects containers to the flannel overlay network (one IP per pod)
  - enforces network policy, ensuring pods talk to authorized resources only.

```
$ sudo vi /etc/systemd/system/calico-node.service
[Unit]
Description=Calico per-host agent
Requires=network-online.target
After=network-online.target

[Service]
Slice=machine.slice
Environment=CALICO_DISABLE_FILE_LOGGING=true
Environment=HOSTNAME=192.168.24.11
Environment=IP=192.168.24.11
Environment=FELIX_FELIXHOSTNAME=192.168.24.11
Environment=CALICO_NETWORKING=false
Environment=NO_DEFAULT_POOLS=true
Environment=ETCD_ENDPOINTS=http://192.168.24.11:2379,http://192.168.24.12:2379,http://192.168.24.13:2379,http://192.168.24.14:2379
ExecStart=/usr/bin/rkt run --inherit-env --stage1-from-dir=stage1-fly.aci \
--volume=modules,kind=host,source=/lib/modules,readOnly=false \
--mount=volume=modules,target=/lib/modules \
--trust-keys-from-https quay.io/calico/node:v0.19.0

KillMode=mixed
estart=always
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

#### Set up the policy-agent Pod
policy-agent 
* monitors the API for changes related to network policy.
* configures Calico to implement that policy.
```
$ sudo vi /etc/kubernetes/manifests/policy-agent.yaml
apiVersion: v1
kind: Pod 
metadata:
  name: calico-policy-agent
  namespace: calico-system 
spec:
  hostNetwork: true
  containers:
    # The Calico policy agent.
    - name: k8s-policy-agent
      image: calico/k8s-policy-agent:v0.1.4
      env:
        - name: ETCD_ENDPOINTS
          value: "http://192.168.24.11:2379,http://192.168.24.12:2379,http://192.168.24.13:2379,http://192.168.24.14:2379"
        - name: K8S_API
          value: "http://127.0.0.1:8080"
        - name: LEADER_ELECTION 
          value: "true"
    # Leader election container used by the policy agent.
    - name: leader-elector
      image: quay.io/calico/leader-elector:v0.1.0
      imagePullPolicy: IfNotPresent
      args:
        - "--election=calico-policy-election"
        - "--election-namespace=calico-system"
        - "--http=127.0.0.1:4040"
```

#### Set up the CNI config
CNI configuration
* is read by kubelet on startup.
* used to determine which cNI plugin to call.
* tells the kubelet to call the flannel plugin
* tells the kubelet to delegate control to Calico plugin.
```
$ sudo mkdir -p /etc/kubernetes/cni/net.d
$ sudo vi /etc/kubernetes/cni/net.d/10-calico.conf
{
    "name": "calico",
    "type": "flannel",
    "delegate": {
        "type": "calico",
        "etcd_endpoints": "http://192.168.24.11:2379,http://192.168.24.12:2379,http://192.168.24.13:2379,http://192.168.24.14:2379",
        "log_level": "none",
        "log_level_stderr": "info",
        "hostname": "192.168.24.11",
        "policy": {
            "type": "k8s",
            "k8s_api_root": "http://127.0.0.1:8080/api/v1/"
        }
    }
}
```


### Start Services

* Load changed units
  - $ sudo systemctl daemon-reload
* Configure flannel network
  - curl -X PUT -d \
    "value={\"Network\":\"10.2.0.0/16\",\"Backend\":{\"Type\":\"vxlan\"}}" \
    "http://192.168.24.11:2379/v2/keys/coreos.com/network/config"
  - etcdctl get /coreos.com/network/config
    {"Network":"10.2.0.0/16","Backend":{"Type":"vxlan"}}
* Start kubelet
  - $ sudo systemctl start kubelet  (This takes a long time)
  - $ sudo systemctl enable kubelet
* Start Calico
  - $ sudo systemctl start calico-node
  - $ sudo systemctl enable calico-node
* Create Namespaces
  - Pods exist in their own namespace. so create namespace for the cluster.
  - check if kubernetes API server is up.
    $ curl http://127.0.0.1:8080/version
    {
      "major": "1",
      "minor": "2",
      "gitVersion": "v1.2.3+coreos.0",
      "gitCommit": "c2d31de51299c6239d8b061e63cec4cb4a42480b",
      "gitTreeState": "clean"
    }
  - If up, create kube-system namespace
    ```
    $ curl -H "Content-Type: application/json" -XPOST -d'{"apiVersion":"v1","kind":"Namespace","metadata":{"name":"kube-system"}}'
    "http://127.0.0.1:8080/api/v1/namespaces"
    {
      "kind": "Namespace",
      "apiVersion": "v1",
      "metadata": {
        "name": "kube-system",
        "selfLink": "/api/v1/namespaces/kube-system",
        "uid": "2ca8c779-16b0-11e6-b3c0-00163e147140",
        "resourceVersion": "635958",
        "creationTimestamp": "2016-05-10T13:07:40Z"
    },
    "spec": {
      "finalizers": [
        "kubernetes"
      ]
    },
    "status": {
      "phase": "Active"
    }
    ```
  - Calico policy-agent runs in its own calico-system namespace. Create it.
    ```
    $ curl -H "Content-Type: application/json" -XPOST -d'{"apiVersion":"v1","kind":"Namespace","metadata":{"name":"calico-system"}}' "http://127.0.0.1:8080/api/v1/namespaces"
    {
      "kind": "Namespace",
      "apiVersion": "v1",
      "metadata": {
        "name": "calico-system",
        "selfLink": "/api/v1/namespaces/calico-system",
        "uid": "a480249a-16b1-11e6-b3c0-00163e147140",
        "resourceVersion": "637049",
        "creationTimestamp": "2016-05-10T13:18:10Z"
      },
      "spec": {
        "finalizers": [
          "kubernetes"
        ]
      },
      "status": {
        "phase": "Active"
      }
      ```
* Enable network policy API
  - Network policy in Kubernetes is implemented as a third party resource.
    ```
    $ curl -H "Content-Type: application/json" -XPOST http://127.0.01:8080/apis/extensions/v1beta1/namespaces/default/thirdpartyresources --data-binary @- <<BODY
    > {
    >   "kind": "ThirdPartyResource",
    >   "apiVersion": "extensions/v1beta1",
    >   "metadata": {
    >     "name": "network-policy.net.alpha.kubernetes.io"
    >   },
    >   "description": "Specification for a network isolation policy",
    >   "versions": [
    >     {
    >       "name": "v1alpha1"
    >     }
    >   ]
    > }
    > BODY
    {
      "kind": "ThirdPartyResource",
      "apiVersion": "extensions/v1beta1",
      "metadata": {
        "name": "network-policy.net.alpha.kubernetes.io",
        "namespace": "default",
        "selfLink": "/apis/extensions/v1beta1/namespaces/default/thirdpartyresources/network-policy.net.alpha.kubernetes.io",
        "uid": "544a9d6d-16b2-11e6-b3c0-00163e147140",
        "resourceVersion": "637541",
        "creationTimestamp": "2016-05-10T13:23:05Z"
      },
      "description": "Specification for a network isolation policy",
      "versions": [
        {
          "name": "v1alpha1",
          "apiGroup": "extensions"
        }
      ]
    }
    ```
  - check the health of kubelet systemd unit
    $ sudo systemctl status kubelet.service


## Kubernetes Worker Nodes


# Metal Kubes 
## Install Kubernetes Cluster on Bare Metal Machines
### Using only Open Source Tools

    Disclaimer: This document is *not* intended for Production use. Best Practices recommended at kubernetes.io and other sites must be charted. Motivated by Kelsey Hightower doc - Kubernetes The Hard Way

Principal drive for crafting this manuscript is for my reference. I created multiple Kubernetes Clusters using contrasting hardware. The document attested very beneficial when setting up OnPrem Kubernetes Cluster at work. I hope you will find it worthwhile.

## Install kubectl

kubectl (aka kube control or "kubecutle" or "kube-c-t-l) is your primary command line utility to interact with Kubernetes. 
Its a go binary available for all platforms.

Source Code: https://github.com/kubernetes/kubernetes/tree/master/pkg/kubectl

    wget -qc https://storage.googleapis.com/kubernetes-release/release/v1.20.2/bin/linux/amd64/kubectl

    chmod 755 kubectl

    sudo mv kubectl /usr/local/bin/ 

(or ~/bin - where ~/bin folder present in your $PATH)

Verify kubectl version 1.20.2 or higher is installed:

    kubectl version --client

```console
Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.2", GitCommit:"faecb196815e248d3ecfb03c680a4507229c2a56", GitTreeState:"clean", BuildDate:"2021-01-13T13:28:09Z", GoVersion:"go1.15.5", Compiler:"gc", Platform:"linux/amd64"}
```

Windows:
```console
Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.2", GitCommit:"faecb196815e248d3ecfb03c680a4507229c2a56", GitTreeState:"clean", BuildDate:"2021-01-13T13:28:09Z", GoVersion:"go1.15.5", Compiler:"gc", Platform:"windows/amd64"}
```

(you will need kubeconfig in your ~/.kube or KUBECONFIG for secure interaction, which we will cover later)

> alias k=kubectl # it will be used extensively

## Provision PKI Infrastructure and generate TLS certificates using CloudFlare's PKI toolkits

Download, Build, and install cfssl and cfssljson:

    git clone https://github.com/cloudflare/cfssl.git
    make
    sudo mv cfssl cfssljson /usr/local/bin/ 

(or ~/bin - where ~/bin folder present in your $PATH)

cfssl and cfssljson version:

```console
$ cfssl version
Version: 1.5.0
Runtime: go1.16.2
```

cfssljson --version

```console
$ cfssljson --version
Version: 1.5.0
Runtime: go1.16.2
```

## Create VMs for your compute nodes

You will need odd number of nodes in your control plane (preferably 3 though you can get away with 1). 
You want to avoid split-brain as kube-scheduler and kube-control-manager run on it. 
Even though only one kube-scheduler is active at any one time, if it goes down, one of others will take over. 
You may even have etcd server on your control nodes. 

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. 
In this section you will provision the compute resources required for running a secure and highly available Kubernetes cluster.

We will create bridge network on each host machine, and make sure each VM gets its IP Address from your Router, this will insure flat networking. 
You will need to assign static IP Address to your nodes (using MAC Address on your Router should ensure that)

Use virt-manager on host machine (ArchLinux/Ubuntu/Centos/OpenSuse) to create VMs,

With NetworkManager or systemd.networkd or netplan, create bridge network (br0).

Using NetworkManager,

```console
$ nmcli con show
NAME                UUID                                  TYPE      DEVICE
Wired connection 1  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  ethernet  enp6s0

nmcli con show 'Wired connection 1'

sudo nmcli con add ifname br0 type bridge con-name br0
sudo nmcli con add type bridge-slave ifname enp6s0 master br0
sudo nmcli con modify br0 bridge.stp no
sudo nmcli con down 'Wired connection 1'
# might loose connection
sudo nmcli con up br0
```

```console
$ nmcli con show
NAME                 UUID                                  TYPE      DEVICE
br0                  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  bridge    br0
bridge-slave-enp6s0  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  ethernet  enp6s0
```

```console
$ ip addr show
...
2: enp6s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br0 state UP group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet 10.244.10.177/24 brd 10.244.10.255 scope global noprefixroute br0
       valid_lft forever preferred_lft forever
    inet6 xxxx::xxxx:xxxx:xxx:xxxx/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

## Networking

The Kubernetes networking model assumes a flat network in which containers and nodes can communicate with each other. 
In cases where this is not desired network policies can limit how groups of containers are allowed to communicate with each other and external network endpoints.

A subnet must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Firewall Rules (Not needed on premise setup, as all nodes will be open behind your Router - DHCP)

An external load balancer will be used to expose the Kubernetes API Servers to remote clients, covered later.

Kubernetes Public IP Address
Using MAC Address, assign a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

### Compute Nodes

The compute nodes in this section will be provisioned using Ubuntu Server 20.04, which has good support for the containerd container runtime.
Each compute node will be provisioned with a static IP address to simplify the Kubernetes bootstrapping process.

#### Kubernetes Controllers (kube-maestro-?)

Create 3 compute nodes which will host the Kubernetes control plane:

#### Kubernetes Workers

Each worker node requires a pod subnet allocation from the Kubernetes cluster CIDR (*cider*) range.
The pod subnet allocation will be used to configure container networking in a later exercise. 
The pod-cidr node metadata will be used to expose pod subnet allocations to compute nodes at runtime.

The Kubernetes cluster CIDR range is defined by the Controller Manager's --cluster-cidr flag. 
In this tutorial the cluster CIDR range will be set to 10.100.0.0/16, which supports 254 subnets.

Create 6 compute nodes which will host the Kubernetes worker nodes (you can add Worker nodes at anytime, as showwn later):

List the compute nodes (this should go to /etc/hosts of your auxiliary machine):

```console
10.244.10.156 kube-lb
10.244.10.153 kube-maestro-0
10.244.10.154 kube-maestro-1
10.244.10.155 kube-maestro-2
10.244.10.150 kube-minion-0
10.244.10.151 kube-minion-1
10.244.10.152 kube-minion-2
10.244.10.136 kube-artisan-0
10.244.10.137 kube-artisan-1
10.244.10.138 kube-artisan-2
```

> Use ssh-copy-id script to install your auxiliary machine public key(s) each node

Test SSH access to the kube-maestro-0 compute node:

    ssh kube-maestro-0

> Make sure /etc/resolv.conf works properly (soft linked to /run/systemd/resolve/resolv.conf in my case)
#### External IP Address (EXT_IP)

Find your External IP, which will be used throughout as EXT_IP (assigned by your ISP - be mindful it can change)

Use one of the following:

```console
dig +short myip.opendns.com @resolver1.opendns.com
dig @ns1.google.com -t txt o-o.myaddr.l.google.com +short
dig -4 @ns1-1.akamaitech.net -t a whoami.akamai.net +short
curl ifconfig.me/ip
```

> In my case: EXT_IP=99.98.10.12

###### Environment variables used across all scripts:
```shell
EXT_IP=99.98.10.12
KUBERNETES_PUBLIC_ADDRESS=10.244.10.156 # IP Address of your External LB
KUBE_MAESTRO_0=10.244.10.153
KUBE_MAESTRO_1=10.244.10.154
KUBE_MAESTRO_2=10.244.10.155
POD_CIDR=10.100.0.0/16
Cluster CIDR: 10.50.0.0/24
```

## Provisioning a CA and Generate TLS Certificates

Provision a PKI Infrastructure using cfssl, then use it to bootstrap a Certificate Authority,and generate TLS certificates for the following components:
  
> etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, kube-proxy.

#### Certificate Authority

In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates.

Generate the CA configuration file, certificate, and private key:

```console
$ cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

$ cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Folsom",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "California"
    }
  ]
}
EOF

$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca

ca-key.pem
ca.pem
```

> Save ca-config.json, ca.pem, ca-key.pem, as they will be needed anytime you want to add new worker node

#### Client and Server Certificates

Generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes admin user.

##### The Admin Client Certificate

Generate the admin client certificate and private key:

```console
$ cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Folsom",
      "O": "system:masters",
      "OU": "Metal Kubes",
      "ST": "California"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

admin-key.pem
admin.pem
```

##### The Kubelet Client Certificates

Kubernetes uses a special-purpose authorization mode called Node Authorizer, that specifically authorizes API requests made by Kubelets.
In order to be authorized by the Node Authorizer, kubelets must use a credential that identifies them as being in the system:nodes group,
with a username of system:node:<nodeName>.
In this section you will create a certificate for each Kubernetes worker node that meets the Node Authorizer requirements.

Generate a certificate and private key for each Kubernetes worker node:

```console
$ for k in kube-artisan-0 kube-artisan-1 kube-artisan-2 kube-minion-0 kube-minion-1 kube-minion-2; do
cat > ${k}-csr.json <<EOF
{
  "CN": "system:node:${k}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Folsom",
      "O": "system:nodes",
      "OU": "Metal Kubes",
      "ST": "California"
    }
  ]
}
EOF

EXT_IP=99.98.10.12
INT_IP=$(ping -q -c 1 ${k} |grep  -Po ' \(\K[\d.]+')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${k},${EXT_IP},${INT_IP} \
  -profile=kubernetes \
  ${k}-csr.json | cfssljson -bare ${k}
done

kube-artisan-0-key.pem
kube-artisan-0.pem
kube-artisan-1-key.pem
kube-artisan-1.pem
kube-artisan-2-key.pem
kube-artisan-2.pem
kube-minion-0-key.pem
kube-minion-0.pem
kube-minion-1-key.pem
kube-minion-1.pem
kube-minion-2-key.pem
kube-minion-2.pem
```
> openssl x509 -text -noout -in kube-artisan-0.pem (get details of the cert)

##### The Controller Manager Client Certificate

Generate the kube-controller-manager client certificate and private key:

```console
$ cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Folsom",
      "O": "system:kube-controller-manager",
      "OU": "Metal Kubes",
      "ST": "California"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

kube-controller-manager-key.pem
kube-controller-manager.pem
```

##### The Kube Proxy Client Certificate

Generate the kube-proxy client certificate and private key:

```console
$ cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Folsom",
      "O": "system:node-proxier",
      "OU": "Metal Kubes",
      "ST": "California"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

kube-proxy-key.pem
kube-proxy.pem
```

##### The Scheduler Client Certificate

Generate the kube-scheduler client certificate and private key:

```console
$ cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Folsom",
      "O": "system:kube-scheduler",
      "OU": "Metal Kubes",
      "ST": "California"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

kube-scheduler-key.pem
kube-scheduler.pem
```

##### The Kubernetes API Server Certificate

The metal-kubes static IP address will be included in the list of subject alternative names for the Kubernetes API Server certificate.
This will ensure the certificate can be validated by remote clients.

Generate the Kubernetes API Server certificate and private key:

> KUBERNETES_PUBLIC_ADDRESS is the IP Address of your External Load Balancer (kube-lb) - 10.244.10.156

```console
$ KUBERNETES_PUBLIC_ADDRESS=10.244.10.156
$ KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

$ cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Folsom",
      "O": "Kubernetes",
      "OU": "Metal Kubes",
      "ST": "California"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.50.0.1,${KUBE_MAESTRO_0},${KUBE_MAESTRO_1},${KUBE_MAESTRO_2},${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

kubernetes-key.pem
kubernetes.pem
```

  The Kubernetes API server is automatically assigned the kubernetes internal dns name,
  which will be linked to the first IP address (10.50.0.1) from the address range (10.50.0.0/24)
  reserved for internal cluster services during the control plane bootstrapping section.

##### The Service Account Key Pair

The Kubernetes Controller Manager leverages a key pair to generate and sign service account tokens as described in the managing service accounts documentation.

Generate the service-account certificate and private key:

```console
$ cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Folsom",
      "O": "Kubernetes",
      "OU": "Metal Kubes",
      "ST": "California"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

service-account-key.pem
service-account.pem
```

Distribute the Client and Server Certificates

Copy the appropriate certificates and private keys to each worker instance:

```console
$ for k in kube-artisan-0 kube-artisan-1 kube-artisan-2 kube-minion-0 kube-minion-1 kube-minion-2; do
  scp ca.pem ${k}-key.pem ${k}.pem ${k}:~/
done
```

Copy the appropriate certificates and private keys to each controller instance:

```console
$ for k in kube-maestro-0 kube-maestro-1 kube-maestro-2; do
  scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${k}:~/
done
```

> The kube-proxy, kube-controller-manager, kube-scheduler, and kubelet client certificates
> will be used to generate client authentication configuration files in the next section.

## Setup your External Load Balancer, First

Provision the Kubernetes Frontend Load Balancer

I setup Open Source Nginx Load Balancer compiled with flag "--with-stream" 

```console
 wget -qc https://ftp.pcre.org/pub/pcre/pcre-8.44.tar.gz
 wget -qc http://zlib.net/zlib-1.2.11.tar.gz
 wget -qc http://www.openssl.org/source/openssl-1.1.1g.tar.gz
 wget -qc https://nginx.org/download/nginx-1.19.0.tar.gz
 gzip -dc zlib-1.2.11.tar.gz |tar -xvf -
 gzip -dc pcre-8.44.tar.gz |tar -xvf -
 gzip -dc openssl-1.1.1g.tar.gz |tar -xvf -
 gzip -dc nginx-1.19.0.tar.gz |tar -xvf -

./configure \
  --sbin-path=/usr/local/nginx/nginx \
  --conf-path=/usr/local/nginx/nginx.conf \
  --pid-path=/usr/local/nginx/nginx.pid \
  --with-pcre=../pcre-8.44 \
  --with-zlib=../zlib-1.2.11 \
  --with-http_ssl_module \
  --with-stream \
  --with-mail=dynamic \
  --add-module=../nginx-rtmp-module
```

nginx.conf
----------

```console
#user  nobody;
worker_processes  1;

error_log  logs/error.log;
error_log  logs/error.log  notice;
error_log  logs/error.log  info;

#pid        logs/nginx.pid;

events {
    worker_connections  1024;
}

stream{
  upstream kubernetes {
      server kube-maestro-0:6443;
      server kube-maestro-1:6443;
      server kube-maestro-2:6443;
  }
  server {
      listen 6443;
      proxy_pass kubernetes;
  }

  upstream etcd {
      server kube-maestro-0:2379;
      server kube-maestro-1:2379;
      server kube-maestro-2:2379;
  }
  server {
      listen 2379;
      proxy_pass etcd;
  }
}
```

> You will be needing its IP Address as to generate the Kubernetes API Server certificate and private key.
> In my case it is the IP Address of my External Load Balancer - KUBERNETES_PUBLIC_ADDRESS=10.244.10.156

## Generating Kubernetes Configuration Files for Authentication

In this section you will generate Kubernetes configuration files, also known as kubeconfigs,
which enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers.

### Client Authentication Configs

In this section you will generate kubeconfig files for the controller manager, kubelet, kube-proxy, and scheduler clients and the admin user.

##### Kubernetes Public IP Address

Each kubeconfig requires a Kubernetes API Server to connect to.
To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

> KUBERNETES_PUBLIC_ADDRESS=10.244.10.156

The kubelet Kubernetes Configuration File

When generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used.
This will ensure Kubelets are properly authorized by the Kubernetes Node Authorizer.

> The following commands must be run in the same directory used to generate the SSL certificates during the Generating TLS Certificates section, or give path to *.pem files.

Generate a kubeconfig file for each worker node:

```console
for k in kube-artisan-0 kube-artisan-1 kube-artisan-2 kube-minion-0 kube-minion-1 kube-minion-2; do
  kubectl config set-cluster metal-kubes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${k}.kubeconfig

  kubectl config set-credentials system:node:${k} \
    --client-certificate=${k}.pem \
    --client-key=${k}-key.pem \
    --embed-certs=true \
    --kubeconfig=${k}.kubeconfig

  kubectl config set-context default \
    --cluster=metal-kubes \
    --user=system:node:${k} \
    --kubeconfig=${k}.kubeconfig

  kubectl config use-context default --kubeconfig=${k}.kubeconfig
done

kube-artisan-0.kubeconfig
kube-artisan-1.kubeconfig
kube-artisan-2.kubeconfig
kube-minion-0.kubeconfig
kube-minion-1.kubeconfig
kube-minion-2.kubeconfig
```

The kube-proxy Kubernetes Configuration File

Generate a kubeconfig file for the kube-proxy service:

```console
$ kubectl config set-cluster metal-kubes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

$ kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

$ kubectl config set-context default \
    --cluster=metal-kubes \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

kube-proxy.kubeconfig
```

> Save kube-proxy.kubeconfig as it will be needed anytime you want to add new worker node

The kube-controller-manager Kubernetes Configuration File

Generate a kubeconfig file for the kube-controller-manager service:

```console
$ kubectl config set-cluster metal-kubes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config set-context default \
    --cluster=metal-kubes \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig

kube-controller-manager.kubeconfig
```

The kube-scheduler Kubernetes Configuration File

Generate a kubeconfig file for the kube-scheduler service:

```console
$ kubectl config set-cluster metal-kubes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-context default \
    --cluster=metal-kubes \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig

kube-scheduler.kubeconfig
```

The admin Kubernetes Configuration File

Generate a kubeconfig file for the admin user:

```console
$ kubectl config set-cluster metal-kubes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

$ kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

$ kubectl config set-context default \
    --cluster=metal-kubes \
    --user=admin \
    --kubeconfig=admin.kubeconfig

$ kubectl config use-context default --kubeconfig=admin.kubeconfig

admin.kubeconfig
```

Distribute the Kubernetes Configuration Files

Copy the appropriate kubelet and kube-proxy kubeconfig files to each worker instance:

```console
$ for k in kube-artisan-0 kube-artisan-1 kube-artisan-2 kube-minion-0 kube-minion-1 kube-minion-2; do
  scp ${k}.kubeconfig kube-proxy.kubeconfig ${k}:~/
done
```

Copy the appropriate kube-controller-manager and kube-scheduler kubeconfig files to each controller instance:

```console
$ for k in kube-maestro-0 kube-maestro-1 kube-maestro-2; do
  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${k}:~/
done
```
## Generating the Data Encryption Config and Key

Kubernetes stores a variety of data including cluster state, application configurations, and secrets.
Kubernetes supports the ability to encrypt cluster data at rest.

In this section you will generate an encryption key and an encryption config suitable for encrypting Kubernetes Secrets.

The Encryption Key

Generate an encryption key:

```console
$ ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

The Encryption Config File

Create the encryption-config.yaml encryption config file:

```console
$ cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

Copy the encryption-config.yaml encryption config file to each controller instance:

```console
$ for k in kube-maestro-0 kube-maestro-1 kube-maestro-2; do
  scp encryption-config.yaml ${k}:~/
done
```
## Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in etcd.
In this section you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

Prerequisites

The commands in this section must be run on each controller instance: kube-maestro-0, kube-maestro-1, and kube-maestro-2.
Login to each controller instance and,

> ssh kube-maestro-0

Bootstrapping an etcd Cluster Member

Download and Install the etcd Binaries

Download the official etcd release binaries from the etcd GitHub project:

```console
$ wget -qc --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.10/etcd-v3.4.10-linux-amd64.tar.gz"
```

Extract and install the etcd server and the etcdctl command line utility:

```console
$ gzip -dc etcd-v3.4.10-linux-amd64.tar.gz |tar -xvf -
$ sudo mv etcd-v3.4.10-linux-amd64/etcd* /usr/local/bin/
```

Configure the etcd Server

```console
$ sudo mkdir -p /etc/etcd /var/lib/etcd
$ sudo chmod 700 /var/lib/etcd
$ sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers.
Retrieve the internal IP address for the current node:

> INT_IP=$(hostname -I|awk '{print $1}')

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current node:

> ETCD_NAME=$(hostname -s)

Create the etcd.service systemd unit file:

```console
$ cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INT_IP}:2380 \\
  --listen-peer-urls https://${INT_IP}:2380 \\
  --listen-client-urls https://${INT_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INT_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster kube-maestro-0=https://${KUBE_MAESTRO_0}:2380,kube-maestro-1=https://${KUBE_MAESTRO_1}:2380,kube-maestro-2=https://${KUBE_MAESTRO_2}:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Start the etcd Server

```console
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
```

> Remember to run the above commands on each controller node: kube-maestro-0, kube-maestro-1, and kube-maestro-2.

List the etcd cluster members:

```console
$ sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem

453b323a60823483, started, kube-maestro-0, https://10.244.10.153:2380, https://10.244.10.153:2379, false
67afae84e7289022, started, kube-maestro-2, https://10.244.10.155:2380, https://10.244.10.155:2379, false
6e42aa71fcfdb88d, started, kube-maestro-1, https://10.244.10.154:2380, https://10.244.10.154:2379, false
```
## Bootstrapping the Kubernetes Control Plane

In this section you will bootstrap the Kubernetes control plane across three nodes and configure it for high availability.
You will also create an external load balancer that exposes the Kubernetes API Servers to remote clients.
The following components will be installed on each node: Kubernetes API Server, Scheduler, and Controller Manager.

Prerequisites

The commands in this section must be run on each controller instance: kube-maestro-0, kube-maestro-1, and kube-maestro-2.
Login to each controller instance using the ssh command. Example:

ssh kube-maestro-0

#### Provision the Kubernetes Control Plane

Create the Kubernetes configuration directory:

```console
sudo mkdir -p /etc/kubernetes/config
```

Download and Install the Kubernetes Controller Binaries

```console
$  wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.20.2/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.20.2/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.20.2/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.20.2/bin/linux/amd64/kubectl"
```

Install the Kubernetes binaries:

```console
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

Configure the Kubernetes API Server

```console
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
```

The instance internal IP address will be used to advertise the API Server to members of the cluster.
Retrieve the internal IP address for the current node:

> INT_IP=$(hostname -I|awk '{print $1}')

Create the kube-apiserver.service systemd unit file:

```console
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INT_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://${KUBE_MAESTRO_0}:2379,https://${KUBE_MAESTRO_1}:2379,https://${KUBE_MAESTRO_2}:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.50.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Controller Manager

Move the kube-controller-manager kubeconfig into place:

> sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/

Create the kube-controller-manager.service systemd unit file:

```console
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.100.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.50.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

#### Configure the Kubernetes Scheduler

Move the kube-scheduler kubeconfig into place:

> sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/

Create the kube-scheduler.yaml configuration file:

```console
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

Create the kube-scheduler.service systemd unit file:

```console
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Start the Controller Services

```console
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.

Enable HTTP Health Checks

An External Load Balancer will be used to distribute traffic across the three API servers
and allow each API server to terminate TLS connections and validate client certificates.
The network load balancer only supports HTTP health checks which means the HTTPS endpoint exposed by the API server cannot be used.
As a workaround the nginx webserver can be used to proxy HTTP health checks.
In this section nginx will be installed and configured to accept HTTP health checks on port 80
and proxy the connections to the API server on https://127.0.0.1:6443/healthz.

> The /healthz API server endpoint does not require authentication by default.

##### Install a basic web server to handle HTTP health checks:

```console
$ sudo apt update
$ sudo apt install -y nginx
```

```console
cat > kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF

$ sudo mv kubernetes.default.svc.cluster.local \
    /etc/nginx/sites-available/kubernetes.default.svc.cluster.local

$ sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/

$ sudo systemctl restart nginx
$ sudo systemctl enable nginx
```

```console
$ kubectl get componentstatuses --kubeconfig admin.kubeconfig
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
controller-manager   Healthy   ok
```

Test the nginx HTTP health check proxy:

```console
$ curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz

HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Sun, 21 Mar 2021 14:32:25 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: keep-alive
Cache-Control: no-cache, private
X-Content-Type-Options: nosniff
X-Kubernetes-Pf-Flowschema-Uid: 5c1cad28-d0bc-4b40-8208-eb354f102441
X-Kubernetes-Pf-Prioritylevel-Uid: c6aefb90-728b-4b2b-9209-aafe63d9a931
```

    Remember to run the above commands on each controller node: kube-maestro-0, kube-maestro-1, kube-maestro-2.

> It is important that you maintain same version for all Kubernetes components, in my case I am using 1.20.2
## RBAC for Kubelet Authorization

In this section you will configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node.
Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods.

    This tutorial sets the Kubelet --authorization-mode flag to Webhook. Webhook mode uses the SubjectAccessReview API to determine authorization.

The commands in this section will effect the entire cluster and only need to be run once from one of the controller nodes.

ssh kube-maestro-0

Create the system:kube-apiserver-to-kubelet ClusterRole with permissions to access the Kubelet API and perform most common tasks associated with managing pods:

```console
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

The Kubernetes API Server authenticates to the Kubelet as the kubernetes user using the client certificate as defined by the --kubelet-client-certificate flag.

Bind the system:kube-apiserver-to-kubelet ClusterRole to the kubernetes user:

```console
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## Bootstrapping the Kubernetes Worker Nodes (artisan/minion)

In this section you will bootstrap six Kubernetes worker nodes.
The following components will be installed on each node: runc, container networking plugins, containerd, kubelet, kube-proxy.

Prerequisites

The commands in this section must be run on each worker instance:  kube-artisan-0 kube-artisan-1 kube-artisan-2 kube-minion-0 kube-minion-1 kube-minion-2.

Running commands in parallel with multiple terminals or tabed terminals 
> disown is your friend in case you have to shutdown terminal

### Provisioning a Kubernetes Worker Node

Install the OS dependencies:

```console
$ sudo apt update
$ sudo apt -y install socat conntrack ipset
```

The socat binary enables support for the kubectl port-forward command.

Disable Swap

```console
$ sudo rm /swap.img
#back up /etc/fstab:
$ sudo cp /etc/fstab /etc/fstab.bak
$ sudo sed -i '/\/swap.img/d' /etc/fstab
$ sudo swapoff -a -v
```

By default the kubelet will fail to start if swap is enabled.
It is recommended that swap be disabled to ensure Kubernetes can provide proper resource allocation and quality of service.

Verify if swap is enabled:

```console
$ sudo swapon --show
```

If output is empty then swap is not enabled. If swap is enabled run the following command to disable swap immediately:

```console
$ sudo swapoff -a
```

> To ensure swap remains off after reboot consult your Linux distro documentation.

Download and Install Worker Binaries

```console
$ wget -qc --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.20.0/crictl-v1.20.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc91/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz \
  https://github.com/containerd/containerd/releases/download/v1.4.3/containerd-1.4.3-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.20.2/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.20.2/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.20.2/bin/linux/amd64/kubelet
```

Create the installation directories:

```console
$ sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Install the worker binaries:

```console
$ mkdir containerd
$ tar -xvf crictl-v1.20.0-linux-amd64.tar.gz
$ tar -xvf containerd-1.4.3-linux-amd64.tar.gz -C containerd
$ sudo tar -xvf cni-plugins-linux-amd64-v0.9.1.tgz -C /opt/cni/bin/
$ sudo mv runc.amd64 runc
$ chmod +x crictl kubectl kube-proxy kubelet runc
$ sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
$ sudo mv containerd/bin/* /bin/
```


### Configure CNI Networking

Retrieve the Pod CIDR range for the current node:

> POD_CIDR=10.100.0.0/16

Create the bridge network configuration file:

```console
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.4.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

> cniVersion can be tricky, 0.4.0 is compatible with 0.9.1

Create the loopback network configuration file:

```console
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.4.0",
    "name": "lo",
    "type": "loopback"
}
EOF
```

Configure containerd

Create the containerd configuration file:

```console
$ sudo mkdir -p /etc/containerd/
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF
```

> containerd not very friendly with docker (like you want to use docker registry for your container images)

Create the containerd.service systemd unit file:

```console
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

Configure the Kubelet

```console
$ sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
$ sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
$ sudo mv ca.pem /var/lib/kubernetes/
```

Create the kubelet-config.yaml configuration file:

```console
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.50.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

The resolvConf configuration is used to avoid loops when using CoreDNS for service discovery on systems running systemd-resolved.

Create the kubelet.service systemd unit file:

```console
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Configure the Kubernetes Proxy

```console
$ sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Create the kube-proxy-config.yaml configuration file:

```console
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.100.0.0/16"
EOF
```

Create the kube-proxy.service systemd unit file:

```console
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Start the Worker Services # Make sure you copied *.pem files in /var/lib/kubernetes

```console
$ sudo systemctl daemon-reload
$ sudo systemctl enable containerd kubelet kube-proxy
$ sudo systemctl start containerd kubelet kube-proxy
```

> Remember to run the above commands on each worker node: kube-artisan-0 kube-artisan-1 kube-artisan-2 kube-minion-0 kube-minion-1 kube-minion-2


Verification

Run the following commands from the auxiliary node.

List the registered Kubernetes nodes:

```console
$ kubectl get nodes --kubeconfig admin.kubeconfig
NAME             STATUS   ROLES    AGE   VERSION
kube-artisan-0   Ready    <none>   33s    v1.20.2
kube-artisan-1   Ready    <none>   33s    v1.20.2
kube-artisan-2   Ready    <none>   33s    v1.20.2
kube-minion-0    Ready    <none>   33s    v1.20.2
kube-minion-1    Ready    <none>   33s    v1.20.2
kube-minion-2    Ready    <none>   33s    v1.20.2
```

## Configuring kubectl for Remote Access

In this section you will generate a kubeconfig file for the kubectl command line utility based on the admin user credentials.

> Run the commands in this section from the same directory used to generate the admin client certificates.

The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Generate a kubeconfig file suitable for authenticating as the admin user:

```console
  KUBERNETES_PUBLIC_ADDRESS=10.244.10.156

  kubectl config set-cluster metal-kubes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

  kubectl config set-context metal-kubes \
    --cluster=metal-kubes \
    --user=admin

  kubectl config use-context metal-kubes
```

Verification

Check the health of the remote Kubernetes cluster:

```console
$ kubectl get componentstatuses --kubeconfig admin.kubeconfig
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
controller-manager   Healthy   ok
```

List the nodes in the remote Kubernetes cluster:

```console
$ k get no
NAME             STATUS   ROLES    AGE   VERSION
kube-artisan-0   Ready    <none>   3m20s    v1.20.2
kube-artisan-1   Ready    <none>   3m20s    v1.20.2
kube-artisan-2   Ready    <none>   3m20s    v1.20.2
kube-minion-0    Ready    <none>   3m20s    v1.20.2
kube-minion-1    Ready    <none>   3m20s    v1.20.2
kube-minion-2    Ready    <none>   3m20s    v1.20.2
```

Make a HTTP request for the Kubernetes version info (from a worker node):
```console
$ curl --cacert /var/lib/kubernetes/ca.pem https://10.244.10.154:6443/version
{
  "major": "1",
  "minor": "20",
  "gitVersion": "v1.20.2",
  "gitCommit": "faecb196815e248d3ecfb03c680a4507229c2a56",
  "gitTreeState": "clean",
  "buildDate": "2021-01-13T13:20:00Z",
  "goVersion": "go1.15.5",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```
## Deploying the DNS Cluster Add-on

In this section you will deploy the DNS add-on which provides DNS based service discovery, backed by CoreDNS,
to applications running inside the Kubernetes cluster.

The DNS Cluster Add-on

Deploy the coredns cluster add-on:

```console
$ kubectl apply -f coredns-1.7.0.yaml
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```

List the pods created by the kube-dns deployment:

```console
$ kubectl get pods -l k8s-app=kube-dns -n kube-system
NAME                       READY   STATUS    RESTARTS   AGE
coredns-75774bf574-9gn2c   1/1     Running   0          20d
coredns-75774bf574-dnv8h   1/1     Running   0          20d
```


Create a busybox deployment:

```console
$ kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```

List the pod created by the busybox deployment:

```console
$ kubectl get pods -l run=busybox
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

Retrieve the full name of the busybox pod:

```console
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Execute a DNS lookup for the kubernetes service inside the busybox pod:

```console
$ kubectl exec -ti $POD_NAME -- nslookup kubernetes
Server:    10.50.0.10
Address 1: 10.50.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.50.0.1 kubernetes.default.svc.cluster.local
```
## Smoke Test

In this section you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.
Data Encryption

In this section you will verify the ability to encrypt secret data at rest.

Create a generic secret:

```console
$ kubectl create secret generic metal-kubes --from-literal="mykey=mydata"
secret/metal-kubes created
```

```console
$ kubectl describe secrets metal-kubes
Name:         metal-kubes
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
mykey:  6 bytes
```

Print a hexdump of the metal-kubes secret stored in etcd:

```console
$ ssh kube-maestro-0 \
  "sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/metal-kubes | hexdump -C"

00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6d 65 74 61 6c 2d  |s/default/metal-|
00000020  6b 75 62 65 73 0a 6b 38  73 3a 65 6e 63 3a 61 65  |kubes.k8s:enc:ae|
00000030  73 63 62 63 3a 76 31 3a  6b 65 79 31 3a d2 34 96  |scbc:v1:key1:.4.|
00000040  29 b8 10 78 f6 77 42 2f  c0 51 f0 23 e3 42 86 71  |)..x.wB/.Q.#.B.q|
00000050  59 ef 25 e1 94 3e c7 83  9f 6a 7f a8 26 ff 36 7e  |Y.%..>...j..&.6~|
00000060  6b a6 68 86 2f be d1 45  0f 3e ca 46 b6 1b 3f 36  |k.h./..E.>.F..?6|
00000070  d7 20 f5 dc 08 87 09 a0  b3 2c 19 83 d5 27 86 6e  |. .......,...'.n|
00000080  6d 6a 72 83 df fc 9c bc  cc 89 0b d7 0b bb 03 ed  |mjr.............|
00000090  96 36 7d 53 b7 6c 92 d8  e5 20 95 af b6 4d 5b 0f  |.6}S.l... ...M[.|
000000a0  3b c0 b8 37 3f 79 27 99  51 eb e1 ff 2c c5 f0 e5  |;..7?y'.Q...,...|
000000b0  4f d0 23 50 4b f5 c3 83  79 13 28 50 6e 7b 80 f9  |O.#PK...y.(Pn{..|
000000c0  1f 4a 69 ab ca 67 d5 54  ea 47 59 08 24 d3 c5 1a  |.Ji..g.T.GY.$...|
000000d0  40 51 d4 e3 48 ff 72 8c  a8 a8 98 cf be cd f7 02  |@Q..H.r.........|
000000e0  8c 65 b3 83 61 32 03 3a  55 20 e5 b1 33 9c 91 af  |.e..a2.:U ..3...|
000000f0  1d 7e 3c a5 e1 09 ed 55  e0 9e 4e 3c 46 3d f5 e2  |.~<....U..N<F=..|
00000100  c6 2e b8 bd a7 7a 18 cb  8b db fb 74 69 ff a1 55  |.....z.....ti..U|
00000110  62 49 43 77 5a b1 8a 06  97 07 a1 57 43 66 28 11  |bICwZ......WCf(.|
00000120  e7 61 d0 56 67 42 19 e1  cb 3d 7f df cb 66 10 e9  |.a.VgB...=...f..|
00000130  8d cf 9f b4 65 c5 f1 bb  73 03 a4 30 10 0a        |....e...s..0..|
0000013e
```

The etcd key should be prefixed with k8s:enc:aescbc:v1:key1, which indicates the aescbc provider was used to encrypt the data with the key1 encryption key.

Deployments

In this section you will verify the ability to create and manage Deployments.

Create a deployment for the nginx web server:

```console
$ kubectl create deployment nginx --image=nginx
```

List the pod created by the nginx deployment:

```console
$ kubectl get po -l app=nginx -o wide
NAME                     READY   STATUS              RESTARTS   AGE   IP       NODE            NOMINATED NODE   READINESS GATES
nginx-6799fc88d8-9ln76   0/1     ContainerCreating   0          93s   <none>   kube-minion-2   <none>           <none>
```

Port Forwarding

In this section you will verify the ability to access applications remotely using port forwarding.

Forward port 8080 on your local machine to port 80 of the nginx pod:

```console
$ kubectl port-forward nginx-6799fc88d8-9ln76 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```console
$ curl --head http://127.0.0.1:8080
HTTP/1.1 200 OK
Server: nginx/1.19.10
Date: Thu, 22 Apr 2021 02:50:56 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Apr 2021 15:13:59 GMT
Connection: keep-alive
ETag: "6075b537-264"
Accept-Ranges: bytes
```

```console
$ curl -ik localhost:8081
HTTP/1.1 200 OK
Server: nginx/1.19.10
Date: Thu, 22 Apr 2021 02:48:46 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Apr 2021 15:13:59 GMT
Connection: keep-alive
ETag: "6075b537-264"
Accept-Ranges: bytes

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Switch back to the previous terminal and stop the port forwarding to the nginx pod:

```console
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

#### Logs

In this section you will verify the ability to retrieve container logs.

Print the nginx pod logs:

```console
$ kubectl logs $POD_NAME
...
127.0.0.1 - - [18/Jul/2020:07:14:00 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.64.0" "-"
```

#### Exec

In this section you will verify the ability to execute commands in a container.

Print the nginx version by executing the nginx -v command in the nginx container:

```console
$ kubectl exec -it $POD_NAME -- nginx -v
nginx version: nginx/1.19.10
```

#### Services

In this section you will verify the ability to expose applications using a Service.

Expose the nginx deployment using a NodePort service:

```console
$ kubectl expose deployment nginx --port 80 --type NodePort
```

The LoadBalancer service type can not be used because your cluster is not configured with cloud provider integration.
Setting up cloud provider integration is out of scope for this tutorial.

Retrieve the node port assigned to the nginx service:

> NODE_PORT=$(kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}')

Retrieve the external IP address of a worker instance:

> EXT_IP=99.98.10.12

Make an HTTP request using the external IP address and the nginx node port:

```console
$ curl -I http://${EXT_IP}:${NODE_PORT}
HTTP/1.1 200 OK
Server: nginx/1.19.10
Date: Sat, 18 Jul 2020 07:16:41 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 07 Jul 2020 15:52:25 GMT
Connection: keep-alive
ETag: "5f049a39-264"
Accept-Ranges: bytes
```

## Add Worker Node

You need to create <node>-key.pem, <node>.pem, and <node>.kubeconfig
Prerequisites:  # created in "Provisioning a CA and Generating TLS Certificates"
```console
  ca.pem
  ca-key.pem
  ca-config.json
  kube-proxy.kubeconfig 
```

After spinning up a node add/run following,

Add node name and IP address to /etc/hosts of your Auxiliary machine

```console
$ sudo bash -c "echo '10.244.10.159 kube-artisan-4' >> /etc/hosts"
```

```console
$ knode=kube-artisan-4
$ cat > ${knode}-csr.json <<EOF
{
  "CN": "system:node:${knode}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Folsom",
      "O": "system:nodes",
      "OU": "Metal Kubes",
      "ST": "California"
    }
  ]
}
EOF

$ EXT_IP=99.98.10.12
$ INT_IP=$(ping -q -c 1 ${knode} |grep  -Po ' \(\K[\d.]+')

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${knode},${EXT_IP},${INT_IP} \
  -profile=kubernetes \
  ${knode}-csr.json | cfssljson -bare ${knode}
done

$ scp ca.pem ${knode}-key.pem ${knode}.pem ${knode}:~/

$ kubectl config set-cluster metal-kubes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=${knode}.kubeconfig

$ kubectl config set-credentials system:node:${knode} \
  --client-certificate=${knode}.pem \
  --client-key=${knode}-key.pem \
  --embed-certs=true \
  --kubeconfig=${knode}.kubeconfig

$ kubectl config set-context default \
  --cluster=metal-kubes \
  --user=system:node:${knode} \
  --kubeconfig=${knode}.kubeconfig

$ kubectl config use-context default --kubeconfig=${knode}.kubeconfig

$ scp ${knode}.kubeconfig kube-proxy.kubeconfig ${knode}:~/
```

> kube-proxy.kubeconfig was created in "Generating Kubernetes Configuration Files for Authentication"

Now follow the steps in "Bootstrapping the Kubernetes Worker Nodes (artisan/minion)"
## Cleaning Up

Best part about metal setup is no recurring charges, just shutdown your machines.
Nothing much to clean up, as this is bare metal setup (wipe HD and start fresh)

Use virsh destroy to remove VMs (including Nginx External Load Balancer) and corresponding storage

$ virsh destroy <node_name> --graceful

## Notes

Five key components on each Control Plane node,
```console
  kube-apiserver : Passive service thru which every event passes
  kube-scheduler: Dictates which Pod goes to which node
  kubelet: Actually create the containers (Worker/Master)
  kube-proxy: Responsible for iptables functionality on each node (Worker/Master)
  kube-controller-manager: Controls numerous Conrol services listed below.
```

Sample set of controllers running kube-controller-manager:
```console
       attachdetach-controller
       certificate-controller
       clusterrole-aggregation-controller
       cronjob-controller
       daemon-set-controller
       deployment-controller
       disruption-controller
       endpoint-controller
       endpointslice-controller
       endpointslicemirroring-controller
       expand-controller
       generic-garbage-collector
       horizontal-pod-autoscaler
       job-controller
       namespace-controller
       node-controller
       persistent-volume-binder
       pod-garbage-collector
       pv-protection-controller
       pvc-protection-controller
       replicaset-controller
       replication-controller
       resourcequota-controller
       root-ca-cert-publisher
       service-account-controller
       service-controller
       statefulset-controller
       ttl-controller
```

Service is a abstract object item that always exists, in 3 main forms,
```console
  ClusterIP - Internal IP Address
  NodePort - Large Port number that can addressed directly (> 30000)
  LoadBalancer - Only exists in Cloud Kubernetes
```

All containers can communicate with all other containers without NAT, within a cluster.

## Cluster info

```console
> file /lib/systemd/systemd   (OS 32bit or 64bit)

$ k get sa -A
NAMESPACE         NAME                                 SECRETS   AGE
default           default                              1         31d
ingress-nginx     default                              1         19d
ingress-nginx     ingress-nginx                        1         19d
ingress-nginx     ingress-nginx-admission              1         19d
kube-node-lease   default                              1         31d
kube-public       default                              1         31d
kube-system       attachdetach-controller              1         31d
kube-system       certificate-controller               1         31d
kube-system       clusterrole-aggregation-controller   1         31d
kube-system       coredns                              1         28d
kube-system       cronjob-controller                   1         31d
kube-system       daemon-set-controller                1         31d
kube-system       default                              1         31d
kube-system       deployment-controller                1         31d
kube-system       disruption-controller                1         31d
kube-system       endpoint-controller                  1         31d
kube-system       endpointslice-controller             1         31d
kube-system       endpointslicemirroring-controller    1         31d
kube-system       expand-controller                    1         31d
kube-system       generic-garbage-collector            1         31d
kube-system       horizontal-pod-autoscaler            1         31d
kube-system       job-controller                       1         31d
kube-system       kube-router                          1         28d
kube-system       namespace-controller                 1         31d
kube-system       node-controller                      1         31d
kube-system       persistent-volume-binder             1         31d
kube-system       pod-garbage-collector                1         31d
kube-system       pv-protection-controller             1         31d
kube-system       pvc-protection-controller            1         31d
kube-system       replicaset-controller                1         31d
kube-system       replication-controller               1         31d
kube-system       resourcequota-controller             1         31d
kube-system       root-ca-cert-publisher               1         31d
kube-system       service-account-controller           1         31d
kube-system       service-controller                   1         31d
kube-system       statefulset-controller               1         31d
kube-system       ttl-controller                       1         31d
```
---
Kubernetes Running Processes on a Controller node
```console
systemctl status | less
...
   +-kube-apiserver.service
   ¦ +-16384 /usr/local/bin/kube-apiserver --advertise-address=10.244.10.153 --allow-privileged=true --apiserver-count=3 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/var/log/audit.log --authorization-mode=Node,RBAC --bind-address=0.0.0.0 --client-ca-file=/var/lib/kubernetes/ca.pem --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota --etcd-cafile=/var/lib/kubernetes/ca.pem --etcd-certfile=/var/lib/kubernetes/kubernetes.pem --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem --etcd-servers=https://10.244.10.153:2379,https://10.244.10.154:2379,https://10.244.10.155:2379 --event-ttl=1h --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem --kubelet-https=true --runtime-config=api/all=true --service-account-key-file=/var/lib/kubernetes/service-account.pem --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-cluster-ip-range=10.50.0.0/24 --service-node-port-range=30000-32767 --tls-cert-file=/var/lib/kubernetes/kubernetes.pem --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem --v=2

   +-kube-scheduler.service
   ¦ +-616 /usr/local/bin/kube-scheduler --config=/etc/kubernetes/config/kube-scheduler.yaml --v=2

   +-kube-controller-manager.service
   ¦ +-19736 /usr/local/bin/kube-controller-manager --bind-address=0.0.0.0 --cluster-cidr=10.100.0.0/16 --allocate-node-cidrs=true --node-cidr-mask-size=24 --cluster-name=kubernetes --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig --leader-elect=true --cert-dir=/var/lib/kubernetes --root-ca-file=/var/lib/kubernetes/ca.pem --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem --service-cluster-ip-range=10.50.0.0/24 --use-service-account-credentials=true --v=2

   +-etcd.service
   ¦ +-15676 /usr/local/bin/etcd --name kube-maestro-0 --cert-file=/etc/etcd/kubernetes.pem --key-file=/etc/etcd/kubernetes-key.pem --peer-cert-file=/etc/etcd/kubernetes.pem --peer-key-file=/etc/etcd/kubernetes-key.pem --trusted-ca-file=/etc/etcd/ca.pem --peer-trusted-ca-file=/etc/etcd/ca.pem --peer-client-cert-auth --client-cert-auth --initial-advertise-peer-urls https://10.244.10.153:2380 --listen-peer-urls https://10.244.10.153:2380 --listen-client-urlshttps://10.244.10.153:2379,https://127.0.0.1:2379 --advertise-client-urls https://10.244.10.153:2379 --initial-cluster-token etcd-cluster-0 --initial-cluster kube-maestro-0=https://10.244.10.153:2380,kube-maestro-1=https://10.244.10.154:2380,kube-maestro-2=https://10.244.10.155:2380 --initial-cluster-state new --data-dir=/var/lib/etcd
```

### Kubernetes Abbreviations:

```console
    -A      All namespaces
    sa      serviceaccounts
    ing     ingresses
    cm      configmaps
    deploy  deployments
    ep      endpoints
    ev      events
    cs      componentstatuses
    rs      replicasets
    svc     services
    ds      daemonsets
    ns      namespaces
    no      nodes
    pvc     persistentvolumeclaims
    pv      persistentvolumes
    po      pods
    rc      replicationcontrollers
    pdb     poddisruptionbudgets
    psp     podsecuritypolicies
    quota   resourcequotas
    csr     certificatesigningrequests
    hpa     horizontalpodautoscalers
    limits  limitranges
```
_That's all folks_
---

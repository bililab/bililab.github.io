---
title: k8s install
date: 2020-02-11 14:16:43
tags:
---
# test-env
## kubernetes

### prepare
```bash

yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp

# run as root on each node
swapoff -a
systemctl disable firewalld
systemctl stop firewalld
iptables -F
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf

modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4
lsmod | grep ip_vs

```

### cfsssl (node1)

```bash
mkdir -p /opt/local/cfssl
cd /opt/local/cfssl
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
mv cfssl_linux-amd64 cfssl
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
mv cfssljson_linux-amd64 cfssljson
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 cfssl-certinfo
chmod +x *
```

## create certs (node1)

+ create certs
```bash
mkdir /opt/ssl
cd /opt/ssl
cat <<EOF > config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
cat <<EOF > csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Chongqing",
      "L": "Chongqing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
/opt/local/cfssl/cfssl gencert -initca csr.json | /opt/local/cfssl/cfssljson -bare ca

mkdir -p /etc/kubernetes/ssl

cp *.pem /etc/kubernetes/ssl
cp ca.csr /etc/kubernetes/ssl

ssh root@node2 "mkdir -p /etc/kubernetes/ssl/"
ssh root@node3 "mkdir -p /etc/kubernetes/ssl/"

scp *.pem *.csr root@node2:/etc/kubernetes/ssl/
scp *.pem *.csr root@node3:/etc/kubernetes/ssl/

# etcd
cd /opt/ssl
cat <<EOF > etcd-csr.json
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "172.17.3.108",
    "172.17.3.140",
    "172.17.3.223",
    "172.17.3.224",
    "172.17.3.225",
    "172.17.3.226",
    "172.17.3.227",
    "172.17.3.228",
    "172.17.3.229",
    "172.17.3.230"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Chongqing",
      "L": "Chongqing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
/opt/local/cfssl/cfssl gencert -ca=/opt/ssl/ca.pem \
    -ca-key=/opt/ssl/ca-key.pem \
    -config=/opt/ssl/config.json \
    -profile=kubernetes etcd-csr.json | /opt/local/cfssl/cfssljson -bare etcd

cp etcd*.pem /etc/kubernetes/ssl/
chmod 644 /etc/kubernetes/ssl/etcd-key.pem

scp etcd*.pem root@node2:/etc/kubernetes/ssl/
ssh root@node2 "chmod 644 /etc/kubernetes/ssl/etcd-key.pem"

scp etcd*.pem root@node3:/etc/kubernetes/ssl/
ssh root@node3 "chmod 644 /etc/kubernetes/ssl/etcd-key.pem"

# admin
cd /opt/ssl
cat <<EOF > admin-csr.json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Chongqing",
      "L": "Chongqing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

/opt/local/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
    -ca-key=/etc/kubernetes/ssl/ca-key.pem \
    -config=/opt/ssl/config.json \
    -profile=kubernetes admin-csr.json | /opt/local/cfssl/cfssljson -bare admin

cp admin*.pem /etc/kubernetes/ssl/
scp admin*.pem root@node2:/etc/kubernetes/ssl/
scp admin*.pem root@node3:/etc/kubernetes/ssl/

# kuernetes 证书
cd /opt/ssl
cat <<EOF > kubernetes-csr.json
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "172.17.3.108",
    "172.17.3.140",
    "172.17.3.223",
    "10.254.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Chongqing",
      "L": "Chongqing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

## 这里 hosts 字段中 三个 IP 分别为 127.0.0.1 本机， 172.17.3.108 和 172.17.3.140 为 Master 的IP，多个Master需要写多个。  10.254.0.1 为 kubernetes SVC 的 IP， 一般是 部署网络的第一个IP , 如: 10.254.0.1 ， 在启动完成后，我们使用   kubectl get svc ， 就可以查看到

/opt/local/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
    -ca-key=/etc/kubernetes/ssl/ca-key.pem \
    -config=/opt/ssl/config.json \
    -profile=kubernetes kubernetes-csr.json | /opt/local/cfssl/cfssljson -bare kubernetes

cp kubernetes*.pem /etc/kubernetes/ssl/

scp kubernetes*.pem root@node2:/etc/kubernetes/ssl/

scp kubernetes*.pem root@node3:/etc/kubernetes/ssl/

# kube-proxy
cd /opt/ssl
cat <<EOF > kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Chongqing",
      "L": "Chongqing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

/opt/local/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/opt/ssl/config.json \
  -profile=kubernetes  kube-proxy-csr.json | /opt/local/cfssl/cfssljson -bare kube-proxy

cp kube-proxy* /etc/kubernetes/ssl/
scp kube-proxy* root@node2:/etc/kubernetes/ssl/
scp kube-proxy* root@node3:/etc/kubernetes/ssl/
```

### docker (each)
```bash
yum -y install docker
```

### etcd (each)
```bash
mkdir -p /tmp/k8s-deps; cd /tmp/k8s-deps; wget https://github.com/coreos/etcd/releases/download/v3.3.8/etcd-v3.3.8-linux-amd64.tar.gz
ssh root@node2 "mkdir -p /tmp/k8s-deps; cd /tmp/k8s-deps; wget https://github.com/coreos/etcd/releases/download/v3.3.8/etcd-v3.3.8-linux-amd64.tar.gz"
ssh root@node3 "mkdir -p /tmp/k8s-deps; cd /tmp/k8s-deps; wget https://github.com/coreos/etcd/releases/download/v3.3.8/etcd-v3.3.8-linux-amd64.tar.gz"

# etcd
cd /tmp/k8s-deps; tar zxf etcd-v3.3.8-linux-amd64.tar.gz; cd etcd-v3.3.8-linux-amd64; mv etcd  etcdctl /usr/bin/

ssh root@node2 "cd /tmp/k8s-deps; tar zxf etcd-v3.3.8-linux-amd64.tar.gz; cd etcd-v3.3.8-linux-amd64; mv etcd  etcdctl /usr/bin/"
ssh root@node3 "cd /tmp/k8s-deps; tar zxf etcd-v3.3.8-linux-amd64.tar.gz; cd etcd-v3.3.8-linux-amd64; mv etcd  etcdctl /usr/bin/"

# etcd-1
useradd etcd || true; mkdir -p /opt/etcd; chown -R etcd:etcd /opt/etcd
# etcd-2
ssh root@node2 "useradd etcd || true; mkdir -p /opt/etcd; chown -R etcd:etcd /opt/etcd"
# etcd-3
ssh root@node3 "useradd etcd || true; mkdir -p /opt/etcd; chown -R etcd:etcd /opt/etcd"

cat <<EOF > /etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/opt/etcd/
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/usr/bin/etcd \\
  --name=etcd1 \\
  --cert-file=/etc/kubernetes/ssl/etcd.pem \\
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \\
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \\
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --initial-advertise-peer-urls=https://172.17.3.108:2380 \\
  --listen-peer-urls=https://172.17.3.108:2380 \\
  --listen-client-urls=https://172.17.3.108:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://172.17.3.108:2379 \\
  --initial-cluster-token=k8s-etcd-cluster \\
  --initial-cluster=etcd1=https://172.17.3.108:2380,etcd2=https://172.17.3.140:2380,etcd3=https://172.17.3.223:2380 \\
  --initial-cluster-state=new \\
  --data-dir=/opt/etcd/
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

cd /tmp/k8s-deps
cat <<EOF > etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/opt/etcd/
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/usr/bin/etcd \\
  --name=etcd2 \\
  --cert-file=/etc/kubernetes/ssl/etcd.pem \\
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \\
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \\
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --initial-advertise-peer-urls=https://172.17.3.140:2380 \\
  --listen-peer-urls=https://172.17.3.140:2380 \\
  --listen-client-urls=https://172.17.3.140:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://172.17.3.140:2379 \\
  --initial-cluster-token=k8s-etcd-cluster \\
  --initial-cluster=etcd1=https://172.17.3.108:2380,etcd2=https://172.17.3.140:2380,etcd3=https://172.17.3.223:2380 \\
  --initial-cluster-state=new \\
  --data-dir=/opt/etcd/
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
scp etcd.service root@node2:/etc/systemd/system/

cat <<EOF > etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/opt/etcd/
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/usr/bin/etcd \\
  --name=etcd3 \\
  --cert-file=/etc/kubernetes/ssl/etcd.pem \\
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \\
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \\
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --initial-advertise-peer-urls=https://172.17.3.223:2380 \\
  --listen-peer-urls=https://172.17.3.223:2380 \\
  --listen-client-urls=https://172.17.3.223:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://172.17.3.223:2379 \\
  --initial-cluster-token=k8s-etcd-cluster \\
  --initial-cluster=etcd1=https://172.17.3.108:2380,etcd2=https://172.17.3.140:2380,etcd3=https://172.17.3.223:2380 \\
  --initial-cluster-state=new \\
  --data-dir=/opt/etcd/
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
scp etcd.service root@node3:/etc/systemd/system/

echo "systemctl daemon-reload; systemctl enable etcd; systemctl start etcd; systemctl status etcd" > /tmp/init-etcd.sh && cd /tmp && nohup sh init-etcd.sh 2>&1 &

ssh root@node2 "echo \"systemctl daemon-reload; systemctl enable etcd; systemctl start etcd; systemctl status etcd\" > /tmp/init-etcd.sh && cd /tmp && nohup sh init-etcd.sh 2>&1 &"

ssh root@node3 "echo \"systemctl daemon-reload; systemctl enable etcd; systemctl start etcd; systemctl status etcd\" > /tmp/init-etcd.sh && cd /tmp && nohup sh init-etcd.sh 2>&1 &"
```

### test etcd
```bash
etcdctl --endpoints=https://172.17.3.108:2379,https://172.17.3.140:2379,https://172.17.3.223:2379\
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        cluster-health
```

### master and node
```bash
cd /tmp && rm -rf kubernetes
wget https://dl.k8s.io/v1.11.1/kubernetes-server-linux-amd64.tar.gz
tar -xzvf kubernetes-server-linux-amd64.tar.gz; cd kubernetes
scp server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} root@node1:/usr/local/bin/
scp server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} root@node2:/usr/local/bin/
scp server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} root@node3:/usr/local/bin/

# api-server
TOKEN=`head -c 16 /dev/urandom | od -An -t x | tr -d ' '`
echo $TOKEN",kubelet-bootstrap,10001,\"system:bootstrappers\"" > /etc/kubernetes/token.csv
scp /etc/kubernetes/token.csv root@node2:/etc/kubernetes/
scp /etc/kubernetes/token.csv root@node3:/etc/kubernetes/

# 生成高级审核配置文件
cd /etc/kubernetes
cat <<EOF > audit-policy.yaml
# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1beta1
kind: Policy
rules:
- level: Metadata
EOF
scp audit-policy.yaml root@node2:/etc/kubernetes/
scp audit-policy.yaml root@node3:/etc/kubernetes/

# kube-apiserver.service
cat <<EOF > /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/kube-apiserver \\
  --admission-control=MutatingAdmissionWebhook,ValidatingAdmissionWebhook,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \\
  --advertise-address=172.17.3.108 \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/kubernetes/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --secure-port=6443 \\
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
  --etcd-certfile=/etc/kubernetes/ssl/etcd.pem \\
  --etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \\
  --etcd-servers=https://172.17.3.108:2379,https://172.17.3.140:2379,https://172.17.3.223:2379 \\
  --event-ttl=1h \\
  --kubelet-https=true \\
  --insecure-bind-address=127.0.0.1 \\
  --insecure-port=8080 \\
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --service-cluster-ip-range=10.254.0.0/18 \\
  --service-node-port-range=1-65535 \\
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \\
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \\
  --enable-bootstrap-token-auth \\
  --token-auth-file=/etc/kubernetes/token.csv \\
  --v=1
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# k8s 1.8 开始需要 添加 --authorization-mode=Node
# k8s 1.8 开始需要 添加 --admission-control=NodeRestriction
# k8s 1.8 开始需要 添加 --audit-policy-file=/etc/kubernetes/audit-policy.yaml

# 这里面要注意的是 --service-node-port-range=30000-32000
# 这个地方是 映射外部端口时 的端口范围，随机映射也在这个范围内映射，指定映射端口必须也在这个范围内。

systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver

## for node2
cat <<EOF > /tmp/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/kube-apiserver \\
  --admission-control=MutatingAdmissionWebhook,ValidatingAdmissionWebhook,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \\
  --advertise-address=172.17.3.140 \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/kubernetes/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --secure-port=6443 \\
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
  --etcd-certfile=/etc/kubernetes/ssl/etcd.pem \\
  --etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \\
  --etcd-servers=https://172.17.3.108:2379,https://172.17.3.140:2379,https://172.17.3.223:2379 \\
  --event-ttl=1h \\
  --kubelet-https=true \\
  --insecure-bind-address=127.0.0.1 \\
  --insecure-port=8080 \\
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --service-cluster-ip-range=10.254.0.0/18 \\
  --service-node-port-range=1-65535 \\
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \\
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \\
  --enable-bootstrap-token-auth \\
  --token-auth-file=/etc/kubernetes/token.csv \\
  --v=1
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

EOF

scp /tmp/kube-apiserver.service root@node2:/etc/systemd/system/
ssh root@node2 "systemctl daemon-reload; systemctl enable kube-apiserver; systemctl start kube-apiserver; systemctl status kube-apiserver"

## for node3
cat <<EOF > /tmp/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/kube-apiserver \\
  --admission-control=MutatingAdmissionWebhook,ValidatingAdmissionWebhook,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \\
  --advertise-address=172.17.3.223 \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/kubernetes/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --secure-port=6443 \\
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
  --etcd-certfile=/etc/kubernetes/ssl/etcd.pem \\
  --etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \\
  --etcd-servers=https://172.17.3.108:2379,https://172.17.3.140:2379,https://172.17.3.223:2379 \\
  --event-ttl=1h \\
  --kubelet-https=true \\
  --insecure-bind-address=127.0.0.1 \\
  --insecure-port=8080 \\
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --service-cluster-ip-range=10.254.0.0/18 \\
  --service-node-port-range=1-65535 \\
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \\
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \\
  --enable-bootstrap-token-auth \\
  --token-auth-file=/etc/kubernetes/token.csv \\
  --v=1
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

EOF

scp /tmp/kube-apiserver.service root@node3:/etc/systemd/system/
ssh root@node3 "systemctl daemon-reload; systemctl enable kube-apiserver; systemctl start kube-apiserver; systemctl status kube-apiserver"

# kube-controller-manager

cat <<EOF > /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --master=http://127.0.0.1:8080 \\
  --allocate-node-cidrs=true \\
  --service-cluster-ip-range=10.254.0.0/18 \\
  --cluster-cidr=10.254.64.0/18 \\
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \\
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --feature-gates=RotateKubeletServerCertificate=true \\
  --experimental-cluster-signing-duration=86700h0m0s \\
  --cluster-name=kubernetes \\
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --leader-elect=true \\
  --node-monitor-grace-period=40s \\
  --node-monitor-period=5s \\
  --pod-eviction-timeout=5m0s \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

EOF

systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager

scp /etc/systemd/system/kube-controller-manager.service root@node2:/etc/systemd/system/
ssh root@node2 "systemctl daemon-reload; systemctl enable kube-controller-manager; systemctl start kube-controller-manager; systemctl status kube-controller-manager"

# scheduler
cat <<EOF > /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --address=0.0.0.0 \\
  --master=http://127.0.0.1:8080 \\
  --leader-elect=true \\
  --v=1
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler

scp /etc/systemd/system/kube-scheduler.service root@node2:/etc/systemd/system/
ssh root@node2 "systemctl daemon-reload; systemctl enable kube-scheduler; systemctl start kube-scheduler; systemctl status kube-scheduler"

kubectl get componentstatuses

ssh root@node2 "kubectl get componentstatuses"

# kubelet
cd
# 先创建认证请求, user 为 master 中 token.csv 文件里配置的用户
# 只需创建一次就可以
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap

# 配置集群
kubectl config set-cluster kubernetes \
    --certificate-authority=/etc/kubernetes/ssl/ca.pem \
    --embed-certs=true \
    --server=https://172.17.3.108:6443 \
    --kubeconfig=bootstrap.kubeconfig

# 配置客户端认证
kubectl config set-credentials kubelet-bootstrap \
    --token=`cat /etc/kubernetes/token.csv | awk '{split($0,a,",");print a[1]}'` \
    --kubeconfig=bootstrap.kubeconfig

# 配置关联
kubectl config set-context default \
    --cluster=kubernetes \
    --user=kubelet-bootstrap \
    --kubeconfig=bootstrap.kubeconfig

# 配置默认关联
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

# 拷贝生成的 bootstrap.kubeconfig 文件
cp bootstrap.kubeconfig /etc/kubernetes/

scp bootstrap.kubeconfig root@node2:/etc/kubernetes/
scp bootstrap.kubeconfig root@node3:/etc/kubernetes/

# kubelet config
mkdir -p /var/lib/kubelet
ssh root@node2 "mkdir -p /var/lib/kubelet"
ssh root@node3 "mkdir -p /var/lib/kubelet"

cat <<EOF > /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \\
  --cgroup-driver=cgroupfs \\
  --hostname-override=node1 \\
  --pod-infra-container-image=k8s.gcr.io/pause-amd64:3.1 \\
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \\
  --feature-gates=RotateKubeletClientCertificate=true,RotateKubeletServerCertificate=true \\
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
  --cert-dir=/etc/kubernetes/ssl \\
  --cluster_dns=10.254.0.2 \\
  --cluster_domain=cluster.local. \\
  --hairpin-mode promiscuous-bridge \\
  --allow-privileged=true \\
  --fail-swap-on=false \\
  --serialize-image-pulls=false \\
  --logtostderr=true \\
  --max-pods=512 \\
  --runtime-cgroups=/systemd/system.slice \\
  --kubelet-cgroups=/systemd/system.slice \\
  --v=2

[Install]
WantedBy=multi-user.target

EOF

# 如上配置:
# node1            本机hostname
# 10.254.0.2       预分配的 dns 地址
# cluster.local.   为 kubernetes 集群的 domain
# k8s.gcr.io/pause-amd64:3.1  这个是 pod 的基础镜像

systemctl daemon-reload; systemctl enable kubelet; systemctl start kubelet; systemctl status kubelet

cat <<EOF > /tmp/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \\
  --cgroup-driver=cgroupfs \\
  --hostname-override=node2 \\
  --pod-infra-container-image=k8s.gcr.io/pause-amd64:3.1 \\
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \\
  --feature-gates=RotateKubeletClientCertificate=true,RotateKubeletServerCertificate=true \\
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
  --cert-dir=/etc/kubernetes/ssl \\
  --cluster_dns=10.254.0.2 \\
  --cluster_domain=cluster.local. \\
  --hairpin-mode promiscuous-bridge \\
  --allow-privileged=true \\
  --fail-swap-on=false \\
  --serialize-image-pulls=false \\
  --logtostderr=true \\
  --max-pods=512 \\
  --runtime-cgroups=/systemd/system.slice \\
  --kubelet-cgroups=/systemd/system.slice \\
  --v=2

[Install]
WantedBy=multi-user.target

EOF
scp /tmp/kubelet.service root@node2:/etc/systemd/system/
ssh root@node2 "systemctl daemon-reload; systemctl enable kubelet; systemctl start kubelet; systemctl status kubelet"

cat <<EOF > /tmp/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \\
  --cgroup-driver=cgroupfs \\
  --hostname-override=node3 \\
  --pod-infra-container-image=k8s.gcr.io/pause-amd64:3.1 \\
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \\
  --feature-gates=RotateKubeletClientCertificate=true,RotateKubeletServerCertificate=true \\
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
  --cert-dir=/etc/kubernetes/ssl \\
  --cluster_dns=10.254.0.2 \\
  --cluster_domain=cluster.local. \\
  --hairpin-mode promiscuous-bridge \\
  --allow-privileged=true \\
  --fail-swap-on=false \\
  --serialize-image-pulls=false \\
  --logtostderr=true \\
  --max-pods=512 \\
  --runtime-cgroups=/systemd/system.slice \\
  --kubelet-cgroups=/systemd/system.slice \\
  --v=2

[Install]
WantedBy=multi-user.target

EOF
scp /tmp/kubelet.service root@node3:/etc/systemd/system/
ssh root@node3 "systemctl daemon-reload; systemctl enable kubelet; systemctl start kubelet; systemctl status kubelet"

# find all csrs and approve them
kubectl get csr | grep -v NAME | awk '{print $1}' | xargs kubectl certificate approve 

# kube-proxy
# 配置3台主节点, server都传的本地127.0.0.1
# 配置集群
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-proxy.kubeconfig

# 配置客户端认证
kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

# 配置关联
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

# 配置默认关联
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

# 拷贝到需要的 node 端里
scp kube-proxy.kubeconfig root@node1:/etc/kubernetes/
scp kube-proxy.kubeconfig root@node2:/etc/kubernetes/
scp kube-proxy.kubeconfig root@node3:/etc/kubernetes/

mkdir -p /var/lib/kube-proxy
ssh root@node2 "mkdir -p /var/lib/kube-proxy"
ssh root@node3 "mkdir -p /var/lib/kube-proxy"

cat <<EOF > /etc/systemd/system/kube-proxy.service

[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \\
  --bind-address=172.17.3.108 \\
  --hostname-override=node1 \\
  --cluster-cidr=10.254.64.0/18 \\
  --masquerade-all \\
  --proxy-mode=ipvs \\
  --ipvs-min-sync-period=5s \\
  --ipvs-sync-period=5s \\
  --ipvs-scheduler=rr \\
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \\
  --logtostderr=true \\
  --v=1
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

EOF

cat <<EOF > /tmp/kube-proxy.service

[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \\
  --bind-address=172.17.3.140 \\
  --hostname-override=node2 \\
  --cluster-cidr=10.254.64.0/18 \\
  --masquerade-all \\
  --proxy-mode=ipvs \\
  --ipvs-min-sync-period=5s \\
  --ipvs-sync-period=5s \\
  --ipvs-scheduler=rr \\
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \\
  --logtostderr=true \\
  --v=1
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

EOF

scp /tmp/kube-proxy.service root@node2:/etc/systemd/system/

cat <<EOF > /tmp/kube-proxy.service

[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \\
  --bind-address=172.17.3.223 \\
  --hostname-override=node3 \\
  --cluster-cidr=10.254.64.0/18 \\
  --masquerade-all \\
  --proxy-mode=ipvs \\
  --ipvs-min-sync-period=5s \\
  --ipvs-sync-period=5s \\
  --ipvs-scheduler=rr \\
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \\
  --logtostderr=true \\
  --v=1
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

EOF

scp /tmp/kube-proxy.service root@node3:/etc/systemd/system/

ssh root@node1 "mkdir -p /var/lib/kube-proxy"
ssh root@node2 "mkdir -p /var/lib/kube-proxy"
ssh root@node3 "mkdir -p /var/lib/kube-proxy"

ssh root@node1 "systemctl daemon-reload; systemctl enable kube-proxy; systemctl start kube-proxy; systemctl status kube-proxy"
ssh root@node2 "systemctl daemon-reload; systemctl enable kube-proxy; systemctl start kube-proxy; systemctl status kube-proxy"
ssh root@node3 "systemctl daemon-reload; systemctl enable kube-proxy; systemctl start kube-proxy; systemctl status kube-proxy"


apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 172.17.3.223
clientConnection:
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
clusterCIDR: 10.254.64.0/18
healthzBindAddress: 172.17.3.223:10256
hostnameOverride: node3
kind: KubeProxyConfiguration
metricsBindAddress: 172.17.3.223:10249
mode: "ipvs"

```

### Calico
+ wget https://docs.projectcalico.org/v3.6/getting-started/kubernetes/installation/hosted/calico.yaml

```bash
# 注意修改如下选项:

# etcd 地址

  etcd_endpoints: "https://172.17.3.108:2379,https://172.17.3.140:2379,https://172.17.3.223:2379"
  
# etcd 证书路径
  # If you're using TLS enabled etcd uncomment the following.
  # You must also populate the Secret below with these files. 
    etcd_ca: "/calico-secrets/etcd-ca"  
    etcd_cert: "/calico-secrets/etcd-cert"
    etcd_key: "/calico-secrets/etcd-key"  

# etcd 证书 base64 地址 (执行里面的命令生成的证书 base64 码，填入里面)

data:
  etcd-key: (cat /etc/kubernetes/ssl/etcd-key.pem | base64 -w 0)
  etcd-cert: (cat /etc/kubernetes/ssl/etcd.pem | base64 -w 0)
  etcd-ca: (cat /etc/kubernetes/ssl/ca.pem | base64 -w 0)
  
  
# 修改 pods 分配的 IP 段

            - name: CALICO_IPV4POOL_CIDR
              value: "10.254.64.0/18"
```
+ vi /etc/systemd/system/kubelet.service
  --network-plugin=cni \
```bash
# 重新加载配置
systemctl daemon-reload
systemctl restart kubelet.service
systemctl status kubelet.service
```

### 安装 calicoctl
+ curl -O -L https://github.com/projectcalico/calicoctl/releases/download/v3.2.3/calicoctl

```bash
``` 

### flannel
+ vi /usr/lib/systemd/system/docker.service
```bash
EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}

EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock $DOCKER_NETWORK_OPTIONS

```
```bash
etcdctl --endpoints=https://172.17.3.108:2379,https://172.17.3.140:2379,https://172.17.3.223:2379\
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        set /flannel/network/config \ '{"Network":"10.254.64.0/18","SubnetLen":24,"Backend":{"Type":"vxlan"}}'

# check
etcdctl --endpoints=https://172.17.3.108:2379,https://172.17.3.140:2379,https://172.17.3.223:2379 \
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        get /flannel/network/config
# list
etcdctl --endpoints=https://172.17.3.108:2379,https://172.17.3.140:2379,https://172.17.3.223:2379 \
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        ls /flannel/network/subnets

rpm -ivh flannel-0.10.0-1.x86_64.rpm

cat <<EOF > /etc/sysconfig/flanneld
# Flanneld configuration options  

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="https://172.17.3.108:2379,https://172.17.3.140:2379,https://172.17.3.223:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/flannel/network"

# Any additional options that you want to pass
#FLANNEL_OPTIONS=""

FLANNEL_OPTIONS="-ip-masq=true -etcd-cafile=/etc/kubernetes/ssl/ca.pem -etcd-certfile=/etc/kubernetes/ssl/etcd.pem -etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem -iface=eth0"
EOF
```

### ingress
```bash
curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml
curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml
curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml
curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml
curl -O  https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml
curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml

curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml


kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml

# 部署 Ingress RBAC 认证
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml

# 部署 Ingress Controller 组件
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml
```


### ref
+ [k8s jicki](https://jicki.me/kubernetes/2018/04/23/kubernetes-1.10.1.html)

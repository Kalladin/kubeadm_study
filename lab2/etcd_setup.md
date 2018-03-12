# Helpful Docs

- configuring etcd & apiserver
https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/

- etcdctl
https://coreos.com/etcd/docs/latest/dev-guide/interacting_v3.html
- etcd launch setup
https://github.com/coreos/etcd/blob/master/Documentation/op-guide/clustering.md

- etcd security explaination
https://github.com/coreos/etcd/blob/master/Documentation/op-guide/security.md

- etcd disaster recovery
https://github.com/coreos/etcd/blob/master/Documentation/op-guide/recovery.md#restoring-a-cluster

# Setup a Secure Etcd Cluster
- create certificate & keys
- launch 3 etcd nodes by static bootstrapping mechanism

## Create Certificate & Keys
How to use cfssl: `https://coreos.com/os/docs/latest/generate-self-signed-certificates.html`
Example of etcd's cert creating process: `https://github.com/coreos/etcd/tree/master/hack/tls-setup`

- install cfssl & cfssljson
- create root CA
- create cert & key of each nodes for serving clients
- create cert & key of each nodes for peer communication
- create different certs & keys for etcd clients (ex: api-server, etcdctl)

## Launch Nodes
nodes: 10.128.112.153, 10.128.112.154, 10.128.112.155
cluster-token: steven-kube-etcd-cluster-1

```
# for node-0
etcd --name infra0 --initial-advertise-peer-urls https://10.128.112.153:2380 \
  --listen-peer-urls https://10.128.112.153:2380 \
  --listen-client-urls https://10.128.112.153:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.128.112.153:2379 \
  --initial-cluster-token steven-kube-etcd-cluster-1 \
  --initial-cluster infra0=https://10.128.112.153:2380,infra1=https://10.128.112.154:2380,infra2=https://10.128.112.155:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=./cert/ca.pem \
  --cert-file=./cert/infra0-client.pem --key-file=./cert/infra0-client-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=./cert/ca.pem \
  --peer-cert-file=./cert/infra0.pem --peer-key-file=./cert/infra0-key.pem

# for node-1
etcd --name infra1 --initial-advertise-peer-urls https://10.128.112.154:2380 \
  --listen-peer-urls https://10.128.112.154:2380 \
  --listen-client-urls https://10.128.112.154:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.128.112.154:2379 \
  --initial-cluster-token steven-kube-etcd-cluster-1 \
  --initial-cluster infra0=https://10.128.112.153:2380,infra1=https://10.128.112.154:2380,infra2=https://10.128.112.155:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=./cert/ca.pem \
  --cert-file=./cert/infra1-client.pem --key-file=./cert/infra1-client-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=./cert/ca.pem \
  --peer-cert-file=./cert/infra1.pem --peer-key-file=./cert/infra1-key.pem

# for node-2
etcd --name infra2 --initial-advertise-peer-urls https://10.128.112.155:2380 \
  --listen-peer-urls https://10.128.112.155:2380 \
  --listen-client-urls https://10.128.112.155:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.128.112.155:2379 \
  --initial-cluster-token steven-kube-etcd-cluster-1 \
  --initial-cluster infra0=https://10.128.112.153:2380,infra1=https://10.128.112.154:2380,infra2=https://10.128.112.155:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=./cert/ca.pem \
  --cert-file=./cert/infra2-client.pem --key-file=./cert/infra2-client-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=./cert/ca.pem \
  --peer-cert-file=./cert/infra2.pem --peer-key-file=./cert/infra2-key.pem
```

# Switch to external Etcd

- take a snapshot from old etcd cluster: `etcd_snapshot_20180312.db`
- restore snapshot to new etcd cluster `steven-kube-etcd-cluster-1.1` for each etcd node
- edit `/etc/kubernetes/manifests/kube-apiserver.yaml` (add flag:etcd cert/endpoints/...), kubelet will autoly redeploy the api-server pod
- test: `kubectl run my-nginx --image=nginx` to run pod `my-nginx`, confirm the external etcd works

## Restore Snapshot
- restore by the snapshot `etcd_snapshot_20180312.db`
- create new cluster `steven-kube-etcd-cluster-1.{generation}` & new node name `infra{node#}.{generation}`

### Restore Commands
```
ETCDCTL_API=3 etcdctl snapshot restore etcd_snapshot_20180312.db \
  --name infra0.1 \
  --initial-cluster infra0.1=https://10.128.112.153:2380,infra1.1=https://10.128.112.154:2380,infra2.1=https://10.128.112.155:2380 \
  --initial-cluster-token steven-kube-etcd-cluster-1.1 \
  --initial-advertise-peer-urls https://10.128.112.153:2380

ETCDCTL_API=3 etcdctl snapshot restore etcd_snapshot_20180312.db \
  --name infra1.1 \
  --initial-cluster infra0.1=https://10.128.112.153:2380,infra1.1=https://10.128.112.154:2380,infra2.1=https://10.128.112.155:2380 \
  --initial-cluster-token steven-kube-etcd-cluster-1.1 \
  --initial-advertise-peer-urls https://10.128.112.154:2380

ETCDCTL_API=3 etcdctl snapshot restore etcd_snapshot_20180312.db \
  --name infra2.1 \
  --initial-cluster infra0.1=https://10.128.112.153:2380,infra1.1=https://10.128.112.154:2380,infra2.1=https://10.128.112.155:2380 \
  --initial-cluster-token steven-kube-etcd-cluster-1.1 \
  --initial-advertise-peer-urls https://10.128.112.155:2380

```

### Launch Commands
```
# for node-0
etcd --name infra0.1 --initial-advertise-peer-urls https://10.128.112.153:2380 \
  --listen-peer-urls https://10.128.112.153:2380 \
  --listen-client-urls https://10.128.112.153:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.128.112.153:2379 \
  --initial-cluster-token steven-kube-etcd-cluster-1.1 \
  --initial-cluster infra0.1=https://10.128.112.153:2380,infra1.1=https://10.128.112.154:2380,infra2.1=https://10.128.112.155:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=./cert/ca.pem \
  --cert-file=./cert/infra0-client.pem --key-file=./cert/infra0-client-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=./cert/ca.pem \
  --peer-cert-file=./cert/infra0.pem --peer-key-file=./cert/infra0-key.pem &

# for node-1
etcd --name infra1.1 --initial-advertise-peer-urls https://10.128.112.154:2380 \
  --listen-peer-urls https://10.128.112.154:2380 \
  --listen-client-urls https://10.128.112.154:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.128.112.154:2379 \
  --initial-cluster-token steven-kube-etcd-cluster-1.1 \
  --initial-cluster infra0.1=https://10.128.112.153:2380,infra1.1=https://10.128.112.154:2380,infra2.1=https://10.128.112.155:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=./cert/ca.pem \
  --cert-file=./cert/infra1-client.pem --key-file=./cert/infra1-client-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=./cert/ca.pem \
  --peer-cert-file=./cert/infra1.pem --peer-key-file=./cert/infra1-key.pem &

# for node-2
etcd --name infra2.1 --initial-advertise-peer-urls https://10.128.112.155:2380 \
  --listen-peer-urls https://10.128.112.155:2380 \
  --listen-client-urls https://10.128.112.155:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.128.112.155:2379 \
  --initial-cluster-token steven-kube-etcd-cluster-1.1 \
  --initial-cluster infra0.1=https://10.128.112.153:2380,infra1.1=https://10.128.112.154:2380,infra2.1=https://10.128.112.155:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=./cert/ca.pem \
  --cert-file=./cert/infra2-client.pem --key-file=./cert/infra2-client-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=./cert/ca.pem \
  --peer-cert-file=./cert/infra2.pem --peer-key-file=./cert/infra2-key.pem &
```

# Disaster Recovery

- take a snapshot from current etcd cluster `etcd_1.1_snapshot_20180312.db`
- take 2 nodes down, remove 2 nodes' data
- api-server now can't get resources from etcd if half nodes of etcd cluster down (kubectl timeout)
- restore snapshot to new etcd cluster `steven-kube-etcd-cluster-1.2` for each etcd node (including the living one)
- (optional.) if nodes' ip changed, edit `/etc/kubernetes/manifests/kube-apiserver.yaml` (add flag:etcd cert/endpoints/...), kubelet will autoly redeploy the api-server pod
- test: `kubectl run my-nginx2 --image=nginx` to run pod `my-nginx2`, confirm the external etcd works

## Restore
- restore by the snapshot `etcd_1.1_snapshot_20180312.db`
- create new cluster `steven-kube-etcd-cluster-1.{generation}` & new node name `infra{node#}.{generation}`

### Restore Commands
```
ETCDCTL_API=3 etcdctl snapshot restore etcd_snapshots/etcd_1.1_snapshot_20180312.db \
  --name infra0.2 \
  --initial-cluster infra0.2=https://10.128.112.153:2380,infra1.2=https://10.128.112.154:2380,infra2.2=https://10.128.112.155:2380 \
  --initial-cluster-token steven-kube-etcd-cluster-1.2 \
  --initial-advertise-peer-urls https://10.128.112.153:2380

ETCDCTL_API=3 etcdctl snapshot restore etcd_snapshots/etcd_1.1_snapshot_20180312.db \
  --name infra1.2 \
  --initial-cluster infra0.2=https://10.128.112.153:2380,infra1.2=https://10.128.112.154:2380,infra2.2=https://10.128.112.155:2380 \
  --initial-cluster-token steven-kube-etcd-cluster-1.2 \
  --initial-advertise-peer-urls https://10.128.112.154:2380

ETCDCTL_API=3 etcdctl snapshot restore etcd_snapshots/etcd_1.1_snapshot_20180312.db \
  --name infra2.2 \
  --initial-cluster infra0.2=https://10.128.112.153:2380,infra1.2=https://10.128.112.154:2380,infra2.2=https://10.128.112.155:2380 \
  --initial-cluster-token steven-kube-etcd-cluster-1.2 \
  --initial-advertise-peer-urls https://10.128.112.155:2380

```

### Launch Commands
```
# for node-0
etcd --name infra0.2 --initial-advertise-peer-urls https://10.128.112.153:2380 \
  --listen-peer-urls https://10.128.112.153:2380 \
  --listen-client-urls https://10.128.112.153:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.128.112.153:2379 \
  --initial-cluster-token steven-kube-etcd-cluster-1.2 \
  --initial-cluster infra0.2=https://10.128.112.153:2380,infra1.2=https://10.128.112.154:2380,infra2.2=https://10.128.112.155:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=./cert/ca.pem \
  --cert-file=./cert/infra0-client.pem --key-file=./cert/infra0-client-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=./cert/ca.pem \
  --peer-cert-file=./cert/infra0.pem --peer-key-file=./cert/infra0-key.pem &

# for node-1
etcd --name infra1.2 --initial-advertise-peer-urls https://10.128.112.154:2380 \
  --listen-peer-urls https://10.128.112.154:2380 \
  --listen-client-urls https://10.128.112.154:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.128.112.154:2379 \
  --initial-cluster-token steven-kube-etcd-cluster-1.2 \
  --initial-cluster infra0.2=https://10.128.112.153:2380,infra1.2=https://10.128.112.154:2380,infra2.2=https://10.128.112.155:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=./cert/ca.pem \
  --cert-file=./cert/infra1-client.pem --key-file=./cert/infra1-client-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=./cert/ca.pem \
  --peer-cert-file=./cert/infra1.pem --peer-key-file=./cert/infra1-key.pem &

# for node-2
etcd --name infra2.2 --initial-advertise-peer-urls https://10.128.112.155:2380 \
  --listen-peer-urls https://10.128.112.155:2380 \
  --listen-client-urls https://10.128.112.155:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.128.112.155:2379 \
  --initial-cluster-token steven-kube-etcd-cluster-1.2 \
  --initial-cluster infra0.2=https://10.128.112.153:2380,infra1.2=https://10.128.112.154:2380,infra2.2=https://10.128.112.155:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=./cert/ca.pem \
  --cert-file=./cert/infra2-client.pem --key-file=./cert/infra2-client-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=./cert/ca.pem \
  --peer-cert-file=./cert/infra2.pem --peer-key-file=./cert/infra2-key.pem &
```

# TroubleShooting

## How to switch External Etcd From the Old One
- setup external etcd, switch api-server to new etcd endpoints and found all old resources are gone.
- snapshot + restore
- if the k8s cluster is not setup yet, `kubeadm init` provides flag `--config` to set external etcd.

## Etcd Version Should Match the Corresponding Kubeadm
- use v3.3.2 at first, which occurs connection reject issue
  `https://github.com/coreos/etcd/issues/9285`
- switch to v3.1.11 (kubeadm v1.9.3 use etcd 3.1.11,[Nov 29, 2017])
```
etcd Version: 3.1.11
Git SHA: 960f4604b
Go Version: go1.8.5
Go OS/Arch: linux/amd64
```

# Notes
- Scale: Do not configure any auto scaling groups for etcd clusters. It is highly recommended to always run a static five-member etcd cluster for production Kubernetes clusters at any officially supported scale.

# Lab 1: Kubernetes Cluster

# Purposes
1. Create kubernetes cluster by kubeadm
2. Apply `calico` as the network solution
3. Find how the kubeadm start components below:
- `kube-apiserver`
- `kubelet`
- `etcd`
- `kube-controller-manager`
- `kube-schedular`
4. Find out components starting flags, know the meaning of the flags.
5. (Optional.) Know how to use systemd by systemdctl and how to write a systemd unit file

# Calico

## Apply network solution

```
$ kubectl apply -f https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml
configmap "calico-config" created
daemonset "calico-etcd" created
service "calico-etcd" created
daemonset "calico-node" created
deployment "calico-kube-controllers" created
clusterrolebinding "calico-cni-plugin" created
clusterrole "calico-cni-plugin" created
serviceaccount "calico-cni-plugin" created
clusterrolebinding "calico-kube-controllers" created
clusterrole "calico-kube-controllers" created
serviceaccount "calico-kube-controllers" created
```

# Init pod (with networks)

```
kube-system   calico-etcd-472z5                          1/1       Running   0          20m
kube-system   calico-kube-controllers-559b575f97-2fs6j   1/1       Running   0          20m
kube-system   calico-node-8tpf2                          2/2       Running   0          12m
kube-system   calico-node-jxvdp                          2/2       Running   0          20m
kube-system   etcd-kube-master-01                        1/1       Running   0          25m
kube-system   kube-apiserver-kube-master-01              1/1       Running   0          25m
kube-system   kube-controller-manager-kube-master-01     1/1       Running   0          25m
kube-system   kube-dns-6f4fd4bdf-pb7c6                   3/3       Running   0          26m
kube-system   kube-proxy-6lb2d                           1/1       Running   0          26m
kube-system   kube-proxy-mk4wh                           1/1       Running   0          12m
kube-system   kube-scheduler-kube-master-01              1/1       Running   0          25m
```

# Main Components

## Kubelet
kubelet, run as systemd service

### Deploy
Systemd service

### Setting Location
`/lib/systemd/system/kubelet.service`
`/etc/systemd/system/kubelet.service.d`

### Setting Parameters

**/lib/systemd/system/kubelet.service**
```
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=http://kubernetes.io/docs/

[Service]
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target

```

**/etc/systemd/system/kubelet.service.d**
```
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS

```

##  kube-controller-manager

### Deploy
Pod

### Settings Location
template:
`/etc/kubernetes/manifests/kube-controller-manager.yaml`

### Settings Parameters

```
  - command:
    - kube-controller-manager
    - --controllers=*,bootstrapsigner,tokencleaner
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --leader-elect=true
    - --use-service-account-credentials=true
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --address=127.0.0.1
    - --allocate-node-cidrs=true
    - --cluster-cidr=192.168.0.0/16
    - --node-cidr-mask-size=24
```

| Flag                               | Type        | Value                                   | Description                                                                                                                                                             |
| ---------------------------------- | ----------- | --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| --controllers                      | stringSlice | *,bootstrapsigner,tokencleaner          | A list of controllers to enable.  '*' enables all on-by-default controllers, 'foo' enables the controller named 'foo', '-foo' disables the controller named 'foo'.      |
| --kubeconfig                       | string      | /etc/kubernetes/controller-manager.conf | Path to kubeconfig file with authorization and master location information.                                                                                             |
| --root-ca-file                     | string      | /etc/kubernetes/pki/ca.crt              | If set, this root certificate authority will be included in service account's token secret. This must be a valid PEM-encoded CA bundle.                                 |
| --leader-elect                     | boolean     | true                                    | Start a leader election client and gain leadership before executing the main loop. Enable this when running replicated components for high availability. (default true) |
| --use-service-account-credentials  | boolean     | true                                    | If true, use individual service account credentials for each controller.                                                                                                |
| --service-account-private-key-file | string      | /etc/kubernetes/pki/sa.key              | Filename containing a PEM-encoded private RSA or ECDSA key used to sign service account tokens.                                                                         |
| --cluster-signing-cert-file        | string      | /etc/kubernetes/pki/ca.crt              | Filename containing a PEM-encoded X509 CA certificate used to issue cluster-scoped certificates (default "/etc/kubernetes/ca/ca.pem")                                   |
| --cluster-signing-key-file         | string      | /etc/kubernetes/pki/ca.key              | Filename containing a PEM-encoded RSA or ECDSA private key used to sign cluster-scoped certificates (default "/etc/kubernetes/ca/ca.key")                               |
| --address                          | string      | 127.0.0.1                               | The IP address to serve on (set to 0.0.0.0 for all interfaces) (default 0.0.0.0)                                                                                        |
| --allocate-node-cidrs              | boolean     | true                                    | Should CIDRs for Pods be allocated and set on the cloud provider.                                                                                                       |
| --cluster-cidr                     | string      | 192.168.0.0/16                          | CIDR Range for Pods in cluster. Requires --allocate-node-cidrs to be true                                                                                               |
| --node-cidr-mask-size              | int32       | 24                                      | Mask size for node cidr in cluster. (default 24)                                                                                                                        |

## kube-apiserver

### Deploy
Pod

### Settings Location
template:
`/etc/kubernetes/manifests/kube-apiserver.yaml`

### Settings Parameters

```
  - command:
    - kube-apiserver
    - --requestheader-allowed-names=front-proxy-client
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --insecure-port=0
    - --admission-control=Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota
    - --allow-privileged=true
    - --requestheader-username-headers=X-Remote-User
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --secure-port=6443
    - --enable-bootstrap-token-auth=true
    - --advertise-address=10.128.112.146
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    - --authorization-mode=Node,RBAC
    - --etcd-servers=http://127.0.0.1:2379

```

| Flag                                 | Type        | Value                                                                                                                                 | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ------------------------------------ | ----------- | ------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| --requestheader-allowed-names        | stringSlice | front-proxy-client                                                                                                                    | List of client certificate common names to allow to provide usernames in headers specified by --requestheader-username-headers. If empty, any client certificate validated by the authorities in --requestheader-client-ca-file is allowed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| --service-account-key-file           | stringArray | /etc/kubernetes/pki/sa.pub                                                                                                            | File containing PEM-encoded x509 RSA or ECDSA private or public keys, used to verify ServiceAccount tokens. If unspecified, --tls-private-key-file is used. The specified file can contain multiple keys, and the flag can be specified multiple times with different files.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| --proxy-client-cert-file             | string      | /etc/kubernetes/pki/front-proxy-client.crt                                                                                            | Client certificate used to prove the identity of the aggregator or kube-apiserver when it must call out during a request. This includes proxying requests to a user api-server and calling out to webhook admission plugins. It is expected that this cert includes a signature from the CA in the --requestheader-client-ca-file flag. That CA is published in the 'extension-apiserver-authentication' configmap in the kube-system namespace. Components receiving calls from kube-aggregator should use that CA to perform their half of the mutual TLS verification.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| --kubelet-preferred-address-types    | stringSlice | InternalIP,ExternalIP,Hostname                                                                                                        | List of the preferred NodeAddressTypes to use for kubelet connections. (default [Hostname,InternalDNS,InternalIP,ExternalDNS,ExternalIP])                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| --requestheader-group-headers        | stringSlice | X-Remote-Group                                                                                                                        | List of request headers to inspect for groups. X-Remote-Group is suggested.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| --requestheader-extra-headers-prefix | stringSlice | X-Remote-Extra-                                                                                                                       | List of request header prefixes to inspect. X-Remote-Extra- is suggested.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| --service-cluster-ip-range           | ipNet       | 10.96.0.0/12                                                                                                                          | A CIDR notation IP range from which to assign service cluster IPs. This must not overlap with any IP ranges assigned to nodes for pods.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| --tls-cert-file                      | string      | /etc/kubernetes/pki/apiserver.crt                                                                                                     | File containing the default x509 Certificate for HTTPS. (CA cert, if any, concatenated after server cert). If HTTPS serving is enabled, and --tls-cert-file and --tls-private-key-file are not provided, a self-signed certificate and key are generated for the public address and saved to the directory specified by --cert-dir.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| --requestheader-client-ca-file       | string      | /etc/kubernetes/pki/front-proxy-ca.crt                                                                                                | Root certificate bundle to use to verify client certificates on incoming requests before trusting usernames in headers specified by --requestheader-username-headers                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| --proxy-client-key-file              | string      | /etc/kubernetes/pki/front-proxy-client.key                                                                                            | Private key for the client certificate used to prove the identity of the aggregator or kube-apiserver when it must call out during a request. This includes proxying requests to a user api-server and calling out to webhook admission plugins.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| --insecure-port                      | int         | 0                                                                                                                                     | The port on which to serve unsecured, unauthenticated access. It is assumed that firewall rules are set up such that this port is not reachable from outside of the cluster and that port 443 on the cluster's public address is proxied to this port. This is performed by nginx in the default setup. (default 8080)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| --admission-control                  | stringSlice | Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota | Admission is divided into two phases. In the first phase, only mutating admission plugins run. In the second phase, only validating admission plugins run. The names in the below list may represent a validating plugin, a mutating plugin, or both. Within each phase, the plugins will run in the order in which they are passed to this flag. Comma-delimited list of: AlwaysAdmit, AlwaysDeny, AlwaysPullImages, DefaultStorageClass, DefaultTolerationSeconds, DenyEscalatingExec, DenyExecOnPrivileged, EventRateLimit, ExtendedResourceToleration, ImagePolicyWebhook, InitialResources, Initializers, LimitPodHardAntiAffinityTopology, LimitRanger, MutatingAdmissionWebhook, NamespaceAutoProvision, NamespaceExists, NamespaceLifecycle, NodeRestriction, OwnerReferencesPermissionEnforcement, PVCProtection, PersistentVolumeClaimResize, PersistentVolumeLabel, PodNodeSelector, PodPreset, PodSecurityPolicy, PodTolerationRestriction, Priority, ResourceQuota, SecurityContextDeny, ServiceAccount, ValidatingAdmissionWebhook. (default [AlwaysAdmit]) |
| --allow-privileged                   | boolean     | true                                                                                                                                  | If true, allow privileged containers. [default=false]                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| --requestheader-username-headers     | stringSlice | X-Remote-User                                                                                                                         | List of request headers to inspect for usernames. X-Remote-User is common.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| --client-ca-file                     | string      | /etc/kubernetes/pki/ca.crt                                                                                                            | If set, any request presenting a client certificate signed by one of the authorities in the client-ca-file is authenticated with an identity corresponding to the CommonName of the client certificate.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| --kubelet-client-certificate         | string      | /etc/kubernetes/pki/apiserver-kubelet-client.crt                                                                                      | Path to a client cert file for TLS.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| --kubelet-client-key                 | string      | /etc/kubernetes/pki/apiserver-kubelet-client.key                                                                                      | Path to a client key file for TLS.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| --secure-port                        | int         | 6443                                                                                                                                  | The port on which to serve HTTPS with authentication and authorization. If 0, don't serve HTTPS at all. (default 6443)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| --enable-bootstrap-token-auth        | boolean     | true                                                                                                                                  | Enable to allow secrets of type 'bootstrap.kubernetes.io/token' in the 'kube-system' namespace to be used for TLS bootstrapping authentication.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| --advertise-address                  | ip          | 10.128.112.146                                                                                                                        | The IP address on which to advertise the apiserver to members of the cluster. This address must be reachable by the rest of the cluster. If blank, the --bind-address will be used. If --bind-address is unspecified, the host's default interface will be used.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| --tls-private-key-file               | string      | /etc/kubernetes/pki/apiserver.key                                                                                                     | File containing the default x509 private key matching --tls-cert-file.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| --authorization-mode                 | string      | Node,RBAC                                                                                                                             | Ordered list of plug-ins to do authorization on secure port. Comma-delimited list of: AlwaysAllow,AlwaysDeny,ABAC,Webhook,RBAC,Node. (default "AlwaysAllow")                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| --etcd-servers                       | stringSlice | http://127.0.0.1:2379                                                                                                                 | List of etcd servers to connect with (scheme://ip:port), comma separated.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |


## kube-schedular

### Deploy
Pod

### Settings Location
template:
`/etc/kubernetes/manifests/kube-schedular.yaml`

### Settings Parameters

```
  - command:
    - kube-scheduler
    - --address=127.0.0.1
    - --leader-elect=true
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    image: gcr.io/google_containers/kube-scheduler-amd64:v1.9.3
```

| Flag           | Type    | Value                          | Description                                                                                                                                              |
| -------------- | ------- | ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| --address      | string  | 127.0.0.1                      | The IP address to serve on (set to 0.0.0.0 for all interfaces)                                                                                           |
| --leader-elect | boolean | true                           | Start a leader election client and gain leadership before executing the main loop. Enable this when running replicated components for high availability. |
| --kubeconfig   | string  | /etc/kubernetes/scheduler.conf | Path to kubeconfig file with authorization and master location information.                                                                              |


## etcd

### Deploy
Pod

### Settings Location
template:
`/etc/kubernetes/manifests/etcd.yaml`

### Settings Parameters

`https://coreos.com/etcd/docs/latest/v2/configuration.html`

- image: gcr.io/google_containers/etcd-amd64:3.1.11
- command:
```
- etcd
- --listen-client-urls=http://127.0.0.1:2379
- --advertise-client-urls=http://127.0.0.1:2379
- --data-dir=/var/lib/etcd
```
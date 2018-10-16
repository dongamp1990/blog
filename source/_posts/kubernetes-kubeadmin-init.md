---
title: 使用kubeadm快速构建集群
tags: [kubernetes]
date: 2018-06-15
---
## 前言
本文将介绍如何在Ubuntu server 16.03版本上安装kubeadm，构建一个kubernetes的基础的测试集群，用来做学习和测试用途，当前最新的版本是1.10.4
使用CoreDNS替换kubeDNS，使用calico网络，部署dashboard，部署heapster+grafana+influxdb监控

本次安装2台集群，1个master，1个node，可以根据自己需求增加node机器，每台服务器4G内存，2个CPU核心以上
所有服务器使用Ubuntu server 16.03
所有节点提前安装好Docker[安装示例](/2018/01/11/docker-install-note/)

## 安装kubeadm
在所有节点上安装kubeadm

查看apt安装源如下配置，
```
$ cat /etc/apt/sources.list

# kubeadm及kubernetes组件安装源
deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
```

更新源，可以不理会gpg的报错信息。
```
$ apt-get update

Ign:4 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial InRelease
Fetched 8,993 B in 0s (20.7 kB/s)
Reading package lists... Done
W: GPG error: https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 6A030B21BA07F4FB
W: The repository 'https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial InRelease' is not signed.
N: Data from such a repository can't be authenticated and is therefore potentially dangerous to use.
N: See apt-secure(8) manpage for repository creation and user configuration details.
```

强制安装kubeadm，kubectl，kubelet软件包。

```
$ apt-get install -y kubelet kubeadm kubectl kubernetes-cni --allow-unauthenticated
```
kubeadm安装完以后，就可以使用它来快速安装部署Kubernetes集群了。


## 下载镜像文件
因为在墙内的关系，访问不到gcr站点，所以需要先准备好所需镜像,提供的镜像是1.10.1版本的。
地址: https://pan.baidu.com/s/1CmoDA3jQgD8kkF3QfI90qg 密码: fkxw
下载kube1.10.tar.gz并解压。

master节点load镜像
```
$ docker load -i kube-master.tar
$ docker load -i calico-3.1.3.tar
$ docker load -i coredns-1.1.3.tar
```

node节点load镜像
```
$ docker load -i kube-node.tar
$ docker load -i calico-3.1.3.tar
$ docker load -i coredns-1.1.3.tar
```

## 初始化master节点
--pod-network-cidr这个参数的ip地址可以根据实际情况修改，需要跟calico网络的CALICO_IPV4POOL_CIDR一致
```
$ kubeadm init --pod-network-cidr=172.16.1.0/16 --kubernetes-version v1.10.1

[init] Using Kubernetes version: v1.10.1
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks.
	[WARNING FileExisting-crictl]: crictl not found in system path
Suggestion: go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
[preflight] Starting the kubelet service
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [ubuntu-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.10.250]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated etcd/ca certificate and key.
[certificates] Generated etcd/server certificate and key.
[certificates] etcd/server serving cert is signed for DNS names [localhost] and IPs [127.0.0.1]
[certificates] Generated etcd/peer certificate and key.
[certificates] etcd/peer serving cert is signed for DNS names [ubuntu-master] and IPs [192.168.10.250]
[certificates] Generated etcd/healthcheck-client certificate and key.
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests".
[init] This might take a minute or longer if the control plane images have to be pulled.
[apiclient] All control plane components are healthy after 28.003828 seconds
[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[markmaster] Will mark node ubuntu-master as master by adding a label and a taint
[markmaster] Master ubuntu-master tainted and labelled with key/value: node-role.kubernetes.io/master=""
[bootstraptoken] Using token: rw4enn.mvk547juq7qi2b5f
[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.10.250:6443 --token rw4enn.mvk547juq7qi2b5f --discovery-token-ca-cert-hash sha256:ba260d5191213382a806a9a7d92c9e6bb09061847c7914b1ac584d0c69471579
```

执行如下命令来配置kubectl
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
这样master的节点就配置好了，并且可以使用kubectl来进行各种操作了，根据上面的提示接着往下做，将slave节点加入到集群。

## node节点加入集群
在node节点上执行kubeadm join命令
```
$ kubeadm join 192.168.10.250:6443 --token rw4enn.mvk547juq7qi2b5f --discovery-token-ca-cert-hash sha256:ba260d5191213382a806a9a7d92c9e6bb09061847c7914b1ac584d0c69471579

[preflight] Running pre-flight checks.
	[WARNING FileExisting-crictl]: crictl not found in system path
Suggestion: go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
[discovery] Trying to connect to API Server "192.168.10.250:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.10.250:6443"
[discovery] Requesting info from "https://192.168.10.250:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.10.250:6443"
[discovery] Successfully established connection with API Server "192.168.10.250:6443"

This node has joined the cluster:
* Certificate signing request was sent to master and a response
  was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

查看节点状态
```
$ kubectl get nodes

NAME         STATUS    ROLES     AGE       VERSION
k8s-master   Ready     master    1h        v1.10.4
k8s-node1    Ready     <none>    1h        v1.10.4
```

在master节点查看pod状态
```
$ kubectl get pods -n kube-system -o wide
```

## 安装Calico网络

[calico.yaml文件](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/calico.yaml)
```
$ cat <<EOF > calico.yaml
# Calico Version v3.1.3
# https://docs.projectcalico.org/v3.1/releases#v3.1.3
# This manifest includes the following component versions:
#   calico/node:v3.1.3
#   calico/cni:v3.1.3
#   calico/kube-controllers:v3.1.3

# This ConfigMap is used to configure a self-hosted Calico installation.
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-config
  namespace: kube-system
data:
  # The location of your etcd cluster.  This uses the Service clusterIP defined below.
  etcd_endpoints: "http://10.96.232.136:6666"

  # Configure the Calico backend to use.
  calico_backend: "bird"

  # The CNI network configuration to install on each node.
  cni_network_config: |-
    {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.0",
      "plugins": [
        {
          "type": "calico",
          "etcd_endpoints": "__ETCD_ENDPOINTS__",
          "log_level": "info",
          "mtu": 1500,
          "ipam": {
              "type": "calico-ipam",
              "subnet": "usePodCidr"
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "__KUBECONFIG_FILEPATH__"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        }
      ]
    }

---

# This manifest installs the Calico etcd on the kubeadm master.  This uses a DaemonSet
# to force it to run on the master even when the master isn't schedulable, and uses
# nodeSelector to ensure it only runs on the master.
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: calico-etcd
  namespace: kube-system
  labels:
    k8s-app: calico-etcd
spec:
  template:
    metadata:
      labels:
        k8s-app: calico-etcd
      annotations:
        # Mark this pod as a critical add-on; when enabled, the critical add-on scheduler
        # reserves resources for critical add-on pods so that they can be rescheduled after
        # a failure.  This annotation works in tandem with the toleration below.
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      tolerations:
        # This taint is set by all kubelets running `--cloud-provider=external`
        # so we should tolerate it to schedule the calico pods
        - key: node.cloudprovider.kubernetes.io/uninitialized
          value: "true"
          effect: NoSchedule
        # Allow this pod to run on the master.
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        # Allow this pod to be rescheduled while the node is in "critical add-ons only" mode.
        # This, along with the annotation above marks this pod as a critical add-on.
        - key: CriticalAddonsOnly
          operator: Exists
      # Only run this pod on the master.
      nodeSelector:
        node-role.kubernetes.io/master: ""
      hostNetwork: true
      containers:
        - name: calico-etcd
          image: quay.io/coreos/etcd:v3.1.10
          env:
            - name: CALICO_ETCD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          command:
          - /usr/local/bin/etcd
          args:
          - --name=calico
          - --data-dir=/var/etcd/calico-data
          - --advertise-client-urls=http://$CALICO_ETCD_IP:6666
          - --listen-client-urls=http://0.0.0.0:6666
          - --listen-peer-urls=http://0.0.0.0:6667
          - --auto-compaction-retention=1
          volumeMounts:
            - name: var-etcd
              mountPath: /var/etcd
      volumes:
        - name: var-etcd
          hostPath:
            path: /var/etcd

---

# This manifest installs the Service which gets traffic to the Calico
# etcd.
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: calico-etcd
  name: calico-etcd
  namespace: kube-system
spec:
  # Select the calico-etcd pod running on the master.
  selector:
    k8s-app: calico-etcd
  # This ClusterIP needs to be known in advance, since we cannot rely
  # on DNS to get access to etcd.
  clusterIP: 10.96.232.136
  ports:
    - port: 6666

---

# This manifest installs the calico/node container, as well
# as the Calico CNI plugins and network config on
# each master and worker node in a Kubernetes cluster.
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: calico-node
  namespace: kube-system
  labels:
    k8s-app: calico-node
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        k8s-app: calico-node
      annotations:
        # Mark this pod as a critical add-on; when enabled, the critical add-on scheduler
        # reserves resources for critical add-on pods so that they can be rescheduled after
        # a failure.  This annotation works in tandem with the toleration below.
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      hostNetwork: true
      tolerations:
        # Make sure calico/node gets scheduled on all nodes.
        - effect: NoSchedule
          operator: Exists
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
        - effect: NoExecute
          operator: Exists
      serviceAccountName: calico-cni-plugin
      # Minimize downtime during a rolling upgrade or deletion; tell Kubernetes to do a "force
      # deletion": https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods.
      terminationGracePeriodSeconds: 0
      containers:
        # Runs calico/node container on each Kubernetes node.  This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: quay.io/calico/node:v3.1.3
          env:
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Enable BGP.  Disable to enforce policy only.
            - name: CALICO_NETWORKING_BACKEND
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: calico_backend
            # Cluster type to identify the deployment type
            - name: CLUSTER_TYPE
              value: "kubeadm,bgp"
            # Disable file logging so `kubectl logs` works.
            - name: CALICO_DISABLE_FILE_LOGGING
              value: "true"
            # Set noderef for node controller.
            - name: CALICO_K8S_NODE_REF
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            # Set Felix endpoint to host default action to ACCEPT.
            - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
              value: "ACCEPT"
            # The default IPv4 pool to create on startup if none exists. Pod IPs will be
            # chosen from this range. Changing this value after installation will have
            # no effect. This should fall within `--cluster-cidr`.
            - name: CALICO_IPV4POOL_CIDR
              value: "172.16.1.0/16"
            - name: CALICO_IPV4POOL_IPIP
              value: "Always"
            # Disable IPv6 on Kubernetes.
            - name: FELIX_IPV6SUPPORT
              value: "false"
            # Set MTU for tunnel device used if ipip is enabled
            - name: FELIX_IPINIPMTU
              value: "1440"
            # Set Felix logging to "info"
            - name: FELIX_LOGSEVERITYSCREEN
              value: "info"
            # Auto-detect the BGP IP address.
            - name: IP
              value: "autodetect"
            - name: FELIX_HEALTHENABLED
              value: "true"
          securityContext:
            privileged: true
          resources:
            requests:
              cpu: 250m
          livenessProbe:
            httpGet:
              path: /liveness
              port: 9099
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
          readinessProbe:
            httpGet:
              path: /readiness
              port: 9099
            periodSeconds: 10
          volumeMounts:
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - mountPath: /var/run/calico
              name: var-run-calico
              readOnly: false
            - mountPath: /var/lib/calico
              name: var-lib-calico
              readOnly: false
        # This container installs the Calico CNI binaries
        # and CNI network config file on each node.
        - name: install-cni
          image: quay.io/calico/cni:v3.1.3
          command: ["/install-cni.sh"]
          env:
            # Name of the CNI config file to create.
            - name: CNI_CONF_NAME
              value: "10-calico.conflist"
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # The CNI network config to install on each node.
            - name: CNI_NETWORK_CONFIG
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: cni_network_config
          volumeMounts:
            - mountPath: /host/opt/cni/bin
              name: cni-bin-dir
            - mountPath: /host/etc/cni/net.d
              name: cni-net-dir
      volumes:
        # Used by calico/node.
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: var-run-calico
          hostPath:
            path: /var/run/calico
        - name: var-lib-calico
          hostPath:
            path: /var/lib/calico
        # Used to install CNI.
        - name: cni-bin-dir
          hostPath:
            path: /opt/cni/bin
        - name: cni-net-dir
          hostPath:
            path: /etc/cni/net.d

---

# This manifest deploys the Calico Kubernetes controllers.
# See https://github.com/projectcalico/kube-controllers
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: calico-kube-controllers
  namespace: kube-system
  labels:
    k8s-app: calico-kube-controllers
spec:
  # The controllers can only have a single active instance.
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      name: calico-kube-controllers
      namespace: kube-system
      labels:
        k8s-app: calico-kube-controllers
      annotations:
        # Mark this pod as a critical add-on; when enabled, the critical add-on scheduler
        # reserves resources for critical add-on pods so that they can be rescheduled after
        # a failure.  This annotation works in tandem with the toleration below.
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      # The controllers must run in the host network namespace so that
      # it isn't governed by policy that would prevent it from working.
      hostNetwork: true
      tolerations:
        # Allow this pod to be rescheduled while the node is in "critical add-ons only" mode.
        # This, along with the annotation above marks this pod as a critical add-on.
        - key: CriticalAddonsOnly
          operator: Exists
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      serviceAccountName: calico-kube-controllers
      containers:
        - name: calico-kube-controllers
          image: quay.io/calico/kube-controllers:v3.1.3
          env:
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Choose which controllers to run.
            - name: ENABLED_CONTROLLERS
              value: policy,profile,workloadendpoint,node

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: calico-cni-plugin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-cni-plugin
subjects:
- kind: ServiceAccount
  name: calico-cni-plugin
  namespace: kube-system

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: calico-cni-plugin
rules:
  - apiGroups: [""]
    resources:
      - pods
      - nodes
    verbs:
      - get

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-cni-plugin
  namespace: kube-system

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: calico-kube-controllers
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-kube-controllers
subjects:
- kind: ServiceAccount
  name: calico-kube-controllers
  namespace: kube-system

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: calico-kube-controllers
rules:
  - apiGroups:
    - ""
    - extensions
    resources:
      - pods
      - namespaces
      - networkpolicies
      - nodes
    verbs:
      - watch
      - list
  - apiGroups:
    - networking.k8s.io
    resources:
      - networkpolicies
    verbs:
      - watch
      - list

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-kube-controllers
  namespace: kube-system

EOF
```

部署calico
```
kubectl apply -f calico.yaml
```

## 安装CoreDNS
在/etc/kubernetes/manifests/下创建coredns目录,
需要把官方提供的两个文件deploy.sh和coredns.yaml.sed放到conredns目录

### 准备配置文件
```
$ mkdir -p /etc/kubernetes/manifests/coredns
```

[deploy.sh文件](https://github.com/coredns/deployment/blob/master/kubernetes/deploy.sh)
```
$ cat <<EOF > /etc/kubernetes/manifests/coredns/deploy.sh
#!/bin/bash

# Deploys CoreDNS to a cluster currently running Kube-DNS.

show_help () {
cat << USAGE
usage: $0 [ -r REVERSE-CIDR ] [ -i DNS-IP ] [ -d CLUSTER-DOMAIN ] [ -t YAML-TEMPLATE ]
    -r : Define a reverse zone for the given CIDR. You may specifcy this option more
         than once to add multiple reverse zones. If no reverse CIDRs are defined,
         then the default is to handle all reverse zones (i.e. in-addr.arpa and ip6.arpa)
    -i : Specify the cluster DNS IP address. If not specificed, the IP address of
         the existing "kube-dns" service is used, if present.
USAGE
exit 0
}

# Simple Defaults
CLUSTER_DOMAIN=cluster.local
YAML_TEMPLATE=`pwd`/coredns.yaml.sed


# Get Opts
while getopts "hr:i:d:t:" opt; do
    case "$opt" in
    h)  show_help
        ;;
    r)  REVERSE_CIDRS="$REVERSE_CIDRS $OPTARG"
        ;;
    i)  CLUSTER_DNS_IP=$OPTARG
        ;;
    d)  CLUSTER_DOMAIN=$OPTARG
        ;;
    t)  YAML_TEMPLATE=$OPTARG
        ;;
    esac
done

# Conditional Defaults
if [[ -z $REVERSE_CIDRS ]]; then
  REVERSE_CIDRS="in-addr.arpa ip6.arpa"
fi
if [[ -z $CLUSTER_DNS_IP ]]; then
  # Default IP to kube-dns IP
  CLUSTER_DNS_IP=$(kubectl get service --namespace kube-system kube-dns -o jsonpath="{.spec.clusterIP}")
  if [ $? -ne 0 ]; then
      >&2 echo "Error! The IP address for DNS service couldn't be determined automatically. Please specify the DNS-IP with the '-i' option."
      exit 2
  fi
fi

sed -e s/CLUSTER_DNS_IP/$CLUSTER_DNS_IP/g -e s/CLUSTER_DOMAIN/$CLUSTER_DOMAIN/g -e "s?REVERSE_CIDRS?$REVERSE_CIDRS?g" $YAML_TEMPLATE
EOF
```

[coredns.yaml.sed文件](https://github.com/coredns/deployment/blob/master/kubernetes/coredns.yaml.sed)
```
$ cat <<EOF > /etc/kubernetes/manifests/coredns/coredns.yaml.sed
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes CLUSTER_DOMAIN REVERSE_CIDRS {
          pods insecure
          upstream
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        proxy . /etc/resolv.conf
        cache 30
        reload
    }
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      containers:
      - name: coredns
        image: coredns/coredns:1.1.3
        imagePullPolicy: IfNotPresent
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: CLUSTER_DNS_IP
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
EOF

```

### 使用CoreDNS替换Kube-DNS
最佳的案例场景中，使用CoreDNS替换Kube-DNS只需要使用下面的两个命令：
```
$ ./deploy.sh | kubectl apply -f -
$ kubectl delete --namespace=kube-system deployment kube-dns
```

查看pod情况
```
$ kubectl get pods -n kube-system -o wide

NAME                                       READY     STATUS    RESTARTS   AGE       IP               NODE
calico-etcd-2fn6s                          1/1       Running   0          7m        192.168.10.250   k8s-master
calico-kube-controllers-679568f47c-k7q4w   1/1       Running   0          7m        192.168.10.250   k8s-master
calico-node-8l2mf                          2/2       Running   0          7m        192.168.10.250   k8s-master
calico-node-gx4x8                          2/2       Running   0          5m        192.168.10.57    k8s-node1
coredns-6cc44759b8-8p8cb                   1/1       Running   1          6h        172.16.1.193     k8s-master
coredns-6cc44759b8-xzmfr                   1/1       Running   1          6h        172.16.1.194     k8s-master
etcd-k8s-master                            1/1       Running   3          7h        192.168.10.250   k8s-master
heapster-77b9c5bd7b-cdr7d                  1/1       Running   0          4m        172.16.1.68      k8s-node1
kube-apiserver-k8s-master                  1/1       Running   3          7h        192.168.10.250   k8s-master
kube-controller-manager-k8s-master         1/1       Running   3          7h        192.168.10.250   k8s-master
kube-proxy-7nvtx                           1/1       Running   3          7h        192.168.10.250   k8s-master
kube-proxy-nh6jf                           1/1       Running   0          5m        192.168.10.57    k8s-node1
kube-scheduler-k8s-master                  1/1       Running   3          7h        192.168.10.250   k8s-master
```

至此，node集群已经部署好了。

## 部署Dashboard
node节点load dashboard镜像
```
docker load -i kube-dashboard-1.8.3.tar
```

dashboard.yaml
```
cat <<EOF > dashboard.yaml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Configuration to deploy release version of the Dashboard UI compatible with
# Kubernetes 1.8.
#
# Example usage: kubectl create -f <this_file>

# ------------------- Dashboard Secret ------------------- #

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque

---
# ------------------- Dashboard Service Account ------------------- #

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system

---
# ------------------- Dashboard Role & Role Binding ------------------- #

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
rules:
  # Allow Dashboard to create 'kubernetes-dashboard-key-holder' secret.
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
  # Allow Dashboard to create 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
  verbs: ["get", "update", "delete"]
  # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kubernetes-dashboard-settings"]
  verbs: ["get", "update"]
  # Allow Dashboard to get metrics from heapster.
- apiGroups: [""]
  resources: ["services"]
  resourceNames: ["heapster"]
  verbs: ["proxy"]
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard-minimal
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system

---
# ------------------- Dashboard Deployment ------------------- #

kind: Deployment
apiVersion: apps/v1beta2
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        image: reg.qiniu.com/k8s/kubernetes-dashboard-amd64:v1.8.3
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
          # Uncomment the following line to manually specify Kubernetes API server Host
          # If not specified, Dashboard will attempt to auto discover the API server and connect
          # to it. Uncomment only if the default does not work.
          # - --apiserver-host=http://my-address:port
        volumeMounts:
        - name: kubernetes-dashboard-certs
          mountPath: /certs
          # Create on-disk volume to store exec logs
        - mountPath: /tmp
          name: tmp-volume
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: kubernetes-dashboard-certs
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule

---
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
    - port: 443
      targetPort: 8443
      #以https://ip:nodeport访问dashboard
      nodePort: 32500
  type: NodePort
  selector:
    k8s-app: kubernetes-dashboard

EOF
```

部署dashboard
```
$ kubectl apply -f dashboard.yaml
```
部署dashboard后，为了安全默认是不能匿名访问dashboard的，所以要创建一个cluster-admin角色的用户，使用token登录dashboard

admin-user.yaml
```
cat <<EOF > admin-user.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile

EOF
```

创建admin-user用户
```
$ kubectl apply -f admin-user.yaml
```

查看admin-user用户token， 使用token去登录dashboard
```
$kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

Name:         admin-user-token-nfsg4
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=6a91db25-704e-11e8-8055-080027c6c772

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLW5mc2c0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2YTkxZGIyNS03MDRlLTExZTgtODA1NS0wODAwMjdjNmM3NzIiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.Ue49ptrBZItvG7f8gavEQSoxtSgbz-bh3jZU9RbjLJh2nkmueTVh5WViauUd5TxsnmefB3DDLgTp_CfwWrV0Uyf7JT6GEDMvjDVhsyHa2qyjLwRfvf1vmnx2P7975yRQniq_a3fgOhdNn0zCL2Er9bDUNabCcR0ubNW6I63kp3-UYGIj9OfAkfqNCTFtjZgvcGvnvkhj2pm_peTJ6H3qmxhcb9WM90Nh77p5qdI8gjk2EdKPAhsOmOxBWSSnHqHPr0gVStsewQEHo0CQOr4MxE3NIg_gwjTbn0KD9vvJeECPAX_zsq7ejD0POoWvSXz8khwvv8yLgJQA-ZkUXJKk7w
ca.crt:     1025 bytes
namespace:  11 bytes
```

访问https://ip:32500即可打开dashboard页面

## 部署heapster、grafana、influxdb

### 配置文件
heapster.yaml
```
cat <<EOF > heapster.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: heapster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:heapster
subjects:
- kind: ServiceAccount
  name: heapster
  namespace: kube-system

---  
apiVersion: v1
kind: ServiceAccount
metadata:
  name: heapster
  namespace: kube-system
  
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: heapster
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: heapster
    spec:
      serviceAccountName: heapster
      containers:
      - name: heapster
        image: mirrorgooglecontainers/heapster-amd64:v1.5.3
        imagePullPolicy: IfNotPresent
        #volumeMounts: 
        #- mountPath: /root/.kube
        #  name: config
        command:
        - /heapster
        - --source=kubernetes:https://kubernetes.default
        - --sink=influxdb:http://monitoring-influxdb.kube-system.svc:8086
---
apiVersion: v1
kind: Service
metadata:
  labels:
    task: monitoring
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: Heapster
  name: heapster
  namespace: kube-system
spec:
  ports:
  - port: 80
    targetPort: 8082
  selector:
    k8s-app: heapster
  
EOF
```

grafana.yaml
```
cat <<EOF > grafana.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: monitoring-grafana
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: grafana
    spec:
      containers:
      - name: grafana
        image: mirrorgooglecontainers/heapster-grafana-amd64:v4.4.3
        ports:
        - containerPort: 3000
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/ssl/certs
          name: ca-certificates
          readOnly: true
        - mountPath: /var
          name: grafana-storage
        env:
        - name: INFLUXDB_HOST
          value: monitoring-influxdb
        - name: GF_SERVER_HTTP_PORT
          value: "3000"
          # The following env variables are required to make Grafana accessible via
          # the kubernetes api-server proxy. On production clusters, we recommend
          # removing these env variables, setup auth for grafana, and expose the grafana
          # service using a LoadBalancer or a public IP.
        - name: GF_AUTH_BASIC_ENABLED
          value: "false"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          value: Admin
        - name: GF_SERVER_ROOT_URL
          # If you're only using the API Server proxy, set this value instead:
          # value: /api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
          value: /
      volumes:
      - name: ca-certificates
        hostPath:
          path: /etc/ssl/certs
      - name: grafana-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: monitoring-grafana
  name: monitoring-grafana
  namespace: kube-system
spec:
  # In a production setup, we recommend accessing Grafana through an external Loadbalancer
  # or through a public IP.
  # type: LoadBalancer
  # You could also use NodePort to expose the service at a randomly-generated port
  # type: NodePort
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 32600
  type: NodePort
  selector:
    k8s-app: grafana

EOF
```

influxdb.yaml
```
cat <<EOF > influxdb.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: monitoring-influxdb
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: influxdb
    spec:
      containers:
      - name: influxdb
        image: mirrorgooglecontainers/heapster-influxdb-amd64:v1.3.3
        volumeMounts:
        - mountPath: /data
          name: influxdb-storage
      volumes:
      - name: influxdb-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    task: monitoring
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: monitoring-influxdb
  name: monitoring-influxdb
  namespace: kube-system
spec:
  ports:
  - port: 8086
    targetPort: 8086
  selector:
    k8s-app: influxdb
EOF    
```

### 部署
```
$ kubectl create -f grafana.yaml
$ kubectl create -f heapster.yaml
$ kubectl create -f influxdb.yaml

```
访问http://ip:32600即可访问到grafana ui了


参考资料:
https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/calico
https://jimmysong.io/kubernetes-handbook/practice/coredns.html
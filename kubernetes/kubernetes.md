# Kubernetes

### Core concepts

#### Cluster Architecture

![image-20200311180437752](../.img/image-20200311180437752.png)

**Master** nodes responsible for managing cluster: storing information regarding the different nodes (etcd), scheduling, monitoring containers etc. Master control components:

**Worker** nodes intended for deploying applictions on

**kube-apiserver** responsible for all operations within the cluster

*kubeadm deploys it as a static pod in `/etc/kubernetes/manifests/`*
*Hard way:* install binary and enable `/etc/systemd/system/kube-apiserver.service`

**kube-controller-manager** (replication-controller, node-controller, ) controls that required components are available and restart some of them if they're doesn't.

*kubeadm deploys it as a static pod in `/etc/kubernetes/manifests/`*
*Hard way:* install binary and enable `/etc/systemd/system/kube-controller-manager.service`

**kube-scheduler** choosing right node to deploy pod on by containers requirements such as capacity, available memory, taints & tolerations, node affinity rules etc.

Argument `--leader-elect` used in case of High Availability cluster setup with multiple copies of the sheduler on different masters. When a few copies of the scheduler deployed on each master node at the same time, the cant work together, so one of them should be a `leader`.

*kubeadm deploys it as a static pod in `/etc/kubernetes/manifests/`*
*Hard way:* install binary and enable `/etc/systemd/system/kube-controller-manager.service`

**kubelet** installed on each node, responsible for communication with master, do all node managing jobs (responsible for creating/updating/deleting/monitoring pods, monitoring node).

***kubeadm doesn't automaticly deploy kubelet!***
always install binary and enable `/etc/systemd/system/kubelet.service`

**kube-proxy** process on each node, looks for a new services and creates trafic forwarding rules to pods through the service.

*kubeadm deploys it as a daemonset*
*Hard way:* install binary and enable `/etc/systemd/system/kube-proxy.service`

#### ETCD

*key-value database distributed across **master** nodes. Stores information about what pods on which nodes etc*

* download & extract

    ```bash
    $ curl -L https://github.com/etcd-io/etcd/releases/download/v3.3.11/etcdv3.3.11-linux-amd64.tar.gz -o etcd-v3.3.11-linux-amd64.tar.gz
    $ tar xzvf etcd-v3.3.11-linux-amd64.tar.gz
    ```

* operate

    ```bash
    $ etcd							# run service
    $ etcdctl set key value			# set value
    $ etcdctl get key				# get value
    ```

##### ETCD in Kubernetes

```bash
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
--initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
--listen-peer-urls https://${INTERNAL_IP}:2380 \\
--listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
--advertise-client-urls https://${INTERNAL_IP}:2379 \\ # This is the address on which ETCD listens. It happens to be on the IP of the server and on port 2379, which is the default port on which etcd listens.
--initial-cluster-token etcd-cluster-0 \\
--initial-cluster controller-0=https://${CONTROLLER0_IP}:2380,controller-1=https://${CONTROLLER1_IP}:2380 \\ # this is for High Availability (multiple etcd instances on each master node)
--initial-cluster-state new \\
--data-dir=/var/lib/etcd
```

##### Explore

```bash
$ kubectl exec etcd-master –n kube-system etcdctl get / --prefix –keys-only
/registry/apiregistration.k8s.io/apiservices/v1.
/registry/apiregistration.k8s.io/apiservices/v1.apps
/registry/apiregistration.k8s.io/apiservices/v1.authentication.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.authorization.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.autoscaling
/registry/apiregistration.k8s.io/apiservices/v1.batch
/registry/apiregistration.k8s.io/apiservices/v1.networking.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.rbac.authorization.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.storage.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1beta1.admissionregistration.k8s.io
```

#### Kubernetes resources

**Pods** can run a few containers within the cluster

**ReplicaSets** manage a few instances of the same pod

**Deploymants** provides an ability to do rolling updates on replicasets

**Namespaces** a way to group a set of k8s resources

**Services** make pods reacheble inside and outside the cluster


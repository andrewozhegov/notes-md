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

---

### Scheduling

#### Manual scheduling

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 8080
  nodeName: node02
```

*specify **nodeName** to place pod on a specific node without sheduler*

##### How sheduling works

1.  Scheduler looks for pod definitions **without nodeName** specified

2.  Sheduler indetifies the right node to place pod on, using sheduling algoritms (paying attention to taints & tolerations, nodeAffinity, Resouce requirements & Limits).

3.  Sheduler adds **nodeName** to a pod using **Binding** resource (you cant move already assigned pod to another node this way)

```yaml
apiVersion: v1
kind: Binding
metadata:
  name: nginx
target:
  apiVersion: v1
  kind: Node
  name: node01
```

```bash
curl --header "Content-Type:application/json" --request POST --data '{"apiVersion":"v1", "kind": "Binding“ …. }' http://$SERVER/api/v1/namespaces/default/pods/$PODNAME/binding/
```

#### Labels & Selectors

*standart method of grouping resouces*

##### Add label

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: bee
    ...
```

##### Use selector

```bash
kubectl get pods --selector run=bee
```

*select pod by name in a service defenition*

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  ports:
    - targetPort: 8080
      port: 8080
      nodePort: 30080
  selector:
    name: simple-webapp
```

*also **ReplicaSet** select pods as replicas by labels*

#### Taints & Tolerations

*defines what pods can be scheduled on a node*

*if particular pod doesn't **tolerate** (haven't toleration) with **taint** on a node, it can't be placed on this node*

##### Add Taint to node

```bash
# if pod NOT TOLERATE with node it will not be deployed on this node anyway
# even if pod will wait for node in a Pending state
kubectl taint nodes node01 spray=mortain:NoSchedule
# NOT TOLERATE pod will not be placed on this node if any other candidate is available (system will try to avoid placing pod on this node)
kubectl taint nodes node01 key=value:PreferNoSchedule
# even already deployed NOT TOLERATE pods will be evicted
kubectl taint nodes node01 key=value:NoExecute
```

##### Add toleration to pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  -  image: nginx
     name: nginx
  tolerations:
  - key: "spray"
    operator: "Equal"
    value: "mortein"
    effect: "NoSchedule"
```

**Taints & Toleration will NOT guaratee that Tolerate Pod be plased only on Tainted NODE! (use NodeAffinity for this) Instead it tells the node to only accept pods with certain toleration.**

#### Node Selectors

1.  Add special **label** to node

```bash
kubectl label nodes node01 size=Large
```

2.  Use **nodeSelector** in

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  -  image: nginx
     name: nginx
  nodeSelector:
    size: Large
```

3.  Now `nginx` pod will be places on `node01`

**For more complex conditions like "NOT Large" of "Large OR Medium" etc. use NodeAffinity**

#### Node Affinity

*one more way to control pod placemant on a specific node*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  -  image: nginx
     name: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In # NotIn, Exists, etc.
            values:
            - Large
            - Medium
```

*'requiredDuringSchedulingIgnoredDuringExecution' will not schedule pod if there is no mathing node*
*If NodeAffinity may not match any node better use 'preferredDuringSchedulingIgnoredDuringExecution', in that case if matching node will not fount, NodeAffinity will ignore this rule.*

**Add Node Affinity to Deployment:**

1. Label required node

``kubectl label node node01 color=blue``

2. Set Node Affinity to the deployment to place the PODs on node01 only

add in pod's `spec` section:

```yaml
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: color
                operator: In
                values:
                - blue
```

**NodeAffinity does not guarantee that any other pods will not be placed on the particular node! (use Taints & Tolerations for that)**

#### Resources requirements & Limits

Node resources: CPU, RAM, disc space (0.5 cpu & 256Mi by default)

Default limits: 1vCPU, 512Mi of RAM

**Container can't use more CPU than it's limit! (will cause THROTTLE)**

**Container can use more memory than limiited (but if it will use more memory constantly, pod will be TERMINATED)**

*if there is no node which have required amount of resources, pod will stay in a Pending state*

##### Specify pod resources requirements & Set a limit of resources usage

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  -  image: nginx
     name: nginx
     resources:
       requests:
         memory: "1Gi"
         cpu: 1
       limits:
         memory: "2Gi"
         cpu: 2
```

#### DaemonSets

*runs an instance of the pod on each node in the cluster*

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: replicaset-1
spec:
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
```

#### Static Pods

**kubelet** can manage node independently (without apiserver) by creating **pods** from manifest files in a specific manifest folder (works only for pods).

How to get manifest folder for static pods:
`cat $(ps -aux | grep kubelet | grep -Po '(?<=\-\-config=)\S+') | grep staticPodPath | awk '{print $2}'` or look for a `staticPodPath` in the kubeconfig file (specified in a `--config` argument for kubelet).

#### Multiple Schedulers

*for specific applications witch requires some specific checks for choosing a node to be placed on*

1.  Custom-sheduler should have a unique `--scheduler-name` specified as argument.
2.  Set argument `--leader-elect=false` to get multiple schedulers working (if there is not a High Availability cluster setup).
3.  For HA cluster with multiple masters set `--lock-object-name=sheduler-name` to differentiate the new custom scheduler from the default during the leader election process.
4.  Deploy as pod
5.  Specify custon scheduler for the pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  -  image: nginx
     name: nginx
  schedulerName: my-scheduler # here it is
```

*if scheduler isn't working correct pod will continue in a **Pending** state*

6.  Check what scheduler been used to pick a node for pod

```bash
kubectl get events
kubectl logs --namespce=kube-system my-scheduler
```

---

### Logging & Monitoring

#### Monitoring

*monitoring resource consumption*

**Metrics:**
**node-level** - number of nodes, healthy nodes, performance mertics (cpu, memory, disk, network)
**pod-level** - num of pods, healhy pods, pods performance

**Monitoring solutions:** Metrics Server, Elastic Stack, Prometheus

##### Metrics-server

*one server per cluster*
*retrieves performance data from cluster components, aggregates it and stores in memory*
*in-memory solution (doesn't store data history on the disk)*
*uses **kubelet** as agent for retrieving metrics data*

The **kubelet** also contains a subcomponent known as as cAdvisor or Container Advisor. cAdvisor is responsible for retrieving performance metrics from pods, and exposing them through the kubelet API to make the metrics available for the Metrics Server.

GitHub: https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/metrics-server, https://github.com/kubernetes-sigs/metrics-server

```bash
kubectl top pods
kubectl top node
```

#### Logs managing

```bash
kubectl logs -f -namepod.container-name
```

---

### Lifecycle Management

#### Rolling Update & Rollback

##### Rollout and Versioning

*when you first create a deployment it triggers a rollout a new rollout creates a new deployment revision*

*when the container version is updated to a new one a new rollout is triggered and a new deployment revision is created*

![image-20200404192550005](../.img/image-20200404192550005.png)

```bash
# get rollout status
kubectl rollout status deployment/nginx

# get rollout history
kubectl rollout history deployment/nginx
```

##### Deplyment Strategy

**Recreate** - destroy all replicas with old versions and then create all new ones (applications unavailable due upgrading).
*scaling replicaset to 0, and then scale it back to required amount*

**Rolling Update** - destroy old version pod and bring a new version one by one (default).
*creates a new replicaset inside deployment. one step is scale old replica down and scale new replica up. and repeat for each pod in a replica.*

##### Run version upgrage

```bash
# change manifest and apply it or set new image directly in deployment
kubectl set image deployment/nginx-deployment <container-name>=nginx:1.9.1 --record
```

##### Rollback

*undo upgrade. get old versions of pods back*

```bash
kubectl rollout undo deployment/nginx
```

#### Commands & Arguments

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  -  image: nginx
     name: nginx
     command: # same as Dockerfiles entrypoint
     - sleep
     args:    # same as Dockerfiles ARGS
     - 10
```

#### Environment variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  -  image: nginx
     name: nginx
     env:
     - name: HADCODE
       value: val
     - name: FROM_CONFIGMAP
       valueFrom:
         configMapKeyRef:
           name: my-configmap
           key: FROM_CONFIGMAP
     - name: FROM_SECRET
       valueFrom:
         secretKeyRef:
           name: my-secret
           key: FROM_SECRET
```

#### ConfigMaps

*used to store configuration data in the form of key value pairs in Kubernetes*

##### Create ConfigMap

```bash
kubectl create configmap <config-name> \
    --from-literal=<key>=<value>
kubectl create configmap <config-name> \
    --from-file=app-config.properties
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-config
data:
  FROM_CONFIGMAP: value
  SECOND_ENV: value2
```

##### Use ConfigMap in manifest file

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  -  image: nginx
     name: nginx
     envFrom: 				# use all env variables from configmap
     - configMapRef:
       name: game-config
     env:
     - name: FROM_CONFIGMAP # get single env from configmap
       valueFrom:
         configMapKeyRef:
           name: game-config
           key: FROM_CONFIGMAP
     volumes:				# volume inside the container as a file
     - name: game-config
       configMap:
       - name: game-config
```

#### Secrets

*ConfigMaps store data in a plain text format, so it defenetely not the good place to store credentials*

*Secrets stored data in encoded format*

##### Create Secret

```bash
kubectl create secret generic <secret-name> \
    --from-literal=<key>=<value>
kubectl create secret generic <secret-name> \
    --from-file=app-secret.properties
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: game-secret
data: # values encoded by base64
  FROM_CONFIGMAP: dmFsdWUy # echo -n "value" | base64
  SECOND_ENV: dmFsdWU=
```

```bash
$ # decode secret value
$ echo -n "dmFsdWUy" | base64 --decode
value
```

##### Use Secret in manifest file

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  -  image: nginx
     name: nginx
     envFrom: 				# use all env variables from secret
     - secretRef:
       name: game-secret
     env:
     - name: FROM_SECRET # get single env from secret
       valueFrom:
         secretKeyRef:
           name: game-secret
           key: FROM_SECRET
     volumes: # volume as a files (will create one file per each key)
     - name: game-secret
       secret:
       - name: game-secret
```

There are other better ways of handling sensitive data like passwords in Kubernetes, such as using tools like Helm Secrets, [HashiCorp Vault](https://www.vaultproject.io/).

#### Scale

```bash
kubectl scale --replicas=5 replicaset new-replica-set
kubectl scale deployment httpd-frontend-2 --replicas=3
```

#### Multi-Container pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: yellow
spec:
  containers:
  - name: lemon
    image: busybox
  - name: gold
    image: redis
```

**Usecase:**

pod: *container1-app,* **container2-elasticsearch-loggingagent -> elasticsearch -> kibana**

#### Init containers

When a POD is first created the initContainer is run, and the process in the initContainer must run to a completion before the real container hosting the application starts. Each init container is run one at a time in sequential order. If any of the initContainers fail to complete, Kubernetes restarts the Pod repeatedly until the Init Container succeeds.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

#### Self Healing Applications

Kubernetes supports self-healing applications through **ReplicaSets** and **Replication Controllers**. The replication controller helps in ensuring that a POD is re-created automatically when the application within the POD crashes. It helps in ensuring enough replicas of the application are running at all times.

Kubernetes provides additional support to check the health of applications running within PODs and take necessary actions through **Liveness** and **Readiness** Probes.

---

### Cluster Maintenance

#### OS Upgrade

0. Make all critical nodes unscdulebale (for no schedule evicted apps on it)

```bash
kubectl cordon node03
```

1. Make `node` unschedulable & evict all pods from there

```bash
kubectl drain node01 --ignore-daemonsets
```

drain node with daemonsets deployed on it

**All pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet must be `--force`d**:

```bash
kubectl drain node02 --ignore-daemonsets --force
```

(after that unmanaged pods will be lost forever)

2. Do any required OS updates

3. Make `node` schedulable again

```bash
kubectl uncordon node01
```

(after that only new pods will be scheduled on node01, current pods will not be placed back immediately when node available again)

#### Cluster Upgrade

Current version of cluster: `kubectl get node -owide`

Upgrade one node at a time while moving all workloads from it to another node.

Get latest available version: `kubeadm upgrade plan`

Do upgrades across all minor versions until reach latest.

**Upgrading process:**

1. Make upgradeble `node` unschedulable:

```bash
kubectl drain master --ignore-daemonsets
```

2. Upgrade `kubeadm` tool

```bash
sudo apt policy kubeadm | grep '1.17' # get full version string
sudo apt-get install kubeadm=1.17.0-00 --no-install-recommends
```

3. Apply new cluster version

```bash
kubeadm upgrade apply v1.17.0
```

4. Upgrade `kubelet`

```bash
sudo apt install kubelet=1.17.0-00
```

5. Make `node` schedulable back

```bash
kubectl uncordon master
```

6. Upgrade another nodes:

Drain node
SSH into it
Upgrade `kubeadm` and `kubelet`
Logout
Upgrade `node` from master:

```bash
kubeadm upgrade node --kubelet-version v1.17.0
```

Uncordom node: `kubectl uncordon node01`

#### Backup & Restore

Get all required information about etcd from Command: `kubectl describe pods etcd-master`

(**listen-client-urls, peer-trusted-ca-file, cert-file, key-file**)

* Do backup `etcd`:

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save /tmp/snapshot-pre-boot.db
```

* Restore from backup file

*additional fields:* node-name, desctination data-dir, initial-cluster, initial-cluster-token, initial-advertise-peer-urls

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --name=master --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --data-dir /var/lib/etcd-from-backup --initial-cluster=master=https://127.0.0.1:2380  --initial-cluster-token etcd-cluster-1 --initial-advertise-peer-urls=https://127.0.0.1:2380   snapshot restore /tmp/snapshot-pre-boot.db
```

* Modify */etc/kubernetes/manifests/etcd.yaml*

update etcd static pod to use new data dir, initial-cluster-token, volumes & volume mounts

---

### Security

#### Authentication

**Role Based Access Contrlos** - users associated with groupds with specific permissions.

**Users** (Admins & Developers) - access (through kubectl or api) managed by `kube-apiserver`.

* list of user & passwords in a static password file

specify `csv` file in a `--basic-auth-file` field on `kube-apiserver` ExecStart cmd:

```csv
pass1,user1,u001,group1
pass2,user2,u002
```

add volume & volumeMout this file to `kube-apiserver`

```bash
curl -v -k https://master-node-ip:6443/api/v1/pods -u "user1:pass1"
```

* useranme & tokens in a static token file

```csv
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX,user1,u001,group1
```

add volume & volumeMout this file to `kube-apiserver`

specify filename in `--token-auth-file` arg

use token:

```bash
curl -v -k https://master-node-ip:6443/api/v1/pods --header "Authorization: Bearer XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
```

* certificates

```bash
curl https://master-node-ip:6443/api/v1/pods \
--key admin.key \
--cert admin.crt \
--cacert ca.crt
```

```bash
kubectl get pods \
--server master-node-ip:6443 \
--client-key admin.key \
--client-certificate admin.crt \
--certificate-authority ca.crt
```

also can be done by **KubeConfig**

```bash
kubectl get pods \
--kubeconfig admin-kubeconfig.yaml \
```

* third-party auth services (LDAP etc.)

**Services** (External programs) - manages through **ServiceAccounts**

#### TLS

By default all crts located in `/etc/kubernetes/pki`

**TODO: Photo**

* Create certificate:

1. Generate private and public keypair (public key names: *.key* or *-key.pem*)

```bash
openssl genrsa -out my.key 1024				# public
openssl rsa -i my.key -pubout > my.pem		# private
```

2. Create Certificate Signing Request

```bash
openssl req -new -key my.key -out my.csr -subj "/C=US/ST=CA/O=MyOrg, Inc./CN=my-key.com"
```

3. Sign public key with CSR to get a CRT

```bash
openssl x509 -req -in /etc/kubernetes/pki/apiserver-etcd-client.csr \
-CA /etc/kubernetes/pki/etcd/ca.crt \
-CAkey /etc/kubernetes/pki/etcd/ca.key \
-CAcreateserial \
-out /etc/kubernetes/pki/apiserver-etcd-client.crt
```

* Get information about `crt`:

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text
```

**Fix some of the TLS issues**

* check `etcd` logs (since kubectl doesn't have connection to server, use docker itself)

```bash
$ docker logs k8s_etcd_etcd-master_kube-system_49066928fac6c601adc44cc0d3c60674_4
```

`2020-03-15 08:53:33.253857 C | etcdmain: open /etc/kubernetes/pki/etcd/server-certificate.crt: no such file or directory` means that file doesn't exist or broken

**Fix**: Specify corrent path to file in `/etc/kubernetes/manifests/etcd.yaml`

* **etcd** logs `2020-03-15 09:02:30.385037 I | embed: rejected connection from "127.0.0.1:57620" (error "remote error: tls: bad certificate", ServerName "")`

    **apiserver** logs: `W0315 09:04:34.564733       1 clientconn.go:1120] grpc: addrConn.createTransport failed to connect to {https://127.0.0.1:2379 0  <nil>}. Err :connection error: desc = "transport: authentication handshake failed: x509: certificate signed by unknown authority". Reconnecting...`

Check sertificates last update datetime: `ls -la /etc/kubernetes/pki`

Check manifests last update datetime: `ls -la /etc/kubernetes/manifests`

**Fix**: `kube-apiserver.yaml`

* **etcd** logs: `2020-03-15 09:16:45.660414 I | embed: rejected connection from "127.0.0.1:60204" (error "tls: failed to verify client's certificate: x509: certificate has expired or isnot yet valid", ServerName "")` mans that `crt` is expired

**Fix:** Check all certificates date and resign the expired ones!

#### TLS in Kubernetes

![image-20200315132918811](../.img/image-20200315132918811.png)

![image-20200315132847694](../.img/image-20200315132847694.png)

#### Certificates API

the process of signing new certificates for new admins

`CertificateSigningRequest` can be `Reviewd` and `Approved` ba admins using kubectl

**Controller Manager** does all this by csr-approving, csr-signing managers.

**Process:**

1. new admin `jane` creates private key and CSR

```bash
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr
```

2. admin creates  `CertificateSigningRequest`

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: $(cat jane.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
```

3. List all requests

```bash
kubectl get csr
```

4.  Approve request

```bash
kubectl certificate approve jane
```

#### KubeConfig

file with specified certificates for auth

by default k8s looking for this file in `~/.kube/config`

**Clusters** list of available clusters (url)

**Users** list of users with different privileges (keys & crts) (by **RBAC**)

**Contexts** - defines witch user accont wiill be used to whick cluster

```yaml
apiVersion: v1
kind: Config
current-context: admin@clister-playground # default
clusters:
- name: clister-playground
  cluster:
  	certification-authority: /etc/kubernetes/pki/ca.crt
  	# or certification-authority-data with base64 encoded CRT
  	server: https://clister-playground:6443
contexts:
- name: admin@clister-playground
  context:
    cluster: clister-playground
    user: admin
    namespace: my-namespace
users:
- name: admin
  user:
    client-certificates: /etc/kubernetes/pki/admin.crt
    client-key: /etc/kubernetes/pki/admin.key
```

```bash
kubectl config view
kubectl config use-context admin@clister-playground # change default context
```

#### API Groups

![image-20200315152933490](../.img/image-20200315152957339.png)

![image-20200315153034357](../.img/image-20200315153034357.png)

https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13

#### Role Based Access Controls (RBAC): Role & RoleBinding

*always related to namespace (default by default)*

* Create role for users

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  resourceNames: ["blue"] # user can read only 'blue' pod
  verbs: ["get", "watch", "list"]
```

* Link user to this role using `RoleBinding`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "jane" to read pods in the "default" namespace.
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

* View roles & rolebindings

```bash
kubectl get roles
kubectl get rolebindings
kubectl describe role pod-reader
kubectl describe rolebinding read-pods
```

* Check users abilities (can `jane` delete pods in a test namespace?)

```bash
kubectl auth can-i delete pods --namespace test
kubectl get pdos --as jane
```

#### RBAC: ClusterRole & ClusterRoleBinding

*do not related to namespace unlike Roles&Rolebindings*

**ClusterRoles** to manage Cluster wide resources like nodes can't be associated with particular namespaces. (Nods, Persistant Volumes, ClusterRoles, ClusterRoleBindings, Certificate Signing Requests, Namespaces)

```bash
kubectl api-recources --namespaced=false # get cluster scoped resources
```

**Example:** view, update and delete nodes.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-manager
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```

* `ClusterRoleBinding` to link user with `ClusterRole`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jane-node-manager
subjects:
- kind: User
  name: jane # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-manager
  apiGroup: rbac.authorization.k8s.io
```

*Atlso can be used to get user access to manage resource across all namespaces!*

#### Image security

*configuring pods to use images from a secure private container registries*

* Docker approach

Login into a private registry at first:

```bash
docker login private-registry.io
docker run private-registry.io/apps/private-image
```

* Kubernetes approach

secret with private registry credentials

```bash
kubectl create secret docker-registry private-registry-io \
	--docker-server=private-registry.io \
	--docker-username= \
	--docker-password= \
	--docker-email=
```

then specify secret with special credentials under `imagePullSecrets`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  -  image: private-registry.io/apps/nginx
     name: nginx
  imagePullSecrets:
   - name: private-registry-io
```

#### Security contexts

*some additional container run args such as linux capabilities or user specifying can be configured in k8s through `Security Contexts`*

`Linux capabilities` allow you to break apart the power of root into smaller groups of privileges (http://man7.org/linux/man-pages/man7/capabilities.7.html).

* Pod level (will impact all containers in pod)
* Container level (will rewrite pod-level contexts)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  securityContext: # pod-level conf
    runAsUser: 1000
  containers:
  -  image: ubuntu
     name: ubuntu
     command: ["sleep", "3600"]
     securityContext: # container-level conf
      runAsUser: 0 # rewrite
      capabilities:
        add: ["MAC_ADMIN"]
```

#### Network policy

*can allow traffic reach some pod only through another specific pod*

* **Ingress** - income traffic
* **Egress** - outgoing traffic

**Example:** allow trafic to reach the DB pod only through DB-API pod.

By default k8s configured with `All Allow` rule to make pods available from any another pods or services.

**Network Policy** rule example: only allow traffic on 3306 port from the DB-API pod.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels: # select the required pod
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: db-api
    ports:
    - protocol: TCP
      port: 3306
  egress: # multiple
  - to:
    - podSelector:
        matchLabels:
          name: mysql
    ports:
    - protocol: TCP
      port: 3306
  - to:
    - podSelector:
        matchLabels:
          name: nginx
    ports:
    - protocol: TCP
      port: 8080
```

*Flannel doesnt support network policies!*


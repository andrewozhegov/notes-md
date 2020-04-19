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

---

### Storage

#### Storage in Docker

* Storage Drivers

*helps manage storage in containers*

All docker related data stores in `/var/lib/docker`.

**Drivers:** AUFS, ZFS, BTRFS, Device Mapper, Overlay

* Volume Driver Plugins

**Plugins:** Local, Azure File Storage, Convoy, DigitalOcean Block Storage, Flocker, gce-docker ...

```bash
docker run -it \
    --volume-driver rexray/ebs \
    ...
```

*to provision a volume from Amazon EBS. save your data from container to cloud storage after exit*

#### Container Storage Interface (CSI)

*Kubernetes do not depends on Docker. It supports many other container runtime engines! CSI defines rules of communication between CRE and Orchestration tools.*

#### Volumes

*data generated by the pod can be stored in a volume (same as volumes for containers)*

Many types of volumes are described here: https://kubernetes.io/docs/concepts/storage/volumes/

*   **hostPath** mounts a file or directory from the host node’s filesystem into your Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /data
      # this field is optional
      type: Directory
```

#### Persistent Volumes

*a cluser wide pull of storage to be used by users deploing applications using Persistent Volume Claims*

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
    #- ReadOnlyMany
    #- ReadWriteMany
  capacity:
    storage: 5Gi
  nfs: # type
    path: /tmp
    server: 172.17.0.2
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle # what will happen to PV when it's claim deleted (Reatin by default)
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-log
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 100Mi
  hostPath:
    path: /pv/log
```

```bash
kubectl get pv
```

#### Persistent Volume Claims

*uses to get a peace of storage from Persistant Volume*

**PV** creates by admin
**PVC** creates by users

PVC will be in a Pending state until the new volumes as required will be available!

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-log-1
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Mi
```

```bash
kubectl get pvc
```

**Using PVC in a Pod manifest:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

---

### Networking

#### Switching Routing

*swith creating a network between machines*

![image-20200318130414061](../.img/image-20200318130414061.png)

* get list of interfaces

*eth0 will be used to connect to the switch*

```bash
ip link
```

* connect to 192.168.1.0 network

assign 192.168.1.10 address to eth0 interface (which is connected to network)

```bash
ip addr add 192.168.1.10/24 dev eth0
ip addr # get addresses assigned to interfaces
```

**Set those changes in /etc/network/interfaces to make it reboot safe!** 

* do the same from another machine in the network ...

* now machines can ping each other

```bash
ping 192.168.1.11
```

##### Communication between networks

*Router helps connect two networks together*

![image-20200318130942762](../.img/image-20200318130942762.png)

* get routing table

```bash
ip route
```

*if there is no any routes, it means that machines in the current network can not reach machines in another network **(Gateway requred)***

**Gateway**  - specify how can machine from current network reach machines from another network. **Example for B:**

```bash
ip route add 192.168.2.0/24 via 192.168.1.1
```

*means B can reach 192.168.2.0 by using 192.168.1.1 gateway (router)*

##### Internet access

*any network doesn't exist in current network goes through **Default Gateway***

**Example:** if there is another router with internaet access, configure default gateway like this

```bash
ip reute add default via 192.168.2.2
```

***Default Gateway is goot place to start network troubleshooting!***

##### Setup linux host as a router

![image-20200318133809964](../.img/image-20200318133809964.png)

```bash
# host A (already in network with B thrue switch)
ip route add 192.168.2.0/24 via 192.168.1.6
```

```bash
# host C (already in network with B thrue switch)
ip route add 192.168.1.0/24 via 192.168.2.6
```

```bash
# host B
# allow forwarding packages from eth0 to eth1
echo 1 > /proc/sys/net/ipv4/ip_forward # untill next reboot
# cat /etc/sysctl.conf | grep ip_forward # better to change here
```

#### DNS

![image-20200318141032131](../.img/image-20200318141032131.png)

```bash
# host A
echo "192.168.1.11 db" >> /etc/hosts
ping db # success
```

Network will grows and it will be uncomfortable to add new entries in host files on each host every time new host will be added to network. Also changins any hosts IP will require to add new ip on every other **/etc/hosts**.

**DNS Server** - a machime in the network which stores all dns records, however others hosts just know dns-servers ip.

*/etc/resolv.conf*

```
nameserver 192.168.1.100
Forward All to 8.8.8.8 # forward all other unresolved dns names to google
```

*Local **/etc/hosts** preffered by default! (can be changed in /etc/nsswitch.conf)*

##### Configure local dns server to resolve names wothout highlevel domain names

**web.myconpany.com** available as just **web** inside mycompany's network

*/etc/hosts*

```
search mycompany.com
```

##### Record types

![image-20200318143120918](../.img/image-20200318143120918.png)

##### Check DSN server

```bash
nslookup www.google.com
dig www.google.com
```

#### CoreDNS

https://github.com/coredns/coredns

```bash
./coredns # by default will listens on port 53
```

##### specifying hosts in *Corefile*

```
. {
  hosts /etc/hosts
}
```

https://github.com/kubernetes/dns/blob/master/docs/specification.md
https://coredns.io/plugins/kubernetes/

#### Network Namespaces

*uses to hide (separate) the processes from the host or from each others (network isolation)*
*root user can see processes from containers by `ps aux` (with different pid)*

Address Resolution Protocol (**ARP**) is the method for finding a host's Link Layer (**MAC address**) when only its IP address is known. The **ARP table** is used to maintain a correlation between each **MAC address** and its corresponding IP address.

**Namespace can have it's own virtual interfaces, routing tables and ARP tables.**

```bash
ip netns add my-ns # create new namespace
ip netns # list namespaces

# host's interfaces isn't available from namespace!
ip netns exec my-ns ip link # list interfaces inside the ns
ip -n my-ns link # same

# Host's ARP entries also isn't available from the NS
ip netns exec my-ns arp

# Host's routes isn't available from the NS
ip netns exec my-ns route
```

##### Connect two namespaces with virtial network

*using virtual ethernet (pipe)*

```bash
# we already have two namespaces: 'red' and 'blue'

# 1. create virtual ethernet cable
# (name it's ends/interfaces as veth-red & veth-blue)
ip link add veth-red type veth peer name veth-blue

# 2. attach each veth interface to the appropriate NS
ip link add set veth-red netns red
ip link add set veth-blue netns blue

# 3. assign IP to each of this namespaces
ip -n red addr add 192.168.15.1 dev veth-red
ip -n blue addr add 192.168.15.2 dev veth-blue

# 4. bring links UP
ip -n red link set veth-red up
ip -n blue link set veth-blue up

# now namespaces identifies each other with ARP entries
```

##### Connect more than two namespaces using bridge

*using virtual switch*

There is many solutions to create a vrtual switch: **Linux Bridge** (native), Open vSwitch, etc.

```bash
# add new interface to the host (bridge)
ip link add v-net-0 type bridge

# bring new interface up
ip link set dev v-net-0 up

# delete link from red namespace (autodeletes another veth-blue)
ip -n red link del veth-red

# connect namespaces to this new virtual switch
ip link add veth-red type veth peer name veth-red-br
ip link add veth-blue type veth peer name veth-blue-br

# attach veth interfaces to the namespaces
ip link add set veth-red netns red
ip link add set veth-blue netns blue

# attack veth's br interfaces to the bridge
ip link set veth-red-br master v-net-0
ip link set veth-blue-br master v-net-0

# set IPs to this links
ip -n red addr add 192.168.15.1 dev veth-red
ip -n blue addr add 192.168.15.2 dev veth-blue

# bring interfaces up
ip -n red link set veth-red up
ip -n blue link set veth-blue up
```

![image-20200318163812716](../.img/image-20200318163812716.png)

##### Virtual Network available from the host

```bash
# make all namespaces from the v-net-0 available from host
# just from the host assign an ip address to v-net-0 interface
ip addr add 192.168.15.5/24 dev v-net-0
```

##### Make Vitrual Network available from outside the host

*from another host in LAN connected through **eth0***

![image-20200318165532299](../.img/image-20200318165532299.png)

*Use host as Gateway to outsde through the bridge*

```bash
ip netns exec blue ip route add 192.168.1.0/24 via 192.168.15.5
```

*Now make blue NS available from 192.168.1.3:*

Host network have it's own internal priveate address (192.168.15.2), and destination network (192.168.1.3) don't know adout it, so it can't reach back.

We need NAT enabled on the host acting as the Gateway. So it can send messages to the LAN in it's own name with it's own address.

```bash
iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE
```

Add a new rule in the NAT IP table in the POSTROUTING chain to masquerade or replace the from address on all packets coming from the source network 192.168.15.0 with its own IP address.

**That way anyone receiving these packets outside the network will think that they are coming from the host and not from within the namespace.**

**Or basically add a route entry to the second host telling that host that the network 192.168.15 can be reached through the host at 192.168.1.2.**

##### Allow namespaces to reach the Internet

*add default Gateway to host (use host to reach any external network)*

```bash
ip netns exec blue ip route add default via 192.168.15.5
```

#### Docker Networking

```bash
docker run --namespace none nginx	# conpletely isolated from host
docker run --namespace host nginx	# attached to host network
docker run --namespace bridge nginx	# creation internal bridge
```

##### Docker Bridge Network

172.17.0.0 by default: **docker0**

![image-20200318180653881](../.img/image-20200318180653881.png)

```bash
# docker creates namespaces for containers
# get namespace from containes (SandboxID)
docker inspect <container>
```

*container(ns) connects to the dokcer0 interface through veth*

*every NS have it's ip in docker0 virtual network*

##### Docker containers ports mapping to outside

```bash
docker run -p 8080:80 nginx
```

![image-20200318180921320](../.img/image-20200318180921320.png)

*Docker adds the rule to the docker chain and sets destination to include the containers IP as well you can see the rule docker creates when you list the rules in iptables.*

```bash
iptables \
	–t nat \
	-A PREROUTING \
	-j DNAT \
	--dport 8080 \
	--to-destination 80
	
iptables \
	–t nat \
	-A DOCKER \
	-j DNAT \
	--dport 8080 \iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE
	--to-destination 172.17.0.3:80
```

#### CNI

##### Creating Namespace Network

1.  Create network NAMESPACE
2.  Create BRIDGE/interface
3.  Create VETH pairs (PIPE)
4.  Attach VETH to NAMESPACE
5.  Attach other VETH  to BRIDGE
6.  Assign IP address with connected NAMESPACES
7.  Bring interfaces UP
8.  Enable NAT - IP Masquerade

*This steps same for Docker, Mesos, Kubernetes etc.*

*So 2 - 8 steps are moved into a single prigramm called **Bridge**.*

*Docker & Kubernetes are only creates a Namespace and then run a **Bridge** programm.*

##### Add container to a particular namespace:

```bash
bridge add 2e34dcf34 /var/run/netns/2e34dcf34
```

*Whenever rkt or kubernetes creates a new container, they call the bridge plugin and pass the container id and namespace to get networking configured for that container.*

##### Container Network Interface

**CNI** - a set standards that define how programs should be developed to solve networking challenges in a container runtime environment.

*Bridge programm is a plugin for CNI*

![image-20200319092116453](../.img/image-20200319092116453.png)

*Docker doesn't implements **CNI** it comes with it's own set of standarts - **CNM** (Container Network Model).*

So Kubernetes creates Docker containers with `none` network:

```bash
docker run --network=none nginx
```

And then run the configured CNI plugin to take care of the rest of configuration:

```bash
bridge add 2e34dcf34 /var/run/netns/2e34dcf34
```

#### Cluster Networking

*networking configuration on a cluster nodes*

##### Required configuration

Each node must have at least one interface connected to a network with IP address assigned.

Host must have a unique hostname and MAC address.

**Required open ports**

![image-20200319224046011](../.img/image-20200319224046011.png)

*ETCD server listens on port 2379.*
*For multiple masters also you need additional port 2380 open so the ETCD clients can communicate with each other.*

[List of required open ports](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports)

#### Pod Networking

*All pods should have assigned IP addresses. and be able to communicate with any pods within the cluster on the same node and on the other nodes without any NAT riles.*

```bash
ip link add v-net-0 type bridge
ip link set dev v-net-0 up
ip addr add 192.168.15.5/24 dev v-net-0

ip link add veth-red type veth peer name veth-red-br
ip link set veth-red netns red
ip link set veth-red-br master v-net-0

ip -n red addr add 192.168.15.1 dev veth-red
ip -n red link set veth-red up

ip netns exec blue ip route add 192.168.1.0/24 via 192.168.15.5

iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE 
```

**Workflow**

![image-20200320001120138](../.img/image-20200320001120138.png)


```bash
# create virtual bridge interface v-net-0 on each node
ip link add v-net-0 type bridge

# up virtual interface on each node
ip link set dev v-net-0 up

# assign an ip address to bridge on each node
ip addr add 10.244.1.1/24 dev v-net-0
```

./net-script.sh (for adding a namespace to a network)

```bash
# for each container (namespace)
# * create veth pair (pipe)
ip link add veth-red type veth peer name veth-red-br
# * attach veth pair
ip link set veth-red netns red
ip link set veth-red-br master v-net-0
# * assign ip address to namespace
ip -n red addr add 192.168.15.1 dev veth-red
# * add default Gateway inside red NS through
ip -n red route add default via 192.168.15.1
# * bring up interface
ip -n red link set veth-red up
```

```bash
# add route to v-net-0 on another node (for each another node)
# we can reach namespace 10.244.1.2 through 192.168.1.11 node
ip route add 10.244.1.2 via 192.168.1.11

# so many configuration when cluster grows
# better configure BRIDGES to use host interface as a default gateway
```

**Bring script to a necessary interface (ADD and DEL cmds)**

*kubelet runned with `--cni-conf-dir`*

*kubelet get's script path from `--cni-bin-dir`*

*and then runs script with add command uses container and namespace as arguments*

![image-20200320092343729](../.img/image-20200320092343729.png)

#### CNI in Kubernetes

![image-20200320093348499](../.img/image-20200320093348499.png)

`/opt/cni/bin` contains bridge programm

`/etc/cni/net.d` contains json bridge confiuration

`/etc/cni/net.d/10-bridge.conf`

```json
{
    "cniVersion": "0.2.0",
    "name": "mynet",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": { 
        "type": "host-local",
        "subnet": "10.22.0.0/16", 
        "routes": [ 
            { 
                "dst": "0.0.0.0/0" 
            } 
        ] 
    } 
}
```

##### CNI Weave

*Weave CNI place an agent deploymant on each node in the cluster*

*all agents connected and communicate between each other*

*every agent knows about pods on it's node*

![image-20200325230337942](../.img/image-20200325230337942.png)

##### How package goes from pod11 on node1 to pod21 on node2

1.  pod11 send package (specifying destination pod: pod21) to weave-agent1 on it's node (node1)
2.  weave-agent1 on node1 knows that weave-agent2 on node2 responsible for recieving packages for pod21
3.  weave-agent1 wrap package for pod21 in a new package (specify weave-agent2 as destination)
4.  weave-agent2 open recieved pacakge and gets the original package for pod21
5.  weave-agent2 send package to pod21 on it's node

##### Weave in details

![image-20200325231158360](../.img/image-20200325231158360.png)

*   *weaveworks agent service deployed on each node*
*   *each agent stores the topology of the entire setup (where the pods and ip of them)*
*   *weave create it's own bridge on each node*

##### Deploy Weave CNI

https://www.weave.works/docs/net/latest/kubernetes/kube-addon/

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

#### IP Address Management in CNI

*Nodes ip manages by external ipam.*

*CNI **bridge program** responsible for assigning ip to node's bridge and pods.*

*CNI stores list of ip like (ip, status, pod) using build-in plugins: **DHCP**, **host-local***

![image-20200326091543006](../.img/image-20200326091543006.png)

`/etc/cni/net.d/net-script.conf` 

```yaml
{
	cniVersion": "0.2.0",
	"name": "mynet",
	"type": "net-script",
	"bridge": "cni0",
	"isGateway": true,
	"ipMasq": true,
	"ipam": {
		"type": "host-local", 			# build-in plugin
		"subnet": "10.244.0.0/16",		# ip range for pods
		"routes": [
			{ "dst": "0.0.0.0/0" }		#
		]
	}
}
```

*weave cni does it automatocly*

#### Service Networking

*pods do not communicate between each other directly*

*to allow any trafic from outside reach the pod, the pod needed to be exposed with **service***

*pod1 can reach other pod2 through it's **service-pod2** ip or name*

*service is accessible from any part of the cluster (service hosts across the cluster)* 

##### Kinds of Services

1.  **ClusterIP** - a service accessible only within the cluster, allocates an internal IP that other components can use to access pod.
2.  **TargetPort** - a service accessible only within the cluster, make application **listen port** available to other components through **service-port**
3.  **NodePort** - works as ClusterIP, but also exposes apllication port as service-port [30000; 32767] on all the nodes and makes application available from outside the cluster.
4.  **External IPs**
5.  **LoadBalancer**
6.  **Ingress**

##### kube-proxy

*watches the changes in the cluster through kube-apiserver, and every time a new service is to be created, kube-proxy assign an IP address from a pre-defined range*

![image-20200326191720782](../.img/image-20200326191720782.png)

The kube-proxy components running on each node, get’s that IP address and creates forwarding rules on each node in the cluster, saying any traffic coming to this IP (IP of the service), should go to the IP of the POD.

Whenever services are created or deleted the kube-proxy component creates or deletes these rules.

##### kube-proxy modes

1.  userspace - listens on a port for each service and proxies connections to the pods;
2.  iptables (default) - using iptables rules;
3.  ipvs - creating ipvs rules.

```bash
kube-proxy --proxy-mode [userspace|iptables|ipvs]
```

##### kube-proxy iptables example

Set ip range to 10.96.0.0 - 10.111.255.255:

``` bash
kube-apiserver --service-cluster-ip-range=10.96.0.0/12 ...
```

*ip ranges for pods and for services shouldn't have same ips*

Find iptables rules created by kube-proxy for services (in case `my-service`):

```bash
iptables -L -t net | grep my-servise
```

*because all iptables rules for services commented with service name*

```bash
# nodes ip range from node interfaces configuration
ip addr
# pods ip range from cni configuration
kubectl logs weave-net-xxxxx weave -n kube-system | grep ipalloc-range
# services ip range from kube-apiserver run arguments
ps aux | grep apiserver | grep ip-range
# kube-proxy mode from kube-proxy container logs
kubectl logs kube-proxy-ddrrw -n kube-system
```

#### kube-dns

*allow name resolution for services and pods within the cluster*

When a service is created `kube-dns` creates a DNS record and maps service-name to it's address. Then any pod in cluster can reach service using it's name.namespace (if different). 

DNS disabled for pods by default.

![image-20200326235738251](../.img/image-20200326235738251.png)

#### CoreDNS in Kubernetes

*deploys as a deployment in a `kube-system` namespace*

`/etc/coredns/Corefile`  is **ConfigMap**

```json
.:53 {
	errors
	health
	kubernetes cluster.local in-addr.arpa ip6.arpa { # specify high domain
		pods insecure 					     # enable dns records for pods
		upstream
		fallthrough in-addr.arpa ip6.arpa
	}
	prometheus :9153
	proxy . /etc/resolv.conf
	cache 30
	reload
}
```

##### Add coredns as nameserver to new pods

*coredns exposes with service called **kube-dns***

*kube-dns adds as a nameserver to a new pods automaticly by **kubelet**:*

`/var/lib/kubelet/config.yaml`

```yaml
...
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
...
```

#### Ingress

**Bad solution**

![image-20200327235322908](../.img/image-20200327235322908.png)

*using a lot of load-balancers (witch is too expencive)*

*requires reconfiguring services every time a new application added*

**Ingress solution**

Ingress helps your users access your application using a single Externally accessible URL, that you can configure to route to different services within your cluster. And implemet SSL security as well.

![image-20200328000655117](../.img/image-20200328000655117.png)

##### Ingress in close look

![image-20200328001548040](../.img/image-20200328001548040.png)

**Controller** - third-party load-banalcing solution deployed. Not a usual load-balancer, but have additional intelligence built into them to monitor the kubernetes cluster for new definitions or ingress resources and configure the nginx server accordingly.

**Resources** - set of rules for colntroller, created using definition files.

##### Basic architecture

![image-20200112161057795](../.img/image-20200112161057795.png)

##### Setup process

1. Create a **namespace** for all ingress components

`kubectl create namespace ingress-space`

2. Create empty **ConfigMap** (for all specific NGINX Controller configuration)

`kubectl create configmap --namespace=ingress-space nginx-configuration`

*now it's empty, but can be used to modify ingress configuration in future*

3. Create **ServiceAccount** (right cluster-managing permissions for NGINX Conltroller)

`kubectl create serviceaccount --namespace=ingress-space ingress-serviceaccount`

4. **Role & RoleBinding** + **ClusterRole & ClusterRoleBinding** for `ingress-serviceaccount`

**Role**

```bash
kubectl get role --namespace=ingress-space ingress-role -oyaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  creationTimestamp: "2020-03-28T15:20:39Z"
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  name: ingress-role
  namespace: ingress-space
  resourceVersion: "1806"
  selfLink: /apis/rbac.authorization.k8s.io/v1/namespaces/ingress-space/roles/ingress-role
  uid: 40262e5a-aa37-412f-bdda-981acf129d9e
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - pods
  - secrets
  - namespaces
  verbs:
  - get
- apiGroups:
  - ""
  resourceNames:
  - ingress-controller-leader-nginx
  resources:
  - configmaps
  verbs:
  - get
  - update
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - get
```

**RoleBinding**

```bash
kubectl get rolebinding --namespace=ingress-space ingress-role-binding -oyaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2020-03-28T15:20:39Z"
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  name: ingress-role-binding
  namespace: ingress-space
  resourceVersion: "1807"
  selfLink: /apis/rbac.authorization.k8s.io/v1/namespaces/ingress-space/rolebindings/ingress-role-binding
  uid: 947b8644-6f1a-47cc-8fba-6bdde2c8d3f0
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-role
subjects:
- kind: ServiceAccount
  name: ingress-serviceaccount
```

**ClusterRole**

```bash
kubectl get clusterrole --namespace=ingress-space ingress-clusterrole -oyaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: "2020-03-28T15:20:39Z"
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  name: ingress-clusterrole
  resourceVersion: "1805"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/ingress-clusterrole
  uid: 66bfce52-a26c-4141-929e-4eee6b81c8ce
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - nodes
  - pods
  - secrets
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - extensions
  resources:
  - ingresses/status
  verbs:
  - update
```

**ClusterRoleBinding**

```bash
kubectl get clusterrolebindings.rbac.authorization.k8s.io --namespace=ingress-space ingress-clusterrole-binding -oyaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: "2020-03-28T15:20:39Z"
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  name: ingress-clusterrole-binding
  resourceVersion: "1808"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/ingress-clusterrole-binding
  uid: ad469988-cee5-4e14-b527-422cbc8ea307
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-clusterrole
subjects:
- kind: ServiceAccount
  name: ingress-serviceaccount
  namespace: ingress-space
```

5. Deploy NGINX **Ingress Controller**

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-controller
  namespace: ingress-space
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      serviceAccountName: ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --default-backend-service=app-space/default-http-backend
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
```

6. Create **Ingress Service** to make it available from outside

`kubectl expose deployment -n ingress-space ingress-controller --type=NodePort --port=80 --name=ingress --dry-run -o yaml > ingress.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress
  namespace: ingress-space
spec:
  type: NodePort
  selector:
    name: nginx-ingress
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
      name: http
    - protocol: TCP
      port: 443
      targetPort: 443
      name: https
```

7. Create Ingress Resource to make applications inside deployments available on different URLs

**Splitting trafic by paths**

```yaml
apiVersion: extentions/v1beta1
kind: Ingress
metadata:
  name: ingress-wear-watch
  namespace: app-space
  annotations:
    # an opetion of nginx-ingres-controller
    # This rewrites the URL by replacing whatever is under 'rules->http->paths->path' with the value in rewrite-target. (beacause wear and video might be an independent applications witch doesn't have it's own /wear and /watch paths)
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /wear
        backend:
          serviceName: wear-service
          servicePort: 8080
      - path: /watch
        backend:
          serviceName: video-service
          servicePort: 8080
```

*If a user tries to access a URL that does not match any of these rules, then the user is directed to the service specified as the **default-http-backend**.*

**Splitting trafic by hostnames**

```yaml
  ...
  rules:
  - host: wear.my-online-store.com
    http:
      paths:
      - backend:
          serviceName: wear-service
          servicePort: 8080
  - host: video.my-online-store.com
    http:
      paths:
      - backend:
          serviceName: video-service
          servicePort: 8080
```

8. Done!

```bash
kubectl get all --namespace=ingress-space
kubectl edit ingress --namespace=app-space ingress-wear-watch
cat <<EOF | kubectl create -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-pay
  namespace: critical-space
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /pay
        backend:
          serviceName: pay-service
          servicePort: 8282
EOF
```

---

### Kubernetes the Hard Way

#### Cluster designing

1.  Purpose

**education** - minikube or single node kubeadm/gcp/aws
**development** - single master and multiple workers, kubeadm/gcp/aws/aks
**production** - High Availability cluster with miltiple masters and workers, kubeadm/gcp/aws/aks

![image-20200405131756729](../.img/image-20200405131756729.png)

2.  Provider

**cloud-provider** - gcp/aws/aks etc.
**self-hosted cluster** - kubeadm

3.  Workloads

**storage:**
High Performance – SSD Backed Storage
Multiple Concurrent connections – Network based storage
Persistent shared volumes for shared access across multiple PODs
Label nodes with specific disk types
Use Node Selectors to assign applications to nodes with specific disk types

**resources requirements:**
![image-20200405131756729](../.img/image-20200405131756729.png)

**etcd:**
High Availability suggests to host etcd instances separated from master nodes on it's own nodes

#### Choosing infrastructure

Fore this example I'll choose self-hosted cluster with virtual machines as a nodes.

#### Network solution

https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model
WeaveNet CNI

#### High Availability

*multiple master nodes*

Master hosts a control-plane conponents: apiserver, controller-manager, etcd, scheduler.

##### kube-apiserver

can be alive a few instances at the same time (requires load-balances in front of the cluster)
![image-20200405142601725](../.img/image-20200405142601725.png)

##### controller-manager & scheduler

can't work at parallel with instances on other masters due to risk of duplicate actions
So only one instance might be Active at a time.
Set option `--leader-elect=false` for all instances except one (set as true by default). It makes this instance Active.
Set option `--leader-elect-lease-duration=15s` to specify a time amount to hold in a passive state.
Option `--leader-elect-renew-deadline=10s` active process then renews the lease every 10 sec.
Both the processes tries to become the leader every two seconds, set by the `--leader-elect-retry-period` option.

##### etcd

*can be **part of a master** or **hosted in separate node***

#### ETCD in a High Availability

Distributed etcd storage provides an ability to read and write data from any of it's instances.
Etcd ensures that every instance contains the same data.

##### Leader election with RAFT Protocol

**Example:**

*   3 nodes
*   random timer on each node
*   node with shortest timer became a leader
*   non-leader will respond with accepting a new leader
*   done
*   from now leader must send a regular notification to followers to prove that is alive
*   if followers do not recieve a notification from leader, they run random timer again

##### ETCD Write-requests

**Only one leader instance responsible for processing write-requests at a time!**
*write-request always goes from follower etcd-instances to current leader instance, however **read**-requests always process by instance itself*

**The write is considered to be complete only of it can be written on a Majority number of nodes.**

**Majority** (Quorum) = NODES / 2 + 1 (**rounded to a smaller**)
*a minimum number of nodes must be available to complete write request*

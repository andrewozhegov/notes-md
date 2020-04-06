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


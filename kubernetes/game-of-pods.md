# Game of Pods

### 1. Bravo. Drupal

##### Install Drupal using Docker

```sh
# run PostgreSQL container at first
docker run --name drupal_db -d \
    -v drupal_data:/var/lib/postgresql/data \
    -e POSTGRES_USER=postgres \
    -e POSTGRES_PASSWORD=postgres \
    -e POSTGRES_DATABASE=postgres \
    postgres:latest

# run Drupal container & link DB container
docker run --name drupal -d \                            # run in background
    -v drupal_modules:/var/www/html/modules \            # volumes
    -v drupal_profiles:/var/www/html/profiles \
    -v drupal_themes:/var/www/html/themes \
    -v drupal_sites:/var/www/html/sites \
    -p 80:80 \                                           # expose 80 port (web)
    --link drupal_db:drupal-postgres \                   # link with drupal_db
    drupal:latest

# visit http://localhost:80 to end up install process
# specify linked db name 'drupal-postgres' as DB host for Drupal
```

##### Install Drupal using Kubernetes

1.  Create a `hostPath` directories for PersistentVolumes on a worker node

```sh
ssh node01
mkdir /drupal-data /drupal-mysql-data
```

2.  Create **PersistentVolumes** for `drupal-data` and `drupal-mysql-data`

```yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: drupal-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/drupal-data"
EOF
```

```yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: drupal-mysql-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/drupal-mysql-data"
EOF
```

3.  **PersistentVolumeClaims** for `drupal-pv` and `drupal-mysql-pv`

```yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: drupal-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF
```

```yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: drupal-mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF
```

4.  Deploy **Drupal** itself

```yaml
cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drupal
  labels:
    app: drupal
spec:
  replicas: 1
  selector:
    matchLabels:
      app: drupal
  template:
    metadata:
      labels:
        app: drupal
    spec:
      initContainers:
      - name: init-sites-volume
        image: drupal:8.6
        command: ['/bin/bash', '-c']
        args: ['cp -r /var/www/html/sites/ /data/; chown www-data:www-data /data/ -R']
        volumeMounts:
          - mountPath: "/data"
            name: drupal-pv
      volumes:
        - name: drupal-pv
          persistentVolumeClaim:
            claimName: drupal-pvc
      containers:
      - name: drupal
        image: drupal:8.6
        ports:
        - containerPort: 80
        volumeMounts:
          - mountPath: "/var/www/html/modules"
            subPath: modules
            name: drupal-pv
          - mountPath: "/var/www/html/profiles"
            subPath: profiles
            name: drupal-pv
          - mountPath: "/var/www/html/sites"
            subPath: sites
            name: drupal-pv
          - mountPath: "/var/www/html/themes"
            subPath: themes
            name: drupal-pv
EOF
```

5.  Create **Secret** with MySQL credentials

```sh
kubectl create secret generic drupal-mysql-secret \
    --from-literal=MYSQL_ROOT_PASSWORD=root_password \
    --from-literal=MYSQL_DATABASE=drupal-database \
    --from-literal=MYSQL_USER=root
```

6.  Deploy **MySQL**

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drupal-mysql
  labels:
    app: drupal-mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: drupal-mysql
  template:
    metadata:
      labels:
        app: drupal-mysql
    spec:
      volumes:
        - name: drupal-mysql-pv
          persistentVolumeClaim:
            claimName: drupal-mysql-pvc
      containers:
      - name: drupal
        image: mysql:5.7
        ports:
        - containerPort: 80
        volumeMounts:
          - mountPath: "/var/lib/mysql"
            subPath: dbdata
            name: drupal-mysql-pv
        env:
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name:  drupal-mysql-secret
              key: MYSQL_DATABASE
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: drupal-mysql-secret
              key: MYSQL_ROOT_PASSWORD
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: drupal-mysql-secret
              key: MYSQL_USER
EOF
```

7.  Expose MySQL as **ClusterIP**

```sh
kubectl expose deployment drupal-mysql \
    --name=drupal-mysql-service \
    --type=ClusterIP \
    --port=3306 \
    --selector='app=drupal-mysql'
```

8.  Expose Drupal as **NodePort**

```sh
kubectl expose deployment drupal \
    --name=drupal-service \
    --type=NodePort \
    --port=80 \
    --selector='app=drupal' \
    --dry-run -oyaml > frontend-svc.yaml
vim frontend-svc.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: drupal
  name: drupal-service
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30095 # specify nodePort
  selector:
    app: drupal
  type: NodePort
status:
  loadBalancer: {}
```

### 2. Pento. Troubleshooting and FileServer deploying

##### Docker approach

```sh
# run fileserver container
docker run -d
    --name fileserver \
    -v /web:/web \
    -p 8080:8080 \
    kodekloud/fileserver

# visit http://localhost:8080 and see contents of '/web' directory
```

##### Kubernetes approach

1.  **master** node troubleshooting

```sh
$ kubectl get pods
Unable to connect to the server: x509: certificate signed by unknown authority

$ ps aux | grep kube-apiserver
... kube-apiserver --advertise-address=172.17.0.45 --secure-port=6443 ...
# check ~/.kube/config and fix ip:port of the cluster
# 6443 is a standart port for kube-apiserver
```

```sh
# check etcd and apiserver logs
$ docker logs k8s_etcd_etcd-master_kube-system_70002ede83541cb00eb7e83dd778d943_0
2020-03-31 17:05:34.440785 I | embed: rejected connection from "172.17.0.17:56792" (error "remote error: tls: bad certificate", ServerName "")

$ docker logs k8s_kube-apiserver_kube-apiserver-master_kube-system_8b923cfadd326ccd8bd3b3404158635e_10
Error: unable to load client CA file: unable to load client CA file: open /etc/kubernetes/pki/ca-authority.crt: no such file or directory

# check kube-apiserver manifest
$ vim /etc/kubernetes/manifests/
# change '--client-ca-file=' to /etc/kubernetes/pki/ca.crt
```

```sh
# check for all pods are Running
$ kubectl get pods --all-namespaces
NAMESPACE    NAME                          READY  STATUS            RESTARTS  AGE
kube-system  pod/coredns-855c87754f-6jbrv  0/1    ImagePullBackOff  0         21m
kube-system  pod/coredns-855c87754f-m9ksd  0/1    ImagePullBackOff  0         21m
...

$ kubectl describe --namespace=kube-system pods coredns-855c87754f-6jbrv
... manifest for k8s.gcr.io/kubedns:1.3.1 not found

# change image to 'k8s.gcr.io/coredns:1.3.1'
$ kubectl edit deployments.apps coredns --namespace=kube-system
```

2.  worker **node01** troubleshooting

```sh
# beacouse katacoda-cloud-provider doesn't tolerate with master and might be deployed only on worker node
$ kubectl get deployments.apps --all-namespaces
NAMESPACE   NAME                                     READY  UP-TO-DATE  AVAILABLE  AGE
kube-system deployment.apps/katacoda-cloud-provider  0/1    1           0          64m

# worker node is cordoned
$ kubectl get node
NAME     STATUS                     ROLES    AGE   VERSION
master   Ready                      master   69m   v1.14.0
node01   Ready,SchedulingDisabled   <none>   69m   v1.14.0

# let's enable Sheduling on node01
$ kubectl uncordon node01
```

3.  Create `/web` directory on worker node

```sh
ssh node01
mkdir /web
```

4.  Create **PersistentVolume** for `/web`

```yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/web"
EOF
```

5.  Create **PersistentVolumeClaim** for `data-pv`

```yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF
```

6.  Create fileserver **Pod**

```yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: gop-fileserver
  labels:
    app: fileserver
spec:
  volumes:
  - name: data-store
    persistentVolumeClaim:
      claimName: data-pvc
  containers:
  - image: kodekloud/fileserver
    name: fileserver
    volumeMounts:
    - mountPath: "/web"
      subPath: web
      name: data-store
EOF
```

7.  Expose pod with **NodePort**

```sh
kubectl expose pod gop-fileserver \
    --name=gop-fs-service \
    --type=NodePort \
    --port=8080 \
    --selector='app=fileserver' \
    --dry-run -oyaml > fileserver-svc.yaml
vim fileserver-svc.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: fileserver
  name: gop-fs-service
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
    nodePort: 31200 # specify nodeport
  selector:
    app: fileserver
  type: NodePort
status:
  loadBalancer: {}
```

### 3. Redis Cluster

##### Docker

```sh
mkdir redis-cluster
vim redis-cluster/cluster-config.conf
```

```
port 6379
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

```sh
cd redis-cluster

# run 6 equal redis containers
for j in `seq 1 6`; do docker run -d -v \
    $PWD/cluster-config.conf:/usr/local/etc/redis/redis.conf \
    --name redis-$j \
    redis redis-server /usr/local/etc/redis/redis.conf; \
done

# get IP of all redis containers
ip_list=`
for j in `seq 2 6`; do
    docker inspect \
        -f '{{ (index .NetworkSettings.Networks "bridge").IPAddress }}' \
        redis-$j; \
done`

# create cluster combining all instances ips
docker exec -it redis-1 redis-cli \
    --cluster create \
    --cluster-replicas 1 \
    $ip_list

# check cluster status from any instance
docker exec -it redis-1 redis-cli cluster info
```

##### Kubernetes

1.  Explore already existing **ConfigMap**

```yaml
$ kubectl get configmaps redis-cluster-configmap -oyaml
apiVersion: v1
data:
  redis.conf: |-
    cluster-enabled yes
    cluster-require-full-coverage no
    cluster-node-timeout 15000
    cluster-config-file /data/nodes.conf
    cluster-migration-barrier 1
    appendonly yes
    protected-mode no
  update-node.sh: |
    #!/bin/sh
    REDIS_NODES="/data/nodes.conf"
    sed -i -e "/myself/ s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/${POD_IP}/" ${REDIS_NODES}
    exec "$@"
kind: ConfigMap
metadata:
  creationTimestamp: "2020-04-01T18:01:53Z"
  name: redis-cluster-configmap
  namespace: default
  resourceVersion: "7623"
  selfLink: /api/v1/namespaces/default/configmaps/redis-cluster-configmap
  uid: de9f9317-7442-11ea-98bc-0242ac11001f
```

2.  Create `hostPath` directories  on the worker node for each redis instance

```sh
ssh node01
for n in `seq 0 6`; do mkdir /redis0$n; done
```

3.  Create **PersistentVolume** for each of the `hostPath` directories

```yaml
for n in `seq 1 6`; do
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis0$n
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/redis0$n"
EOF
done
```

3.  Deploy 6 redis instances as a **StatefulSet**

```yaml
cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  selector:
    matchLabels:
      app: redis-cluster
  serviceName: "redis-cluster-service"
  replicas: 6
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: redis
        image: redis:5.0.1-alpine
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster-configmap
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
EOF
```

4.  Create cluster by combining all instances IPs

```bash
kubectl exec -it redis-cluster-0 -- redis-cli \
    --cluster create \
    --cluster-replicas $(kubectl get pods \
        -l app=redis-cluster \
        -o jsonpath='{range.items[*]}{.status.podIP}:6379')
```

5.  Expose StatefulSet as a **ClusterIP**

```yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster-service
spec:
  ports:
  - port: 6379
    name: client
  - port: 16379
    name: gossip
  clusterIP: None
  selector:
    app: redis-cluster
EOF
```

### 4. Tyro. Jekyll SSG

*static site generator*

##### Docker

```sh
# reate directory for site
mkdir /site

# run site creation and put it in created dir
docker run -d \
    --name jekyll \
    -v /site:/site \
    kodekloud/jekyll new /site

# run created site
docker run -d \
    -p 8080:4000 \
    -v /site:/site \
    kodekloud/jekyll-serve

# now you can access the application on http://localhost:8080
```

##### Kubernetes

1.  `hostPath` dir ona worker node already created
2.  **PersistentVolume** for `hostPath` alredy created
3.  Create **PersistentVolumeClaim**

```yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jekyll-site
  namespace: development
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF
```

4.  Add `drogo` user to kubeconfig

```yaml
...
# add context
contexts:
- context:
    cluster: kubernetes
    user: drogo
  name: developer
...
# add user & specify crts
users:
- name: drogo
  user:
    client-certificate: /root/drogo.crt
    client-key: /root/drogo.key
...
```

5.  Add **Role** for `drogo`

```yaml
cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer-role
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["services", "pods", "persistentvolumeclaims"]
  verbs: ["*"]
EOF
```

5.  Create **RoleBinding** for user `drogo` and `developer-role`

```yaml
cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-rolebinding
  namespace: development
subjects:
- kind: User
  name: drogo
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

6.  Use `developer` context as a default

```sh
kubectl config use-context developer
```

7.  Deply `jekyll` as a **Pod**

```yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: jekyll
  namespace: development
  labels:
    run: jekyll
spec:
  initContainers:
  - name: copy-jekyll-site
    image: kodekloud/jekyll
    command: [ "jekyll", "new", "/site" ]
    volumeMounts:
      - mountPath: "/site"
        name: site
  volumes:
    - name: site
      persistentVolumeClaim:
        claimName: jekyll-site
  containers:
  - name: jekyll
    image: kodekloud/jekyll-serve
    volumeMounts:
      - mountPath: "/site"
        name: site
EOF
```

7. Expose pod `jekyll` as a **NodePort**

```sh
kubectl expose pod jekyll \
    --namespace=development \
    --name jekyll-node-service \
    --port=8080 \
    --target-port=4000 \
    --dry-run -oyaml > jekyll-svc.yaml
vim jekyll-svc.yaml
```

```yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Service
metadata:
  name: jekyll
  namespace: development
spec:
  type: NodePort
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 4000
    nodePort: 30097 # add nodeport
  selector:
    run: jekyll
    namespace: development
EOF
```


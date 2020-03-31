# Game of Pods

### 1. Drupal

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
cat <<EOF | kubectl create -f - --dry-run
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
EOFcat <<EOF | kubectl create -f -
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


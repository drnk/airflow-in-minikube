# PREREQUISITIES

Configuring helm:
* `helm repo add apache-airflow https://airflow.apache.org`
* `helm repo add bitnami https://charts.bitnami.com/bitnami`

Or update helm repo:
`helm repo update`


Create folder for postgres data:
`mkdir -p /home/drnk/dev/k8s/mnt/postgresql/data`

Verify that `postgresql` and `data` directories belongs to your user. Postgres will run with 1000 user id and it should equals to yours.  

Create folder for airflow data:
`mkdir -p /home/drnk/dev/k8s/mnt/airflow`
`chmod 777 /home/drnk/dev/k8s/mnt/airflow`


Create folder for postgres data:
`mkdir -p /home/drnk/dev/k8s/mnt/postgresql/data` 


# STARTING MINIKUBE


Start `minikube` instance:
`minikube start --driver=docker --cpus 6 --memory 6144 --mount --mount-string="/home/drnk/dev/k8s/mnt:/mnt"`

### Another starting minikube exxamples
Start with additional logging:
`minikube start --alsologtostderr --v=2`

Start with different driver (default is `docker`)
`minikube start --driver=kvm2 --cpus 6 --memory 6144`


# STORAGE

Create namespaces for database:
* `kc create namespace databases`

Create PV and PVC:

```
kc apply -f - << EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
  namespace: databases
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 4Gi
  hostPath:
    path: "/mnt/postgresql"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-postgresql-storage
  namespace: databases
spec:
  storageClassName: standard
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
EOF
```

Create namespace for airflow:
`kc create namespace airflow`

And then create PV and PVCs for logs and DAGs:

```
kc apply -f - << EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: airflow-data
  namespace: airflow
spec:
  accessModes:
    - ReadOnlyMany
  capacity:
    storage: 1Gi
  hostPath:
    path: "/mnt/airflow/dags"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: airflow-data-storage
  namespace: airflow
spec:
  storageClassName: ""
  accessModes:
  - ReadOnlyMany
  volumeName: airflow-data
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: airflow-logs
  namespace: airflow
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 4Gi
  hostPath:
    path: "/mnt/airflow/logs"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: airflow-logs-storage
  namespace: airflow
spec:
  storageClassName: ""
  accessModes:
  - ReadWriteOnce
  volumeName: airflow-logs
  resources:
    requests:
      storage: 4Gi
EOF
```

# DATABASE

## Deploy Postgres instance

Deploy Postgres instance:
`helm upgrade --install postgresql-db bitnami/postgresql -f postgresql-values.yaml -n databases --create-namespace`

## Access Postgres instance

Wait until postgres start. You could check availalability of postgres pod via:
`kubectl get pods -n databases`

Exmple response with running Postgres instance:
```
NAME              READY   STATUS    RESTARTS   AGE
postgresql-db-0   1/1     Running   0          11m
```

Run below command to set env variables to connect then to postgres:
```
export POSTGRES_PASSWORD=$(kubectl get secret --namespace databases postgresql-db -o jsonpath="{.data.postgres-password}" | base64 -d) && \
export NODE_IP=$(kubectl get nodes --namespace databases -o jsonpath="{.items[0].status.addresses[0].address}") && \
export NODE_PORT=$(kubectl get --namespace databases -o jsonpath="{.spec.ports[0].nodePort}" services postgresql-db)
```

Connect to database using `psql`. If you don't have then you need to install it before:
`PGPASSWORD="$POSTGRES_PASSWORD" psql --host $NODE_IP --port $NODE_PORT -U postgres -d postgres`


## Setup Airflow database
https://airflow.apache.org/docs/apache-airflow/stable/howto/set-up-database.html#setting-up-a-postgresql-database

```
CREATE DATABASE airflow_db;
CREATE USER airflow_user WITH PASSWORD 'airflow_pass';
GRANT ALL PRIVILEGES ON DATABASE airflow_db TO airflow_user;
```

# AIRFLOW

Deploy Airflow instance:
`helm upgrade --install airflow apache-airflow/airflow -f airflow-values.yaml -n airflow --create-namespace`
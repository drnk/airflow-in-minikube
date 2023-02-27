# PREREQUISITIES

Configuring helm:
* `helm repo add apache-airflow https://airflow.apache.org`
* `helm repo add bitnami https://charts.bitnami.com/bitnami`


Create folder for postgres data:
`mkdir -p /home/drnk/dev/k8s/mnt/postgresql/data`

Verify that `postgresql` and `data` directories belongs to your user. Postgres will run with 1000 user id and it should equals to yours.  

Create folder for airflow data:
`mkdir -p /home/drnk/dev/k8s/mnt/airflow`
`chmod 777 /home/drnk/dev/k8s/mnt/airflow`



# STARTING MINIKUBE


Start `minikube` instance:
`minikube start --alsologtostderr --v=2`
`minikube start --cpus 6 --memory 6144 --alsologtostderr --v=2`

`minikube start --driver=kvm2 --cpus 6 --memory 6144`

`minikube start --driver=docker --cpus 6 --memory 6144 --mount --mount-string="/home/drnk/dev/k8s/mnt:/mnt"`



--extra-config=kubelet.cgroup-driver=systemd

Mount local folder to minikube VM (don't close)
`minikube mount /home/drnk/dev/k8s/mnt:/opt/mnt`
`minikube mount /home/drnk/dev/k8s/mnt:/opt/mnt --uid 10000 --gid 1000`


# STORAGE

Create namespace first:
`kc create namespace databases`


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


# DATABASE

Deploy Postgres instance:
`helm upgrade --install postgresql-db bitnami/postgresql -f postgresql-values.yaml -n databases --create-namespace`


postgresql-db.databases.svc.cluster.local

`export POSTGRES_PASSWORD=$(kubectl get secret --namespace databases postgresql-db -o jsonpath="{.data.postgres-password}" | base64 -d)`

```
** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    postgresql-db.databases.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace databases postgresql-db -o jsonpath="{.data.postgres-password}" | base64 -d)

To connect to your database run the following command:

    kubectl run postgresql-db-client --rm --tty -i --restart='Never' --namespace databases --image docker.io/bitnami/postgresql:11.19.0-debian-11-r2 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host postgresql-db -U postgres -d postgres -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    export NODE_IP=$(kubectl get nodes --namespace databases -o jsonpath="{.items[0].status.addresses[0].address}")
    export NODE_PORT=$(kubectl get --namespace databases -o jsonpath="{.spec.ports[0].nodePort}" services postgresql-db)
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host $NODE_IP --port $NODE_PORT -U postgres -d postgres

WARNING: The configured password will be ignored on new installation in case when previous Posgresql release was deleted through the helm command. In that case, old PVC will have an old password, and setting it through helm won't take effect. Deleting persistent volumes (PVs) will solve the issue.
```

## Setup Database
https://airflow.apache.org/docs/apache-airflow/stable/howto/set-up-database.html#setting-up-a-postgresql-database

```
CREATE DATABASE airflow_db;
CREATE USER airflow_user WITH PASSWORD 'airflow_pass';
GRANT ALL PRIVILEGES ON DATABASE airflow_db TO airflow_user;
```

`PGPASSWORD="airflow_pass" psql --host $NODE_IP --port $NODE_PORT -U airflow_user -d airflow_db`

# AIRFLOW

`kc create namespace airflow`

Create PV and PVCs for logs and DAGs:

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

Deploy Airflow instance:
`helm upgrade --install airflow apache-airflow/airflow -f airflow-values.yaml -n airflow --create-namespace`
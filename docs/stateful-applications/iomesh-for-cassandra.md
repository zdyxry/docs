---
id: iomesh-for-cassandra
title: IOMesh for Cassandra
sidebar_label: IOMesh for Cassandra
---

## Setup k8s Cluster Storage

1. Create a file named `iomesh-cassandra-sc.yaml` with the following content:

```text
iomesh-cassandra-sc.yaml
```

```output
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: iomesh-cassandra-sc
provisioner: com.iomesh.csi-driver # driver.name in values.yaml when install IOMesh cluster
reclaimPolicy: Retain
allowVolumeExpansion: true
parameters:
  csi.storage.k8s.io/fstype: "ext4"
  replicaFactor: "2"
  thinProvision: "true"
```

2. Apply the yaml config:

```bash
kubectl apply -f iomesh-cassandra-sc.yaml
```

## Deploy Cassandra

### Create a headless Service for Cassandra

1. Create a Service used for DNS lookups between Cassandra Pods and clients within your cluster

```text
cassandra-service.yaml
```

```output
apiVersion: v1
kind: Service
metadata:
  labels:
    app: cassandra
  name: cassandra
spec:
  clusterIP: None
  ports:
  - port: 9042
  selector:
    app: cassandra
```

2. Apply the yaml config:

```bash
kubectl apply -f cassandra-service.yaml
```

### Create Cassandra cluster use pv provided for IOMesh Storage

1. Use StatefulSet to create a cassandra cluster

```text
cassandra-statefulset.yaml
```

```output
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cassandra
  labels:
    app: cassandra
spec:
  serviceName: cassandra
  replicas: 3
  selector:
    matchLabels:
      app: cassandra
  template:
    metadata:
      labels:
        app: cassandra
    spec:
      terminationGracePeriodSeconds: 1800
      containers:
      - name: cassandra
        image: gcr.io/google-samples/cassandra:v13
        imagePullPolicy: Always
        ports:
        - containerPort: 7000
          name: intra-node
        - containerPort: 7001
          name: tls-intra-node
        - containerPort: 7199
          name: jmx
        - containerPort: 9042
          name: cql
        resources:
          limits:
            cpu: "500m"
            memory: 1Gi
          requests:
            cpu: "500m"
            memory: 1Gi
        securityContext:
          capabilities:
            add:
              - IPC_LOCK
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - nodetool drain
        env:
          - name: MAX_HEAP_SIZE
            value: 512M
          - name: HEAP_NEWSIZE
            value: 100M
          - name: CASSANDRA_SEEDS
            value: "cassandra-0.cassandra.default.svc.cluster.local"
          - name: CASSANDRA_CLUSTER_NAME
            value: "K8Demo"
          - name: CASSANDRA_DC
            value: "DC1-K8Demo"
          - name: CASSANDRA_RACK
            value: "Rack1-K8Demo"
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
        readinessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - /ready-probe.sh
          initialDelaySeconds: 15
          timeoutSeconds: 5
        volumeMounts:
        - name: cassandra-data
          mountPath: /cassandra_data
  volumeClaimTemplates:
  - metadata:
      name: cassandra-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: iomesh-cassandra-sc # storageClass created above
      resources:
        requests:
          storage: 10Gi
```

2. Apply the yaml config:

```bash
kubectl apply -f cassandra-statefulset.yaml
```

IOMesh Storage will create Persistent Volumes for each cassandra pod. These volumes use the ext4 file system with a replica factor of 2 and are thin provisioned.

## Operate Cassandra Data

User can use the feature provided by IOMesh storage to perform operations such as expansion/snapshot/rollback/clone of the Persistent Volumes  where Cassandra data is located, see reference for details [application-operations](https://docs.iomesh.com/volume-operations/snapshot-restore-and-clone)

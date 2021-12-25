# DigitalOcean Kubernetes Challenge - Deploying a scalable NoSQL database cluster
## Introduction

Kubernetes is a powerful open-source system for managing containerized applications in a clustered environment. Its focus is to improve how related, distributed components and services are managed across varied infrastructure.

DigitalOcean Kubernetes is a managed Kubernetes service lets users deploy scalable and secure Kubernetes clusters without the complexities of administrating the control plane. In this challenge, we would be deploying a scalable MongoDB cluster on Kubernetes.

## Prerequisites
- [DigitalOcean Account](https://cloud.digitalocean.com/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) commandline tool installed
- doctl setup for smooth integration ( [How to install and setup doctl ?](https://docs.digitalocean.com/reference/doctl/how-to/install/) )

## Creating a kubernetes cluster
After the verification of the DigitalOcean account, k8s cluster can be created from the dashboard.

In the dashboard go to kubernetes section.
![DigitalOcean Kubernetes](https://ik.imagekit.io/hrgu22madpa/K8_challenge/Screenshot_2021-12-25_at_11.58.07_AM_p97yugw1d.png?updatedAt=1640413567830)
Select create new cluster
![Create cluster DigitalOcean Kubernetes](https://ik.imagekit.io/hrgu22madpa/K8_challenge/Screenshot_2021-12-25_at_12.01.28_PM_DABD11XZEj5H.png?updatedAt=1640413710237)
Fill out the form and your cluster will be setup within few minuites.
![choose cluster DigitalOcean Kubernetes](https://ik.imagekit.io/hrgu22madpa/K8_challenge/Screenshot_2021-12-25_at_12.02.42_PM_2rRymixMJ43d.png?updatedAt=1640413838536)

Creating cluster DigitalOcean Kubernetes
This is it.


## Connecting to the cluster
It takes around 5 minutes for the cluster to set up. In the meantime, the cluster config file can be downloaded using,
```
cd ~/.kube && kubectl --kubeconfig="k8s-mongo-kubeconfig.yaml" get nodes
```


## Creating MongoDB secrets
Secrets in Kubernetes are the objects used for supplying sensitive information to containers. For the security of our MongoDB instance, it is wise to restrict access to the database with a password. We will use secrets to mount our desired passwords to the containers. The secrets are encoded in base64 version in the `mongodb-secrets.yaml` file.
```
apiVersion: v1
data:
  password: cGFzc3dvcmQxMjM= #password123
  username: YWRtaW51c2Vy #adminuser
kind: Secret
metadata:
  creationTimestamp: null
  name: mongo-creds
```

To apply the changes to our k8s cluster,
```
kubectl apply -f mongodb-secrets.yaml
```

## Creating MongoDB Persistent Volume
We require volumes to store the persistent data. In this way, even if our pod goes down, the data is not lost. There are 2 objects for creating volumes in Kubernetes.
1. Persistent Volume Claims (PVC)
2. Persistent Volumes (PV)

Create a PVC `mongodb-pvc.yaml` file,
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteOnce 
  resources:
    requests:
      storage: 1Gi
```

Create the PVC using the command,
```
kubectl create -f mongodb-pvc.yaml
```

Now, we bind the PVC with a PV specified in `mongodb-pv.yaml` file,
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongo-pv
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /data/mongo
```

Create the PV using the command,
```
kubectl create -f mongodb-pv.yaml
```

## Deploying the MongoDB deployment
Using the official mongo image from docker hub, we create a `mongodb-deployment.yaml` file to deploy the MongoDB cluster on Kubernetes.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: mongo
  name: mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  strategy: {}
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - image: mongo
        name: mongo
        args: ["--dbpath","/data/db"]
        livenessProbe:
          exec:
            command:
              - mongo
              - --disableImplicitSessions
              - --eval
              - "db.adminCommand('ping')"
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 6
        readinessProbe:
          exec:
            command:
              - mongo
              - --disableImplicitSessions
              - --eval
              - "db.adminCommand('ping')"
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 6
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongo-creds
              key: username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-creds
              key: password
        volumeMounts:
        - name: "mongo-pvc-dir"
          mountPath: "/data/db"
      volumes:
      - name: "mongo-pvc-dir"
        persistentVolumeClaim:
          claimName: "mongo-pvc"
```

Create the mongo deployment using the command,
```
kubectl create -f mongodb-deployment.yaml
```

Then to see everything up and running. enter the following command in terminal.[Optional]
```sh
kubectl get all
```
![kubernets get all](https://ik.imagekit.io/hrgu22madpa/K8_challenge/Screenshot_2021-12-25_at_12.09.17_PM_ULpVBoWfIzsC.png?updatedAt=1640414196837)
We can see that our mongoDB cluster is now setup and working correctly. Now we enter the bash shell of our mongo-client

```sh
kubectl exec deployment/mongo-client -it -- /bin/bash
```
![Mongoclient kubernetes](https://ik.imagekit.io/hrgu22madpa/K8_challenge/Screenshot_2021-12-25_at_12.11.36_PM_EAMl7D7uMir.png?updatedAt=1640414328693)
Now we can access our mongo shell via noodeport

> Note: Make sure you have changed the username and password in the 'mongodb-secrets.yaml' file.
```sh
mongo
```

or

```sh
mongo --host mongo-nodeport-svc --port 27017 -u adminuser -p password123
```
![mongo shell kubernetes](https://ik.imagekit.io/hrgu22madpa/K8_challenge/Screenshot_2021-12-25_at_12.13.31_PM_ehcIUmjuv6E1.png?updatedAt=1640414603845)

We have successfully logged in to our mongo shell. We can test everything is working by inserting a document in a collection.

![mongo shell insert kubernetes](https://ik.imagekit.io/hrgu22madpa/K8_challenge/Screenshot_2021-12-25_at_12.19.28_PM_3aLUV_FaptZV.png?updatedAt=1640414819162)

## Conclusion
We have successfully deployed a MongoDB instance on k8s cluster using DigitalOcean Kubernetes.

## License

MIT
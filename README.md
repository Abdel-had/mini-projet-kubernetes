# Kubernetes Mini-Project
Hello everyone!

In this README, I will introduce you to my Kubernetes (k8s) mini-project and explain how I accomplished it step by step.
I completed this project as part of the DevOps eazytraining bootcamp and at the end of the second training module out of the six planned.

## Here is the task:

> <img width="663" alt="Consignes-mini-projet-k8s" src="https://user-images.githubusercontent.com/101605739/227731836-06550c5c-a97e-42ff-af18-46d9e57b9f99.png">

------------

Author: Abdel-had HANAMI

Context: Bootcamp DevOps training, 12th promotion

Training center: eazytraining.fr

Period: March-April-May

Date: March 25, 2023

LinkedIn : https://www.linkedin.com/in/abdel-had-hanami/


## Here is the infrastructure diagram that I propose

![suggested-architecture](https://user-images.githubusercontent.com/101605739/227812568-8ebd8e3e-e28f-4ab8-a0c6-09f379263711.jpg)


## Step 1: Namespace creation
To begin, I created a dev namespace in order to separate my resources from other environments.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```


## Step 2: Persistent Volumes and Persistent Volume Claims

Next, I created Persistent Volumes (PV) and Persistent Volume Claims (PVC) for my two applications: MySQL and WordPress. I chose to use persistent volumes to ensure data persistence between updates and container restarts.

The PV and PVC are configured for single read and write access (ReadWriteOnce). The PVCs request a storage capacity of 5 Gi for MySQL and 2 Gi for WordPress.

Caution: here the data is stored locally on the k8s cluster.

```yaml
# MySQL PV et PVC
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: standard
  hostPath:
    path: "/mnt/mysql-data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: dev
spec:
  resources:
    requests:
      storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
```

```yaml
# WordPress PV et PVC
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: standard
  hostPath:
    path: "/mnt/data-wordpress"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: dev
spec:
  resources:
    requests:
      storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
```

## Step 3: ConfigMap
I created a ConfigMap to store the shared configuration information between MySQL and WordPress. This allows me to centralize the configuration and make updates easier.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-mysql-config
  namespace: dev
data:
  WORDPRESS_DB_HOST: mysql-svc
  WORDPRESS_DB_USER: wp_user
  WORDPRESS_DB_PASSWORD: wp_password
  MYSQL_ROOT_PASSWORD: root_password
  WORDPRESS_DB_NAME: wordpress
```

## Step 4: Deployments
I created deployments for MySQL and WordPress with the Recreate deployment strategy, which ensures that the old pods are deleted before creating the new ones.

The deployments use the official images of MySQL and WordPress and retrieve the configuration information from the ConfigMap.

```yaml
# Déploiement MySQL
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: dev
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        env:
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_NAME
        - name: MYSQL_USER
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_USER
        - name: MYSQL_PASSWORD
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_PASSWORD
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: MYSQL_ROOT_PASSWORD
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-pvc
```
```yaml
# Déploiement WordPress
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: dev
spec:
  strategy:
    type: Recreate
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:6
        env:
        - name: WORDPRESS_DB_HOST
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_HOST
        - name: WORDPRESS_DB_USER
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_USER
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_PASSWORD
        - name: WORDPRESS_DB_NAME
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_NAME
        ports:
        - containerPort: 80
        volumeMounts:
        - name: wordpress-data
          mountPath: /var/www/html
      volumes:
      - name: wordpress-data
        persistentVolumeClaim:
          claimName: wordpress-pvc
```

## Step 5: Services
I created a ClusterIP service (backend) to expose the MySQL application within the Kubernetes cluster, while the service I created for WordPress is of type NodePort (frontend) to allow access from outside the cluster.

```yaml
# Service MySQL
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  namespace: dev
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
```

```yaml
# Service WordPress
apiVersion: v1
kind: Service
metadata:
  name: wordpress-svc
  namespace: dev
spec:
  selector:
    app: wordpress
  ports:
  - port: 8080
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

## Step 6: Ingress
At this stage, our application is already accessible from the outside via the virtual IP (VIP) of the cluster or that of one of the nodes, followed by port 30080. However, I wanted to make my application accessible via a simple URL. To do this, I created an Ingress rule to expose my WordPress application using a custom domain.

Indeed, when http://dev-wordpress.pozos.fr is requested, it is the ingress controller that will receive the request and redirect it to the wordpress-svc service inside the cluster on port 8080. This service will then be responsible for connecting to the pod in which the WordPress container is running.


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-wordpress-ingress
  namespace: dev
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: dev-wordpress.pozos.fr
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: wordpress-svc
            port:
              number: 8080
```

## Step 7: Deployment
To deploy these applications along with their infrastructure components, I use my all-in-one deployment manifest:

```
kubectl apply -f aio-wordpress-mysql-deployment.yml
```

If you wish to deploy the objects one by one, go to the *Step-by-step deployment* folder and run the kubectl apply -f <filename> command for each file in the order I suggest:

![image](https://user-images.githubusercontent.com/101605739/227747834-407fb89f-25e2-42ab-ba3e-18d83c33f79f.png)

You can also use `kubectl` to check the status of your deployed resources:

```
kubectl get all -n dev
```

## Step 8: Consuming the application

After deployment, you can access your WordPress application using the address http://dev-wordpress.pozos.fr (assuming you have correctly configured DNS resolution to point to your Kubernetes cluster).

![suggested-architecture](https://user-images.githubusercontent.com/101605739/227812568-8ebd8e3e-e28f-4ab8-a0c6-09f379263711.jpg)

--------

# Conclusion

In conclusion, this Kubernetes mini-project allowed me to set up a robust infrastructure for deploying a WordPress application coupled with a MySQL database. Thanks to the use of Kubernetes resources such as namespaces, persistent volumes, ConfigMaps, deployments, services, and Ingress, I was able to create a flexible, scalable, and easily maintainable environment for this application.

This experience also allowed me to apply the concepts and skills acquired during the DevOps eazytraining bootcamp, further strengthening my understanding of the various steps and resources involved in deploying an application on a Kubernetes cluster.

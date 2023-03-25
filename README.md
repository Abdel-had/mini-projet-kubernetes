# Mini-projet Kubernetes
Bonjour à tous ! 

Dans ce README, je vais vous présenter mon mini-projet Kubernetes (k8s) et vous expliquer comment je l'ai réalisé étape par étape.
J'ai réalisé ce projet dans le cadre du bootcamp DevOps eazytraining et à l'issue du deuxième module de formation sur les six prévus.

## Voici la consigne :

> <img width="663" alt="Consignes-mini-projet-k8s" src="https://user-images.githubusercontent.com/101605739/227731836-06550c5c-a97e-42ff-af18-46d9e57b9f99.png">

------------

Auteur : Abdel-had HANAMI

Contexte : formation Bootcamp DevOps promotion 12

Centre de formation : eazytraining.fr

Période : mars-apvil-mai

Date : 25 mars 2023

LinkedIn : https://www.linkedin.com/in/abdel-had-hanami/


## Voici le schéma d'infrastructure que je propose

[![](https://mermaid.ink/img/pako:eNp9k01zgjAQhv8Kk5POSL1z8II9OCMOlak9SA8pWZUp-Zgk2LGO_70BggSoXkh299l9s5twRRkngAJ0lFicvPU2ZZ6nyq_G3GAKSuAM9gTOnfVZQQ62BFHwCwWmVRPxvOiSvK0n9dcJT9vwB5cklqDU5L4bY8DIQCgBec4z6Ksk58wKhUWpNMhV3IJjvQruJDem95hLPeDHwjtelHSgG--sbAxS5UaYactNB1z4CDQnxjkdn9KU7g75uLzDh88SejLj5kLODvkxwqKtG0ZdNd-OtmX6VeyyYseK3du1fh9hkZsTTJqlzgrN7XD6vl3vT1qLYD4XEoTkxP8xYqJKfBH8l6uXg6wrNMq-79_vufJ2fVYR92L_yTGDuXvN3g08qWWzXHsI9LXCaFyt8TXtG8eia783izpk5-aMsnYPm3PtPjCUX_RG1u67QMrQDFGQFOfE_PzXikqRPgGFFAVmS7D8TlHKboYrBcEaXkmuuUTBARcKZgiXmicXlqFAyxJaaJlj86aopW5_CI1y1w)](https://mermaid-js.github.io/mermaid-live-editor/edit/#pako:eNp9k01zgjAQhv8Kk5POSL1z8II9OCMOlak9SA8pWZUp-Zgk2LGO_70BggSoXkh299l9s5twRRkngAJ0lFicvPU2ZZ6nyq_G3GAKSuAM9gTOnfVZQQ62BFHwCwWmVRPxvOiSvK0n9dcJT9vwB5cklqDU5L4bY8DIQCgBec4z6Ksk58wKhUWpNMhV3IJjvQruJDem95hLPeDHwjtelHSgG--sbAxS5UaYactNB1z4CDQnxjkdn9KU7g75uLzDh88SejLj5kLODvkxwqKtG0ZdNd-OtmX6VeyyYseK3du1fh9hkZsTTJqlzgrN7XD6vl3vT1qLYD4XEoTkxP8xYqJKfBH8l6uXg6wrNMq-79_vufJ2fVYR92L_yTGDuXvN3g08qWWzXHsI9LXCaFyt8TXtG8eia783izpk5-aMsnYPm3PtPjCUX_RG1u67QMrQDFGQFOfE_PzXikqRPgGFFAVmS7D8TlHKboYrBcEaXkmuuUTBARcKZgiXmicXlqFAyxJaaJlj86aopW5_CI1y1w)

## Étape 1 : Création du namespace
Pour commencer, j'ai créé un namespace `dev` afin de séparer mes ressources des autres environnements.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```


## Étape 2 : Persistent Volumes et Persistent Volume Claims

Ensuite, j'ai créé des Persistent Volumes (PV) et des Persistent Volume Claims (PVC) pour mes deux applications : MySQL et WordPress. J'ai choisi d'utiliser des volumes persistants pour assurer la persistance des données entre les mises à jour et les redémarrages de mes conteneurs.

Les PV et PVC sont configurés pour un accès en lecture et écriture unique (ReadWriteOnce). Les PVC demandent une capacité de stockage de 5 Gi pour MySQL et de 2 Gi pour WordPress.

Attention : ici les données sont stockée en local sur le cluster k8s

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

## Étape 3 : ConfigMap
J'ai créé une ConfigMap pour stocker les informations de configuration partagées entre MySQL et WordPress. Cela me permet de centraliser la configuration et de faciliter les mises à jour.

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

## Étape 4 : Déploiements
J'ai créé des déploiements pour MySQL et WordPress avec la stratégie de déploiement Recreate, qui assure que les anciens pods sont supprimés avant de créer les nouveaux.

Les déploiements utilisent les images officielles de MySQL et WordPress et récupèrent les informations de configuration depuis la ConfigMap.

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

## Étape 5 : Services
J'ai créé un service ClusterIP (backend) pour exposer l'applications MySQL à l'intérieur du cluster Kubernetes, tandis que le service que j'ai créé pour WordPress est de type NodePort (frontend) pour permettre l'accès depuis l'extérieur du cluster.

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

## Étape 6 : Ingress
À ce stade, notre application est déjà accessible depuis l'extérieur via l'adresse IP virtuelle (VIP) du cluster ou celle de l'un des nœuds, suivie du port 30080. Cependant, je souhaitais rendre mon application accessible via une URL simple. Pour ce faire, j'ai créé une règle Ingress permettant d'exposer mon application WordPress à l'aide d'un domaine personnalisé.

En effet, lorsque http://preprod-wordpress.pozos.fr sera sollicité, c'est l'ingress controller qui recevra la requête et la redirigera vers le service wordpress-svc à l'intérieur du cluster sur le port 8080. Ce service se chargera ensuite de joindre le pod à l'intérieur duquel tourne le conteneur wordpress.


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: preprod-wordpress-ingress
  namespace: dev
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: preprod-wordpress.pozos.fr
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

## Étape 7 : Déploiement
Pour déployer ces appplications avec leurs eléments infrastructure, j'utilise mon manifest de déploiement tout-en-un :

```
kubectl apply -f aio-wordpress-mysql-deployment.yml
```

Si, vous souhaitez déployer les objets les uns après les autres, rendez-vous dans le dossier *Step-by-step deployment* puis exécutez la commande `kubectl apply -f <filename>` pour chaque fichier dans l'ordre que je vous propose :

![image](https://user-images.githubusercontent.com/101605739/227746202-7c07b09f-7b34-4ec4-8b55-44dd122c7f85.png)

Vous pouvez également utiliser `kubectl` pour vérifier l'état de vos ressources déployées :

```
kubectl get all -n dev
```

## Étape 8 : Consommation de l'application

Après le déploiement, vous pouvez accéder à votre application WordPress en utilisant l'adresse http://preprod-wordpress.pozos.fr (en supposant que vous ayez correctement configuré la résolution DNS pour pointer vers votre cluster Kubernetes).

--------
[![](https://mermaid.ink/img/pako:eNp9k01zgjAQhv8Kk5POSL1z8II9OCMOlak9SA8pWZUp-Zgk2LGO_70BggSoXkh299l9s5twRRkngAJ0lFicvPU2ZZ6nyq_G3GAKSuAM9gTOnfVZQQ62BFHwCwWmVRPxvOiSvK0n9dcJT9vwB5cklqDU5L4bY8DIQCgBec4z6Ksk58wKhUWpNMhV3IJjvQruJDem95hLPeDHwjtelHSgG--sbAxS5UaYactNB1z4CDQnxjkdn9KU7g75uLzDh88SejLj5kLODvkxwqKtG0ZdNd-OtmX6VeyyYseK3du1fh9hkZsTTJqlzgrN7XD6vl3vT1qLYD4XEoTkxP8xYqJKfBH8l6uXg6wrNMq-79_vufJ2fVYR92L_yTGDuXvN3g08qWWzXHsI9LXCaFyt8TXtG8eia783izpk5-aMsnYPm3PtPjCUX_RG1u67QMrQDFGQFOfE_PzXikqRPgGFFAVmS7D8TlHKboYrBcEaXkmuuUTBARcKZgiXmicXlqFAyxJaaJlj86aopW5_CI1y1w)](https://mermaid-js.github.io/mermaid-live-editor/edit/#pako:eNp9k01zgjAQhv8Kk5POSL1z8II9OCMOlak9SA8pWZUp-Zgk2LGO_70BggSoXkh299l9s5twRRkngAJ0lFicvPU2ZZ6nyq_G3GAKSuAM9gTOnfVZQQ62BFHwCwWmVRPxvOiSvK0n9dcJT9vwB5cklqDU5L4bY8DIQCgBec4z6Ksk58wKhUWpNMhV3IJjvQruJDem95hLPeDHwjtelHSgG--sbAxS5UaYactNB1z4CDQnxjkdn9KU7g75uLzDh88SejLj5kLODvkxwqKtG0ZdNd-OtmX6VeyyYseK3du1fh9hkZsTTJqlzgrN7XD6vl3vT1qLYD4XEoTkxP8xYqJKfBH8l6uXg6wrNMq-79_vufJ2fVYR92L_yTGDuXvN3g08qWWzXHsI9LXCaFyt8TXtG8eia783izpk5-aMsnYPm3PtPjCUX_RG1u67QMrQDFGQFOfE_PzXikqRPgGFFAVmS7D8TlHKboYrBcEaXkmuuUTBARcKZgiXmicXlqFAyxJaaJlj86aopW5_CI1y1w)

# Conclusion

En conclusion, ce mini-projet Kubernetes m'a permis de mettre en place une infrastructure solide pour déployer une application WordPress couplée à une base de données MySQL. Grâce à l'utilisation des ressources Kubernetes telles que les namespaces, les volumes persistants, les ConfigMaps, les déploiements, les services et les Ingress, j'ai pu créer un environnement flexible, évolutif et facilement maintenable pour cette application.

Cette expérience m'a également permis d'appliquer les concepts et les compétences acquises lors du bootcamp DevOps eazytraining, renforçant ainsi ma compréhension des différentes étapes et ressources impliquées dans le déploiement d'une application sur un cluster Kubernetes.

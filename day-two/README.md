# Jour 2 - Architecture 3-tiers

## Step 1: Déployer l'API

### Que se passe-t-il au niveau des Pods de l’API ? Vous pouvez jeter un œil aux logs. (kubectl logs -f nomdupod)

Une erreur apparaît lors du déploiement de la ressource Deployment. En effet, l'API a besoin de se connecter à la base de données.
Or, cette connexion ne peut pas se faire pour le moment puisque j'ai ajouté des variables d'environnements aléatoires. De plus, le Pod de la base de données n'existe toujours pas.

## Step 2: Déployer la base de données

Check

## Step 3: Faire pointer l'API sur la base de données

### Quel est le nom du service de la base de données ?

Le nom du service de la base de données est `pg-service.lilian-andres`.

![](images/step-03.png)

## Step 4: Rendre votre déploiement parfait

### Gestion des Probes

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 5

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 5

startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 30
```

### Gestion des flux NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: pg
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP # open access to internal DNS
          port: 53
        - protocol: TCP # open access to internal DNS
          port: 53
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: pg-network-policy
spec:
  podSelector:
    matchLabels:
      app: pg
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - protocol: TCP
          port: 5432
```

### SecurityContext

```yaml
spec:
  securityContext:
    runAsUser: 1001 # UID de l'API
    runAsGroup: 1001 # GID de l'API
    fsGroup: 1001
  containers:
    - name: api
      image: registry.gitlab.com/takima-school/images/cdb/api:latest
      securityContext:
      allowPrivilegeEscalation: false # Éviter les escalades de privilèges
      capabilities:
        drop:
          - ALL # Supprimer les capacités Linux inutiles
```

## Step 5: C'est au tour du front

![](images/step-05.png)

### Pourquoi plus rien ne fonctionne ? Pourquoi faut-il kubectl rollout restart deployment MON_API ?

En l'état actuel, les données de la base de données ne sont pas persistantes, c'est-à-dire que lors de la destruction du Pod
de la base de données, les modifications apportées à la base sont perdues.

Il est nécessaire de rollout afin de forcer la re-création d'un Pod pour la base de données afin qu'il puisse s'alimenter en données.
Le redémarrage simple du Pod n'est pas utile puisque les données ne sont toujours pas sauvegardées et ne seront donc pas restaurées.

## Step 6: La persistance dans Kubernetes

### D’après le tableau, quel est le type d’accès implémenté par notre Storage Class EBS ? Pourquoi cela convient parfaitement pour la persistance de la base de données Postgres ?

Selon le tableau, le type d'accès implémenté par le Storage Class EBS est ReadWriteOnce, ce qui signifie que le volume peut être monté en lecture-écriture par un seul noeud.

Cela convient parfaitement à nos besoins actuels étant donné que nous n'avons qu'un seul réplica du Pod de la base de donnée, ce réplica ne peut donc se trouver que sur un seul noeud de notre cluster.
De plus, nous avons des opérations de lecture et d'écriture donc ce volume correspond exactement à nos besoins.

### Vérifiez que le PVC est bien créé. Quel est le nom du PV ?

```
kubectl get pvc

NAME    STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pg-db   Pending                                      gp2            35s
```

```
kubectl get pvc

pvc-288c7aa3-e629-4942-8993-d92e596a2e7e   3Gi        RWO            Delete           Bound    lilian-andres/pg-db                                                                                                 gp2                     21s
```

## Bonus

### Bonus 1: Admin de la DB

```
kubectl port-forward service/pgadmin 8081:80
```

### Bonus 2: Les StatefulSets

```
kubectl port-forward service/pgadmin 8081:80
```

### Bonus 2: Les StatefulSets

```
kubectl get all
NAME                                    READY   STATUS    RESTARTS   AGE
pod/pg-statefulset-0                    1/1     Running   0          92s
pod/pg-statefulset-1                    1/1     Running   0          35s
pod/pg-statefulset-2                    1/1     Running   0          29s

NAME                              READY   AGE
statefulset.apps/pg-statefulset   3/3     93s
```

```
kubectl get pvc
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pg-data-statefulset-pg-statefulset-0   Bound    pvc-72afd3bc-ce80-462f-8c5e-f8176cd19381   1Gi        RWO            gp2            2m59s
pg-data-statefulset-pg-statefulset-1   Bound    pvc-0f57b930-0a87-4d15-a3d0-0060e713be6f   1Gi        RWO            gp2            2m2s
pg-data-statefulset-pg-statefulset-2   Bound    pvc-51ee7b49-bd02-4ce6-9a3c-9a4943c5a191   1Gi        RWO            gp2            116s
```

```
kubectl get pvc
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pg-data-statefulset-pg-statefulset-0   Bound    pvc-72afd3bc-ce80-462f-8c5e-f8176cd19381   1Gi        RWO            gp2            2m59s
pg-data-statefulset-pg-statefulset-1   Bound    pvc-0f57b930-0a87-4d15-a3d0-0060e713be6f   1Gi        RWO            gp2            2m2s
pg-data-statefulset-pg-statefulset-2   Bound    pvc-51ee7b49-bd02-4ce6-9a3c-9a4943c5a191   1Gi        RWO            gp2            116s
```

Il y a également 3 PV dont le nom s'apparente à : lilian-andres/pg-data-statefulset-pg-statefulset-0.

### Bonus 3: Operator Postgres

```
kubectl create job --from=cronjob/logical-backup-formation-cdb manual-backup
```

Le backup a été validé en classe.

# Jour 1 - découverte de Kubernetes

## Pods

### Quelles sont les informations que l'on retrouve dans ce fichier ?

Le fichier `kubeconfig` contient la configuration de base de notre cluster.

```yaml
apiVersion: v1
clusters:
  - cluster:
      certificate-authority-data: DATA+OMITTED
      server: https://2A4118E0A6624EF972B22D11244C9A4D.gr7.eu-west-3.eks.amazonaws.com
    name: arn:aws:eks:eu-west-3:198605912558:cluster/cpe
contexts:
  - context:
      cluster: arn:aws:eks:eu-west-3:198605912558:cluster/cpe
      namespace: lilian-andres
      user: lilian-andres-lilian-andres-arn:aws:eks:eu-west-3:198605912558:cluster/cpe
    name: lilian-andres-lilian-andres-arn:aws:eks:eu-west-3:198605912558:cluster/cpe
current-context: lilian-andres-lilian-andres-arn:aws:eks:eu-west-3:198605912558:cluster/cpe
kind: Config
preferences: {}
users:
  - name: lilian-andres-lilian-andres-arn:aws:eks:eu-west-3:198605912558:cluster/cpe
    user:
      token: REDACTED
```

### Quelle est la différence ?

La commande `kubectl get pods -n lilian-andres` renvoie No Resources found car il n'y actuellement pas de pods dans mon cluster.
La commande `kubectl get pods -n default` renvoie une erreur car je n'ai pas les droits d'accès à ce namespace.

### Quelles sont les propriétés principales que l'on retrouve dans le fichier de configuration du pod ?

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubernetes.io/limit-ranger: "LimitRanger plugin set: cpu, memory request for container
      mynginx; cpu, memory limit for container mynginx"
  creationTimestamp: "2024-12-02T08:46:46Z"
  labels:
    run: mynginx
  name: mynginx
  namespace: lilian-andres
  resourceVersion: "194512"
  uid: a9ade137-28d7-4465-9bb9-c7ad7f9c6078
spec:
  containers:
    - image: registry.takima.io/school/proxy/nginx
      imagePullPolicy: Always
      name: mynginx
      resources:
        limits:
          cpu: 300m
          memory: 512Mi
        requests:
          cpu: 50m
          memory: 64Mi
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-tjhbk
          readOnly: true
```

## ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: unicorn-front-replicaset
  labels:
    app: unicorn-front
spec:
  template:
    metadata:
      name: unicorn-front-pod
      labels:
        app: unicorn-front
    spec:
      containers:
        - name: unicorn-front
          image: registry.takima.io/school/proxy/nginx
  replicas: 3
  selector:
    matchLabels:
      app: unicorn-front
```

### Que remarquez-vous dans la description des properties spec: template ? À quoi sert le selector: matchLabels ?

Dans le template, la partie metadata définit les métadonnées de chaque Pod du ReplicaSet.
Il est inhabituel de spécifier metadata.name dans le template d’un ReplicaSet.
Ce champ est souvent omis car Kubernetes génère automatiquement un nom unique pour chaque Pod créé.
La présence de ce champ est généralement redondante.
Les labels app: unicorn-front sont appliqués aux Pods créés par ce ReplicaSet, ce qui est essentiel pour leur identification et leur gestion.
Le selector: matchLabels établit un lien logique entre le ReplicaSet et les Pods qu'il doit gérer, garantissant que Kubernetes maintienne le bon nombre de réplicas pour l'application unicorn-front.

### Que se passe-t-il lors une deuxième création identique en impératif ?

Error from server (AlreadyExists): pods "mynginx" already exists

### Que se passe-t-il lors de cette deuxième création en déclaratif ?

Warning: resource pods/mynginx is missing the kubectl.kubernetes.io/last-applied-configuration annotation which
is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl
create --save-config or kubectl apply. The missing annotation will be patched automatically.

pod/mynginx configured

### Combien y a-t-il de pods déployés dans votre namespace ?

unicorn-front-replicaset-9rdjf 1/1 Running 0 2s
unicorn-front-replicaset-h9gqr 1/1 Running 0 2s
unicorn-front-replicaset-x7vcl 1/1 Running 0 2s

### Que se passe-t-il lors de la suppression d'un Pod du ReplicaSet ?

pod "unicorn-front-replicaset-9rdjf" deleted

Pourtant,

kubectl get pods -n lilian-andres
NAME READY STATUS RESTARTS AGE
unicorn-front-replicaset-2z6zl 1/1 Running 0 5s
unicorn-front-replicaset-h9gqr 1/1 Running 0 82s
unicorn-front-replicaset-x7vcl 1/1 Running 0 82s

### Que se passe-t-il lors de la suppression du ReplicaSet ?

Tous les Pods sont supprimés correctement dans ce cas.

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unicorn-front-deployment
  labels:
    app: unicorn-front
spec:
  replicas: 3
  selector:
    matchLabels:
      app: unicorn-front
  template:
    metadata:
      labels:
        app: unicorn-front
    spec:
      containers:
        - name: unicorn-front
          image: registry.takima.io/school/proxy/nginx:1.7.9
          ports:
            - containerPort: 80
```

### Quels sont les changements avec le ReplicaSet ?

- kind: Deployment : Utilise un Deployment qui gère automatiquement les mises à jour et crée un ReplicaSet en arrière-plan.
- image: registry.takima.io/school/proxy/nginx:1.7.9 : Spécifie une version particulière de l'image nginx (tag 1.7.9), ce qui permet un meilleur contrôle des versions déployées.
- Aucun champ metadata.name dans le template car Kubernetes génère automatiquement les noms des Pods.
- Expose le port 80 dans le conteneur.

### Combien y a-t'il de ReplicaSet ? De Pods ?

```
kubectl get all

NAME                                            READY   STATUS    RESTARTS   AGE
pod/unicorn-front-deployment-588848978b-4692h   1/1     Running   0          5s
pod/unicorn-front-deployment-588848978b-fxqzb   1/1     Running   0          5s
pod/unicorn-front-deployment-588848978b-mzcfc   1/1     Running   0          5s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/unicorn-front-deployment   3/3     3            3           5s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/unicorn-front-deployment-588848978b   3         3         3       5s
```

### Une fois terminé, combien y a-t-il de replicaset ? Combien y a-t-il de Pods ? Allez voir les logs des événements du déploiement avec kubectl describe deployments. Qu’observez-vous ?

```
kubectl get all

NAME                                            READY   STATUS    RESTARTS   AGE
pod/unicorn-front-deployment-77fcc4b7bd-9h8bw   1/1     Running   0          69s
pod/unicorn-front-deployment-77fcc4b7bd-bf692   1/1     Running   0          67s
pod/unicorn-front-deployment-77fcc4b7bd-xdwmz   1/1     Running   0          66s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/unicorn-front-deployment   3/3     3            3           4m16s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/unicorn-front-deployment-588848978b   0         0         0       4m17s
replicaset.apps/unicorn-front-deployment-77fcc4b7bd   3         3         3       70s
```

```
kubectl describe deployments
Name:                   unicorn-front-deployment
Namespace:              lilian-andres
CreationTimestamp:      Mon, 02 Dec 2024 10:52:41 +0100
Labels:                 app=unicorn-front
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=unicorn-front
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=unicorn-front
  Containers:
   unicorn-front:
    Image:         registry.takima.io/school/proxy/nginx:1.9.1
    Port:          80/TCP
    Host Port:     0/TCP
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  unicorn-front-deployment-588848978b (0/0 replicas created)
NewReplicaSet:   unicorn-front-deployment-77fcc4b7bd (3/3 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  6m45s  deployment-controller  Scaled up replica set unicorn-front-deployment-588848978b to 3
  Normal  ScalingReplicaSet  3m38s  deployment-controller  Scaled up replica set unicorn-front-deployment-77fcc4b7bd to 1
  Normal  ScalingReplicaSet  3m36s  deployment-controller  Scaled down replica set unicorn-front-deployment-588848978b to 2 from 3
  Normal  ScalingReplicaSet  3m36s  deployment-controller  Scaled up replica set unicorn-front-deployment-77fcc4b7bd to 2 from 1
  Normal  ScalingReplicaSet  3m35s  deployment-controller  Scaled down replica set unicorn-front-deployment-588848978b to 1 from 2
  Normal  ScalingReplicaSet  3m35s  deployment-controller  Scaled up replica set unicorn-front-deployment-77fcc4b7bd to 3 from 2
  Normal  ScalingReplicaSet  3m34s  deployment-controller  Scaled down replica set unicorn-front-deployment-588848978b to 0 from 1
```

### Après une nouvelle updgrade avec une image cassée, que se passe-t-il ? Pourquoi ?

```
kubectl get all
NAME                                            READY   STATUS             RESTARTS   AGE
pod/unicorn-front-deployment-77fcc4b7bd-9h8bw   1/1     Running            0          10m
pod/unicorn-front-deployment-77fcc4b7bd-bf692   1/1     Running            0          10m
pod/unicorn-front-deployment-77fcc4b7bd-xdwmz   1/1     Running            0          10m
pod/unicorn-front-deployment-84f96bbc4f-2722c   0/1     ImagePullBackOff   0          3m30s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/unicorn-front-deployment   3/3     1            3           13m

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/unicorn-front-deployment-588848978b   0         0         0       13m
replicaset.apps/unicorn-front-deployment-77fcc4b7bd   3         3         3       10m
replicaset.apps/unicorn-front-deployment-84f96bbc4f   1         1         0       3m30s
```

```
kubectl describe deployments
Name:                   unicorn-front-deployment
Namespace:              lilian-andres
CreationTimestamp:      Mon, 02 Dec 2024 10:52:41 +0100
Labels:                 app=unicorn-front
Annotations:            deployment.kubernetes.io/revision: 3
                        kubernetes.io/change-cause:
                          kubectl set image deployment.v1.apps/unicorn-front-deployment unicorn-front=registry.takima.io/school/proxy/nginx:1.91-falseimage --record...
Selector:               app=unicorn-front
Replicas:               3 desired | 1 updated | 4 total | 3 available | 1 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=unicorn-front
  Containers:
   unicorn-front:
    Image:         registry.takima.io/school/proxy/nginx:1.91-falseimage
    Port:          80/TCP
    Host Port:     0/TCP
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:  unicorn-front-deployment-588848978b (0/0 replicas created), unicorn-front-deployment-77fcc4b7bd (3/3 replicas created)
NewReplicaSet:   unicorn-front-deployment-84f96bbc4f (1/1 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  13m    deployment-controller  Scaled up replica set unicorn-front-deployment-588848978b to 3
  Normal  ScalingReplicaSet  10m    deployment-controller  Scaled up replica set unicorn-front-deployment-77fcc4b7bd to 1
  Normal  ScalingReplicaSet  10m    deployment-controller  Scaled down replica set unicorn-front-deployment-588848978b to 2 from 3
  Normal  ScalingReplicaSet  10m    deployment-controller  Scaled up replica set unicorn-front-deployment-77fcc4b7bd to 2 from 1
  Normal  ScalingReplicaSet  10m    deployment-controller  Scaled down replica set unicorn-front-deployment-588848978b to 1 from 2
  Normal  ScalingReplicaSet  10m    deployment-controller  Scaled up replica set unicorn-front-deployment-77fcc4b7bd to 3 from 2
  Normal  ScalingReplicaSet  10m    deployment-controller  Scaled down replica set unicorn-front-deployment-588848978b to 0 from 1
  Normal  ScalingReplicaSet  3m32s  deployment-controller  Scaled up replica set unicorn-front-deployment-84f96bbc4f to 1
```

Les anciens Pods ont été conservés et sont toujours actifs.
Un quatrième Pod a tenté d'être créé mais son statut indique que Kubernetes n'a pas réussi à initialiser l'image dans le container.

### Après avoir affiché l'historique des versions, Combien y a-t-il de révisions ? À quoi correspond le champ CHANGE-CAUSE ?

```
kubectl rollout history deployment.v1.apps/unicorn-front-deployment
deployment.apps/unicorn-front-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         kubectl set image deployment.v1.apps/unicorn-front-deployment unicorn-front=registry.takima.io/school/proxy/nginx:1.91-falseimage --record=true
```

Le champ CHANGE-CAUSE permet d'indiquer le type d'action / la commande à l'origine de la création de la nouvelle version.

### Après avoir lancé la commande de scaling avec 5 réplicas, Combien y a-t'il de Pods?

Il y a maintenant 5 Pods.

### Après avoir mis en pause le deployment et changé la version de l'image, que se passe-t-il au niveau ReplicaSet ?

Etant donné que la pause permet de modifier le déploiement sans déclencher de RollingUpgrade, aucun autre ReplicaSet n'a été créé.

### Après avoir suspendu la mise en pause, que se passe-t-il au niveau ReplicaSet ?

Un quatrième ReplicaSet est apparu et le RollingUpgrade s'est executé.

## Mise en situation

### Après avoir créé le Deployment, que se passe-t-il ? Pourquoi ?

Une erreur apparait, impossible de pull les images car le registry est privé.

### Décrivez ce que répond la Web App ? Actualisez votre page avec CTRL + F5. Que se passe-t-il ?

La page est composé d'un arrière-plan coloré. Au centre se trouve le texte suivant: "Je suis le Taki Pod X situé sur le noeud Y avec l'IP Z".
Lors du rechargement de la page, la couleur de l'arrière-plan change à chaque fois.

### Après avoir configuré les variables d'environnements des conteneurs, que constatez-vous sur le navigateur ?

Le texte utilise désormais les variables d'environnements.
Je suis le Taki-Pod hello-deployment-5cbf7c6677-l4dp9 situé sur le noeud ip-10-60-16-221.eu-west-3.compute.internal avec l'IP 10.60.27.210.
On remarque désormais que lors de chaque rechargement de la page, nous changeons de Pod (le service équilibrant correctement la charge).

## Bonus 1 : Rendre sa couleur secrète

```yml
- name: CUSTOM_COLOR
  valueFrom:
    secretKeyRef:
      name: hello-secret
      key: color
```

## Bonus 2 : Mon pod est-il en vie ?

```yml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
    scheme: HTTP
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 5
```

## Bonus 3 : Let my kubernetes Scale !!!

### Observez ce qu'il se passe. Que constatez-vous ?

On constate que la charge augmente assez rapidement et de manière graduelle.
Le nombre de replicas augmente également jusqu'à atteindre le chiffre de 6.

Au bout de 6 replicas, la charge atteint 50% de CPU (< à la limite que l'on a fixée). Aucun pod supplémentaire n'est donc créé.

## Bonus 4: Control my traffic !!!

### Que constatez-vous ?

On ne peut plus joindre notre pod nginx avec la nouvelle policy.
Afin de remédier à ce problème, on fait en sorte que notre pod matche avec la règle de la policy (label).

## Bonus 5: Run on one AZ

### Qu'est-ce qu'une Avaibility Zone au niveau infrastructure ?

Une Availability Zone est une unité d'isolation au sein d'une Région Cloud, conçue pour garantir la haute disponibilité et la tolérance aux pannes.
Une région peut contenir plusieurs AZ.
Isolation physique : - Chaque AZ dispose de ses propres centres de données, alimentations électriques, et systèmes de refroidissement. - Séparation physique pour réduire les risques liés aux catastrophes.
Connectivité: - Réseau privé à haut débit et faible latence entre les AZ d'une même région.
Redondance: - Infrastructures et systèmes réseaux redondants dans chaque AZ.
Disponibilité accrue: - Répartition des ressources sur plusieurs AZ garantit la continuité des services.

### Faire en sorte de tourner sur une seule AZ, par exemple eu-west-3a

Sous `template.spec`:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
                - eu-west-3a
```

## Bonus 6: Run one pod on each server

D'après la documentation:

> A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: hello-daemonset
  labels:
    app: hello
spec:
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: registry.takima.io/school/proxy/nginx:1.7.9
```

On obtient donc 6 replicas.

## Bonus 7: Change my rolling update policy

### que se passe-t-il au niveau de vos pods pendant l'update ?

Avec la stratégie Recreate, tous les Pods sont supprimés avant que les nouveaux soient créés, ce qui peut entraîner une interruption temporaire du service.

# Fil rouge 1 — Conteneurisation de TaskFlow
# Étape 3

## Choix de l'image de base

Un tag peut changer avec le temps. On utilise donc un digest, une empreinte qui permet de retrouver exactement la même image de base. Pour que l'image finale soit aussi identique, le code et les dépendances doivent rester les mêmes et la construction doit être reproductible.

On a choisi une image Node.js, car elle contient déjà ce qu'il faut pour lancer l'API. Pas besoin de partir d'Ubuntu et d'installer Node.js nous-mêmes.

## Ordre des instructions

On installe les dépendances avant de copier le code source. Comme ça, quand on modifie seulement le code, Docker garde les dépendances déjà installées grâce à son cache.

On l'a vérifié en ajoutant un commentaire dans `api/src/server.js`, puis en reconstruisant l'image. Le dossier de travail, la copie des fichiers de dépendances et leur installation étaient indiqués `CACHED`. Seule la copie des sources a été refaite. Le commentaire a ensuite été retiré.

## Dépendances de développement

L'API n'en a pas besoin pour fonctionner. On utilise donc `npm ci --omit=dev` pour installer uniquement les dépendances nécessaires à son exécution.

Pour le front, les outils de développement servent à préparer les fichiers du site, mais ils n'ont pas besoin d'être présents dans l'image finale.

## Fichier .dockerignore

Le `.dockerignore` évite d'envoyer à Docker des fichiers inutiles, comme les dépendances locales (`node_modules`), l'historique Git ou les fichiers `.env`. Cela limite le transfert et évite de copier ces fichiers dans l'image.

Après son ajout, la construction a réussi avec seulement 2,38 ko transférés lors de notre test. Le dossier `node_modules` présent dans l'image reste normal : il est créé par l'installation des dépendances dans Docker, et ne vient pas de notre machine.

## Test isolé

On a lancé l'image seule, sans base de données, en publiant son port :

```sh
docker run --rm -p 127.0.0.1::3000 taskflow-api:cache-check
```

L'API se lance, puis s'arrête après six tentatives de connexion avec le message : « Base de données injoignable sur localhost:5432 après 6 tentatives ». Les logs indiquent aussi `ECONNREFUSED`, c'est-à-dire une connexion refusée. L'erreur concerne donc bien l'absence de base de données, comme attendu à ce stade.

## Arrêt du conteneur

On a testé `docker stop` pendant que l'API attendait la base de données. Le conteneur s'est arrêté proprement en environ 0,16 seconde. Les logs affichent « Signal SIGTERM reçu : arrêt en cours », puis « Arrêt terminé ».

Le Dockerfile utilise `CMD ["node", "src/server.js"]` : Node.js reçoit directement le signal d'arrêt et l'application le prend en charge. Une attente d'environ dix secondes peut indiquer que le signal n'arrive pas à l'application ou qu'elle ne le traite pas ; Docker finit alors par forcer l'arrêt.

# Étape 4

## Compiler et servir le front

Le fichier `front/Dockerfile` utilise deux étapes. La première utilise Node.js pour installer les dépendances et compiler le front. La seconde garde uniquement Nginx et les fichiers du site générés. Les outils de compilation ne sont donc pas dans l'image finale.

Pour construire l'image depuis la racine du projet :

```sh
docker build -f front/Dockerfile -t taskflow-front .
```

## Taille de l'image

On a comparé les tailles avec :

```sh
docker images --filter reference=taskflow-front --filter reference=nginx:stable-alpine
```

Docker affiche 92,9 Mo pour le front et 93,6 Mo pour la base Nginx. Les deux restent donc autour de 93 Mo. On a aussi vérifié que l'image finale ne contient ni Node.js, ni npm, ni dossier `node_modules` dans les fichiers du site.

## Transmettre les requêtes à l'API

C'est Nginx qui reçoit les requêtes du navigateur et transmet celles qui commencent par `/api/` à l'API. Son adresse est donnée au démarrage avec la variable `API_URL`, par défaut `http://api:3000`.

Pour lancer le front :

```sh
docker network create taskflow
docker run --rm --name front --network taskflow -p 8080:80 \
  -e API_URL=http://api:3000 taskflow-front
```

Le site est accessible sur `http://localhost:8080`. Pour que les appels à l'API fonctionnent, son conteneur doit aussi être lancé sur le réseau `taskflow`, avec le nom `api`, et disposer de sa base de données. Docker permet alors à Nginx de trouver l'API grâce à ce nom. `API_URL` doit contenir l'adresse et le port, sans `/api` ni barre oblique à la fin.

On a vérifié l'accueil, le rechargement de `/a-propos` et le transfert des requêtes avec une API simulée. Le site peut démarrer sans API, mais ses appels à l'API échoueront tant qu'elle reste indisponible.

## Utiliser la même image dans plusieurs environnements

Une variable utilisée pendant la compilation reste figée dans les fichiers du site. Changer sa valeur au lancement ne les modifierait pas : il faudrait reconstruire l'image.

Ici, le navigateur utilise toujours `/api/...` et l'adresse de l'API est réglée dans Nginx au démarrage. On peut donc déployer la même image en développement, en test ou en production, en changeant simplement `API_URL`.

Ce réglage au démarrage utilise le mécanisme fourni par l'[image officielle Nginx](https://hub.docker.com/_/nginx).

# Étape 5

## Lancer les trois services

Le fichier `compose.yaml` lance la base de données, l'API et le front ensemble. L'API attend que la base soit prête, puis le front démarre. Les services se trouvent grâce à leurs noms : `db` pour PostgreSQL et `api` pour l'API.

Un fichier `.env` local a été préparé avec un mot de passe généré. Sur une nouvelle machine, copier `.env.example` vers `.env` et y choisir un mot de passe avant le premier lancement.

```sh
docker compose up -d --build --wait
```

L'application est accessible sur `http://localhost:8080`. Seul le port du front est publié sur la machine.

## Image et stockage de PostgreSQL

On utilise l'image officielle `postgres:18`, avec une version majeure explicite. Pour cette version, le volume doit être monté sur `/var/lib/postgresql` ; les données sont rangées dans `/var/lib/postgresql/18/docker`. Jusqu'à PostgreSQL 17, le point de montage par défaut était `/var/lib/postgresql/data`. Ce changement a été vérifié dans la [documentation officielle](https://hub.docker.com/_/postgres).

Le volume `postgres_data` conserve les données même si le conteneur est recréé. On l'a vérifié avec une tâche temporaire : elle était toujours présente après recréation du conteneur PostgreSQL. La création, la modification et la suppression d'une tâche via le front ont aussi été testées.

Pour voir l'état des services ou leurs logs :

```sh
docker compose ps
docker compose logs -f
```

Pour arrêter la stack en gardant les données :

```sh
docker compose down
```

## Mot de passe

Le mot de passe est uniquement dans le fichier `.env` local, ignoré par Git. Le `compose.yaml` utilise la variable `DB_PASSWORD` et le fichier `.env.example` laisse sa valeur vide.

On a contrôlé le résultat de `docker compose config` : l'API et PostgreSQL reçoivent bien le même mot de passe. Sa valeur n'apparaît dans aucun fichier destiné à être versionné. La sortie de cette commande contient le mot de passe résolu : elle ne doit donc pas être enregistrée dans Git.

## Attendre que la base soit prête

Le contrôle de santé de PostgreSQL utilise `pg_isready` pour vérifier qu'il accepte les connexions. Avec `depends_on` et `condition: service_healthy`, Compose attend ce résultat avant de démarrer l'API. Un simple `depends_on` sans cette condition ne suffit pas.

## Ports publiés

Seul le port local `8080` est publié, pour ouvrir le site dans le navigateur. Nginx transmet ensuite les appels à l'API sur le réseau Docker. Les ports `3000` de l'API et `5432` de PostgreSQL restent internes : le navigateur n'a pas besoin d'y accéder directement.

## Vérification après arrêt et relance

On a créé deux tâches, arrêté la stack avec `docker compose down`, puis relancé avec `docker compose up -d --wait`. Les deux tâches sont toujours présentes et restent visibles sur le site.

Le volume a été conservé car on n'a pas utilisé l'option `-v`. PostgreSQL écrit bien dans `/var/lib/postgresql/18/docker`, à l'intérieur du volume monté sur `/var/lib/postgresql`.

# Étape 6

## Connexion à Docker Hub

On utilise `docker login --username tlemray`, puis on saisit un Personal Access Token à la place du mot de passe. Le jeton doit autoriser la lecture et l'écriture pour publier les images. Il reste dans le gestionnaire d'identifiants Docker, jamais dans le dépôt. Voir la [documentation Docker sur les jetons](https://docs.docker.com/security/access-tokens/).

## Nom et version des images

La convention est `utilisateur/nom-image:version`. On utilise le tag `1.0.0` pour identifier cette version de l'API et du front.

Avec seulement `latest`, on ne sait pas quelle version est déployée : ce tag peut désigner une nouvelle image à chaque publication. Un tag de version rend le suivi plus clair, à condition de ne pas le réutiliser pour un autre contenu. Le digest permet d'identifier exactement l'image.

Les images du projet sont `tlemray/taskflow-api:1.0.0` et `tlemray/taskflow-front:1.0.0`. Pour les construire et les publier depuis la racine du projet :

```sh
docker build -t tlemray/taskflow-api:1.0.0 .
docker build -f front/Dockerfile -t tlemray/taskflow-front:1.0.0 .
docker push tlemray/taskflow-api:1.0.0
docker push tlemray/taskflow-front:1.0.0
```

## Lancer sans construire

Le `compose.yaml` utilise maintenant les images Docker Hub à la place des sections `build`. Sur une autre machine, il suffit de récupérer `compose.yaml` et `.env.example`, de copier ce dernier vers `.env` et d'y choisir un mot de passe pour la base. Puis :

```sh
docker compose up -d --wait
```

Docker télécharge les images manquantes. Le site est ensuite accessible sur `http://localhost:8080`.

## Dépôts publics et vérification

Les dépôts [taskflow-api](https://hub.docker.com/r/tlemray/taskflow-api) et [taskflow-front](https://hub.docker.com/r/tlemray/taskflow-front) sont publics. On a vérifié l'accès aux deux versions `1.0.0` sans connexion au compte Docker Hub. Ces images sont publiées pour Linux amd64.

On a ensuite arrêté la stack sans supprimer les volumes, supprimé les images locales de l'API, du front et de PostgreSQL, puis relancé avec `docker compose up -d --wait`. Docker a bien téléchargé les trois images, sans rien construire. Le site et l'API fonctionnent, et les tâches présentes avant le test ont été conservées.

# Fil rouge 2 — Premiers Pods TaskFlow sur Kubernetes

Le sujet fourni est intitulé « Séance 4 — Projet fil rouge : premiers Pods TaskFlow sur Kubernetes ». On réutilise le cluster `k3s-lab` et les images publiées pendant la première séance. Le fichier `k8s/k3s-config.yaml` reprend la configuration du service k3s installée dans la VM ; ce n'est pas une ressource Kubernetes à appliquer avec `kubectl`.

## Étape 1 — Namespace

Le fichier `k8s/namespace.yaml` déclare le namespace `taskflow`. Depuis Zorin, `kubectl config current-context` a confirmé le contexte `k3s-lab`. Après `kubectl apply -f k8s/namespace.yaml`, Kubernetes a répondu `namespace/taskflow created`. La commande `kubectl get namespace taskflow` a confirmé son état `Active`.

Les manifestes des Pods préciseront `metadata.namespace: taskflow`. C'est plus sûr que de dépendre du namespace par défaut du contexte : un tiers peut ainsi appliquer les fichiers sans créer les Pods par erreur dans `default` ou dans un autre namespace sélectionné sur son poste.

## Étape 2 — Pod PostgreSQL

Le manifeste `k8s/pods/db.yaml` reprend l'image `postgres:18` du service `db` de Compose, la base `taskflow` et l'utilisateur `taskflow`. Il déclare le port PostgreSQL `5432` et les labels `app: taskflow` et `component: db`.

La variable de l'image PostgreSQL à renseigner est `POSTGRES_PASSWORD`. Compose lui transmettait la valeur de `DB_PASSWORD` depuis `.env`. Pour ce TP, le manifeste utilise directement la valeur fictive `taskflow-lab-only` ; le mot de passe réel du fichier `.env` n'est pas repris.

Le volume nommé de Compose n'est pas transposé, conformément au sujet. Sans stockage persistant, supprimer le Pod fera perdre les données de cette base. Ce comportement sera testé à l'étape 5.

Après application du manifeste, le Pod `db` est `Running`, avec `1/1` conteneur prêt, aucun redémarrage et l'adresse `10.42.0.18`, sur le nœud `k3s-lab`.

Le premier affichage des logs montrait encore l'initialisation. Sans sonde de disponibilité, l'état `1/1` ne suffisait pas à confirmer que PostgreSQL acceptait déjà les connexions. On a donc vérifié :

```sh
kubectl logs -n taskflow db --tail=20
kubectl exec -n taskflow db -- pg_isready -h 127.0.0.1 -U taskflow -d taskflow
```

Les logs montrent la fin de l'initialisation, puis PostgreSQL 18.6 à l'écoute sur le port 5432 et le message `database system is ready to accept connections`. La commande `pg_isready` confirme `127.0.0.1:5432 - accepting connections`.

## Étape 3 — Pod API

Le manifeste `k8s/pods/api.yaml` utilise l'image publiée `tlemray/taskflow-api:1.0.0`, le namespace `taskflow` et les labels `app: taskflow` et `component: api`. Le conteneur expose son API sur le port 3000. Les variables `DB_NAME`, `DB_USER` et `DB_PASSWORD` correspondent à la configuration du Pod PostgreSQL, avec le même mot de passe fictif.

Le nom `db` du service Docker Compose n'est pas automatiquement transposé en nom DNS Kubernetes. Aucun Service Kubernetes n'a encore été créé : `DB_HOST` contient donc provisoirement l'adresse du Pod de base, `10.42.0.18`, avec `DB_PORT=5432`. Cette adresse peut changer lors de la recréation du Pod, ce qui obligera à adapter la configuration de l'API.

Après `kubectl apply -f k8s/pods/api.yaml`, Kubernetes a confirmé `pod/api created`. La commande d'attente a retourné `condition met`. Le Pod `api` est `Running`, avec `1/1` conteneur prêt, aucun redémarrage et l'adresse `10.42.0.19`, sur `k3s-lab`. Le Pod `db` reste `Running` à l'adresse `10.42.0.18`.

La sortie des logs était vide lors de cette première vérification. L'état `Running` seul ne suffit pas à valider les échanges avec PostgreSQL : les requêtes HTTP de l'étape suivante vérifieront le fonctionnement de l'application.

## Étape 4 — Vérification HTTP

Le port local 3001 était déjà occupé (`address already in use`). On a donc ouvert le tunnel depuis le port 18080 de Zorin vers le port 3000 du Pod API :

```sh
kubectl port-forward -n taskflow pod/api 18080:3000
```

Dans un deuxième terminal, on a exécuté :

```sh
curl -i http://localhost:18080/healthz
curl -i http://localhost:18080/api/tasks \
  -H 'Content-Type: application/json' \
  -d '{"title":"Tester TaskFlow sur Kubernetes"}'
curl -i http://localhost:18080/api/tasks
```

Résultats observés :

| Requête | Résultat |
| --- | --- |
| `GET /healthz` | `200 OK`, avec `{"status":"ok","version":"1.0.0","hostname":"api"}` |
| `POST /api/tasks` | `201 Created`, tâche nº 1 intitulée « Tester TaskFlow sur Kubernetes », avec `done: false` et `Location: /api/tasks/1` |
| `GET /api/tasks` | `200 OK`, liste contenant la tâche nº 1 précédemment créée |

Les trois réponses portent l'en-tête `X-Served-By: api`, qui correspond au nom du Pod. La création et la relecture confirment que l'API communique avec PostgreSQL ; `/healthz` vérifie seulement que le processus HTTP répond.

La tâche nº 1 est conservée pour tester la persistance des données lors de la suppression du Pod de base à l'étape suivante.

## Étape 5 — Limites de l'approche par Pods

On a supprimé le Pod `db`, puis recréé la base avec le même manifeste :

```sh
kubectl delete pod db -n taskflow
kubectl apply -f k8s/pods/db.yaml
kubectl wait -n taskflow --for=condition=Ready pod/db --timeout=180s
kubectl get pods -n taskflow -o wide
```

L'adresse du Pod `db` est passée de `10.42.0.18` à `10.42.0.20`. L'API est restée `Running`, avec aucun redémarrage, à `10.42.0.19`, mais sa variable `DB_HOST` désigne toujours l'ancienne adresse de la base.

À travers le port-forward encore ouvert, `/healthz` a répondu `200 OK`, tandis que `/api/tasks` a répondu `500 Internal Server Error`, avec `{"error":"Erreur interne"}`. Le processus API est vivant, mais les opérations sur les tâches échouent. L'état `Running` et la route de vivacité ne garantissent donc pas que la dépendance PostgreSQL est accessible.

Le manifeste API a été corrigé avec `DB_HOST=10.42.0.20` pour sa prochaine création. Ce changement du fichier ne modifie pas le Pod déjà lancé. Les constats de cette étape sont également consignés dans `docs/notes-kubernetes.md`.

Après arrêt du port-forward, on a supprimé le Pod API :

```sh
kubectl delete pod api -n taskflow
kubectl get pods -n taskflow
```

Plusieurs affichages successifs n'ont montré que le Pod `db`, toujours `Running`. Le Pod `api` n'a pas été recréé. Une seconde tentative de suppression a retourné `pods "api" not found`, confirmant son absence.

Il manque un contrôleur, par exemple un Deployment s'appuyant sur un ReplicaSet, pour maintenir un nombre de répliques souhaité. Un fichier YAML conservé sur le poste ne constitue pas à lui seul une boucle de réconciliation. Le redémarrage d'un conteneur dans un Pod existant est un mécanisme distinct, testé ci-dessous.

On a ensuite recréé l'API à partir du manifeste corrigé et rouvert le port-forward :

```sh
kubectl apply -f k8s/pods/api.yaml
kubectl wait -n taskflow --for=condition=Ready pod/api --timeout=180s
kubectl port-forward -n taskflow pod/api 18080:3000
```

Depuis un autre terminal, `/healthz` et `/api/tasks` ont tous deux répondu `200 OK`, avec `X-Served-By: api`. La liste des tâches est désormais vide (`[]`) : la tâche nº 1 présente avant suppression de la base a disparu. Cela confirme la perte des données en l'absence de volume persistant, indépendamment du problème d'adresse IP qui empêchait auparavant la lecture.

Enfin, après arrêt du port-forward, on a arrêté le processus principal du conteneur API sans supprimer son Pod :

```sh
kubectl get pod api -n taskflow
kubectl exec -n taskflow api -- sh -c 'kill 1'
kubectl get pod api -n taskflow -w
# Après Ctrl+C :
kubectl logs -n taskflow api --previous --tail=15
```

Avant cette commande, `api` était `Running` et son compteur `RESTARTS` valait déjà 1. L'affichage a ensuite montré `Completed`, brièvement `CrashLoopBackOff`, puis `Running` avec `1/1` conteneur prêt et `RESTARTS=2`. Le nom du Pod est resté `api` et son âge a continué à augmenter : c'est le conteneur qui a été relancé.

Les logs de l'exécution précédente montrent la base joignable sur `10.42.0.20:5432`, puis `Signal SIGTERM reçu : arrêt en cours` et `Arrêt terminé`. L'application a donc pris en charge le signal et s'est arrêtée proprement. Le bref `CrashLoopBackOff` correspond ici à l'attente avant relance, et non à une panne persistante après le test.

La politique Kubernetes par défaut est `restartPolicy: Always` : le kubelet relance le conteneur même après un arrêt réussi, tant que le Pod existe. Notre `compose.yaml` de séance 1 ne définit pas de politique `restart` ; Docker utilise alors `no` par défaut. Une politique Docker `always` ou `unless-stopped` permettrait une relance automatique, mais ne recréerait pas un conteneur supprimé. Voir les documentations [Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-restarts) et [Docker](https://docs.docker.com/engine/containers/start-containers-automatically/).

## Étape 6 — État final et point de contrôle

Après les tests, l'application est de nouveau fonctionnelle. Le manifeste API utilise l'adresse actuelle de la base, `10.42.0.20`. Les seules valeurs de mot de passe présentes dans les manifestes sont les valeurs fictives `taskflow-lab-only` du TP.

Les vérifications finales du 7 octobre 2026 ont donné les résultats suivants (extraits des sorties) :

```text
$ kubectl get nodes
NAME      STATUS   ROLES           AGE   VERSION
k3s-lab   Ready    control-plane   33h   v1.36.5+k3s1

$ kubectl get pods -n taskflow -o wide
NAME   READY   STATUS    RESTARTS        AGE     IP           NODE
api    1/1     Running   2 (3m8s ago)    8m17s   10.42.0.21   k3s-lab
db     1/1     Running   0               22m     10.42.0.20   k3s-lab
```

Avec `kubectl port-forward -n taskflow pod/api 18080:3000` ouvert dans un autre terminal :

```text
$ curl -i http://localhost:18080/healthz
HTTP/1.1 200 OK
X-Served-By: api
Content-Type: application/json; charset=utf-8

{"status":"ok","version":"1.0.0","hostname":"api"}

$ curl -i http://localhost:18080/api/tasks
HTTP/1.1 200 OK
X-Served-By: api
Content-Type: application/json; charset=utf-8

[]
```

Les sorties du nœud, des Pods et de `/healthz` constituent les éléments demandés pour le point de contrôle. Les fichiers à versionner sont `k8s/k3s-config.yaml`, `k8s/namespace.yaml`, `k8s/pods/db.yaml`, `k8s/pods/api.yaml`, `docs/notes-kubernetes.md` et ce compte rendu. Le push et le dépôt du point de contrôle restent à effectuer.

# Fil rouge 4 — Deployments et Services

## Étape 1 — Nettoyage et organisation

Le contexte actif est `k3s-lab`. Les anciens Pods isolés ont été supprimés :

```sh
kubectl delete pod api db -n taskflow
kubectl get pods -n taskflow
```

Les deux suppressions ont été confirmées, puis la commande de consultation a retourné `No resources found in taskflow namespace.` Le namespace est conservé.

Un Service sélectionne les Pods par leurs labels, indépendamment de leur création par un Deployment. Garder les anciens Pods avec les mêmes labels pourrait donc envoyer une partie du trafic vers eux. Pour PostgreSQL sans réplication, cela pourrait même répartir les connexions entre deux bases indépendantes.

Les anciens manifestes `k8s/pods/` sont supprimés. Le namespace est déplacé vers `k8s/app/00-namespace.yaml` : le préfixe `00-` le place avant les composants lors de l'application du répertoire. `k8s/k3s-config.yaml` reste hors de ce répertoire, car il configure le service k3s et ne décrit pas une ressource Kubernetes.

## Étape 2 — Base de données

`k8s/app/db.yaml` regroupe le Deployment et le Service PostgreSQL. Le Deployment utilise une seule réplique et la stratégie `Recreate` : lors d'une mise à jour, l'ancien Pod est arrêté avant la création du nouveau. On évite ainsi un remplacement progressif faisant cohabiter deux bases indépendantes. Cela implique une interruption pendant le remplacement et ne rend pas les données persistantes.

Le Service `db`, comme le nom utilisé dans Compose, expose le port `5432` vers le port `5432` du conteneur. Son type `ClusterIP` permet aux autres Pods de le joindre dans le cluster sans publier de port sur le poste.

L'image `postgres:18` et les variables de l'ancien Pod sont conservées. Le sujet cite `postgres:17-alpine` dans ses prérequis, mais notre TP précédent utilisait déjà PostgreSQL 18. Le mot de passe reste la valeur fictive du TP. Conformément au sujet, aucun stockage persistant, aucune sonde et aucune limite de ressources ne sont ajoutés à ce stade.

Après application des manifestes, le namespace est resté inchangé et le Deployment ainsi que le Service `db` ont été créés. `kubectl rollout status deployment/db -n taskflow --timeout=180s` a confirmé la fin du déploiement.

Résultats observés :

| Ressource | État |
| --- | --- |
| Deployment `db` | `1/1` prêt, 1 réplique à jour et disponible |
| Service `db` | `ClusterIP`, adresse `10.43.204.112`, port `5432/TCP` |
| Pod `db-fc65bd6f6-vwj9k` | `1/1 Running`, 0 redémarrage, IP `10.42.0.22`, nœud `k3s-lab` |

Les logs capturés immédiatement après le déploiement montrent encore l'initialisation de PostgreSQL (création des répertoires et fichiers de configuration). En l'absence de sonde, l'état prêt du Pod ne suffit pas à confirmer que PostgreSQL accepte déjà les connexions.

Les vérifications complémentaires (logs, EndpointSlices et `pg_isready` depuis un Pod temporaire) ont ensuite été confirmées comme réussies. La commande suivante a validé l'accès par le nom de Service `db` :

```sh
kubectl run db-check -n taskflow \
  --image=postgres:17-alpine --restart=Never --rm -i \
  --command -- pg_isready -h db -p 5432 -U taskflow -d taskflow
```

L'image du Pod temporaire fournit uniquement le client de vérification ; elle ne change pas la version du serveur PostgreSQL.

## Étape 3 — API

`k8s/app/api.yaml` contient un Deployment de deux répliques de `tlemray/taskflow-api:1.0.0` et un Service `ClusterIP` nommé `api`, exposant le port `3000` vers le même port des conteneurs. Ce nom et ce port correspondent à la destination `http://api:3000` utilisée par le front.

`DB_HOST` vaut désormais `db`, sans IP de Pod. `DB_PORT` conserve la valeur explicite `"5432"` de notre ancien manifeste. Une valeur d'environnement doit être une chaîne YAML, même lorsqu'elle représente un nombre.

Le Service `db` existant avant les Pods API, Kubernetes peut injecter la variable `DB_PORT=tcp://<ClusterIP>:5432`. Notre code attend un entier : cette valeur provoquerait une erreur au démarrage si elle n'était pas remplacée. Deux solutions sont possibles : définir explicitement `DB_PORT: "5432"` dans les variables du conteneur, ou désactiver les liens de Services via `spec.template.spec.enableServiceLinks: false` et laisser le code utiliser son port par défaut. Nous conservons la première solution, plus explicite. Il s'agit ici d'un piège anticipé, pas d'un échec observé. Voir la [documentation des variables de Service](https://kubernetes.io/docs/concepts/services-networking/service/#environment-variables) et [enableServiceLinks](https://kubernetes.io/docs/tutorials/services/connect-applications-service/#accessing-the-service).

Le Deployment et le Service `api` ont été créés. La commande `kubectl rollout status deployment/api -n taskflow --timeout=180s` a confirmé la fin du déploiement. Les deux Pods observés sur `k3s-lab` sont :

| Pod | État | Redémarrages | IP |
| --- | --- | --- | --- |
| `api-76b6687769-9cjth` | `1/1 Running` | 0 | `10.42.0.25` |
| `api-76b6687769-ztwfw` | `1/1 Running` | 0 | `10.42.0.24` |

La base reste `1/1 Running` à `10.42.0.22`. La capture du déploiement ne contient pas les logs applicatifs.

Avec `kubectl port-forward -n taskflow service/api 18080:3000`, les tests HTTP ont donné :

| Requête | Résultat |
| --- | --- |
| `GET /healthz` | `200 OK`, `status: ok`, version `1.0.0`, hostname `api-76b6687769-ztwfw` |
| `GET /api/tasks` | `200 OK`, liste vide `[]` |

Les deux réponses portent `X-Served-By: api-76b6687769-ztwfw`. La lecture des tâches valide l'accès à PostgreSQL depuis cette instance. Le port-forward sur un Service sélectionne un Pod et y reste attaché : ces appels ne prouvent pas la répartition du trafic entre les deux répliques.

Une boucle de dix appels à `http://api:3000/healthz` depuis le Pod temporaire `api-check` (image `nicolaka/netshoot`) a ensuite affiché les en-têtes `X-Served-By` : 7 réponses de `api-76b6687769-9cjth` et 3 de `api-76b6687769-ztwfw`. Les deux répliques reçoivent donc du trafic via le Service ; la répartition n'est pas nécessairement une alternance stricte. Le Pod temporaire a été supprimé automatiquement à la fin du test (`--rm`).

La version `2.0.0`, non préparée dans les TP précédents, sera créée avant le test de mise à jour.

## Étape 4 — Front : adaptation de Nginx

Avant le déploiement, la lecture de `front/nginx.conf.template` a révélé un résolveur codé en dur : `127.0.0.11`, propre au réseau Docker. Cette configuration ne convient pas aux Pods Kubernetes. Il s'agit d'un problème détecté dans le fichier, pas d'une panne observée sur le cluster.

Le template utilise désormais `proxy_pass ${API_URL};`. L'entrypoint officiel substitue `API_URL` avant de lancer Nginx ; avec `http://api:3000`, Nginx résout le nom du Service au démarrage via le résolveur système. Sans URI ajoutée à `proxy_pass`, le chemin `/api/...` est conservé. Voir la [documentation Nginx](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass).

Cela implique que le Service `api` existe avant le démarrage du front. Sinon, Nginx échoue à résoudre son hôte et quitte ; Kubernetes relance le conteneur. Une fois le Service présent, un démarrage ultérieur peut réussir. La configuration Compose attend déjà la disponibilité de l'API avant le front. Si l'adresse de l'API change dans Docker, il faut relancer le front pour la résoudre à nouveau ; dans Kubernetes, le Service conserve son adresse lors du remplacement de ses Pods.

L'image front corrigée porte le tag `tlemray/taskflow-front:1.0.1`, au lieu d'écraser la version `1.0.0` citée par le sujet et utilisée lors du premier TP. Le comportement antérieur décrit dans le fil rouge 1 concerne l'ancienne image.

Après construction locale, la configuration de l'image `1.0.1` a été testée :

```sh
docker run --rm --add-host api:127.0.0.1 \
  tlemray/taskflow-front:1.0.1 nginx -t
```

L'entrypoint a généré la configuration à partir du template, puis Nginx a retourné `syntax is ok` et `test is successful`. L'entrée d'hôte ajoutée ne sert qu'au contrôle de configuration ; ce test ne valide pas les appels HTTP vers une API réelle.

`k8s/app/front.yaml` est préparé avec deux répliques de cette image, `API_URL=http://api:3000` et un Service `LoadBalancer`. Le port du Service est `8080`, pour éviter les ports `80` et `443` utilisés par Traefik sur la VM ; le port cible des conteneurs reste `80`. Les Services API et base restent de type `ClusterIP`.

Le push de `tlemray/taskflow-front:1.0.1` a réussi. L'application de tout le répertoire avec `kubectl apply -f k8s/app/` a laissé le namespace, l'API et la base inchangés, puis créé le Deployment et le Service du front. Le rollout du front s'est terminé avec succès.

État observé après déploiement :

| Deployment | Répliques prêtes | À jour | Disponibles |
| --- | --- | --- | --- |
| `api` | 2/2 | 2 | 2 |
| `db` | 1/1 | 1 | 1 |
| `front` | 2/2 | 2 | 2 |

| Service | Type | ClusterIP | Adresse externe | Ports affichés |
| --- | --- | --- | --- | --- |
| `api` | ClusterIP | `10.43.19.25` | aucune | `3000/TCP` |
| `db` | ClusterIP | `10.43.204.112` | aucune | `5432/TCP` |
| `front` | LoadBalancer | `10.43.86.169` | `192.168.56.10` | `8080:31682/TCP` |

Dans `kube-system`, le Pod `svclb-front-a782794a-hv2kl` est `1/1 Running`, sans redémarrage. Le Pod ServiceLB de Traefik reste `2/2 Running`. L'accès prévu depuis le navigateur du poste est `http://192.168.56.10:8080` ; aucun port-forward n'est nécessaire pour cet accès. Le port `31682` est le NodePort attribué au Service, distinct du port `8080` choisi pour l'accès LoadBalancer.

Les tests dans le navigateur sur `http://192.168.56.10:8080` ont été confirmés comme réussis : création de la tâche « Tester les Services Kubernetes », passage à l'état terminé, puis relecture après actualisation avec cet état conservé. Des noms d'instances API différents ont été observés dans le pied de page au fil des actualisations.

Le parcours navigateur → front → Service API → API → Service db → PostgreSQL fonctionne donc. La conservation d'une tâche après actualisation de la page ne prouve pas la persistance après suppression du Pod de base : ce dernier n'a toujours pas de volume persistant.

## Étape 5 — Auto-réparation et mise à jour

Le Pod `api-76b6687769-9cjth` a été supprimé pendant l'utilisation de l'application :

```sh
kubectl delete pod -n taskflow api-76b6687769-9cjth
kubectl get pods -n taskflow -l component=api -w
```

Un nouveau Pod `api-76b6687769-6kqlz` est apparu automatiquement, déjà `1/1 Running` au premier affichage, avec un âge de `0s` et aucun redémarrage. L'autre réplique, `api-76b6687769-ztwfw`, est restée `1/1 Running`, sans redémarrage, avec un âge de 15 minutes.

Le ReplicaSet géré par le Deployment a rétabli les deux répliques souhaitées. C'est un nouveau Pod, avec un nouveau nom, et non un redémarrage du conteneur supprimé. Aucune erreur n'a été observée dans le navigateur pendant ce test ; les noms d'instances API affichés ont changé. Cette observation ponctuelle ne garantit pas l'absence de toute interruption, notamment sans sonde de disponibilité.

Après le test de suppression d'un Pod front, deux répliques sont présentes : `front-7ddf57fd6b-hsklm` (`1/1 Running`, 0 redémarrage, âge de 3 min 23 s) et `front-7ddf57fd6b-klhql` (`1/1 Running`, 0 redémarrage, âge de 13 min). L'écart d'âge correspond au remplacement effectué. Le nom du Pod supprimé n'est pas visible dans la capture, mais les deux répliques attendues sont disponibles.

Aucun changement visible du fonctionnement du site n'a été signalé. Le pied de page affiche les noms des Pods API, pas ceux des Pods front : il ne permet pas d'observer directement le remplacement du front.

Pour préparer le test de mise à jour, la version de `api/package.json` et celle du paquet racine dans `api/package-lock.json` passent à `2.0.0`. Aucun comportement métier ni schéma de base n'est modifié : cette version sert à distinguer les instances lors du rollout via `/api/info`, qui lit déjà la version du paquet.

L'image `tlemray/taskflow-api:2.0.0` a été construite. Le contrôle `docker run --rm tlemray/taskflow-api:2.0.0 node -p "require('./package.json').version"` a retourné `2.0.0`, puis le push sur Docker Hub a réussi.

Le manifeste API a été passé à `2.0.0` et appliqué pendant une boucle de requêtes vers `http://192.168.56.10:8080/api/info`. La capture montre la transition suivante :

1. Réponses de la version `1.0.0` avec `HTTP 200`.
2. Une page Nginx `502 Bad Gateway`, avec `HTTP 502`.
3. Une erreur curl 28 : délai dépassé après environ 3 secondes, sans octet reçu. L'affichage `HTTP 000` signifie qu'aucun code HTTP n'a été reçu ; ce n'est pas un statut envoyé par le serveur.
4. Retour des réponses `HTTP 200` avec la version `2.0.0`, depuis deux nouvelles instances du ReplicaSet `api-5cbd67c6c6`.

La mise à jour a donc atteint la nouvelle version, mais pas sans interruption. La capture ne montre pas d'alternance de réponses `1.0.0` et `2.0.0` : elle ne suffit pas à démontrer leur coexistence en train de servir des requêtes, même si un RollingUpdate peut faire cohabiter les anciens et les nouveaux Pods.

L'absence de readinessProbe est une explication compatible avec les erreurs observées : Kubernetes peut considérer un conteneur prêt avant que notre API ait terminé sa connexion à PostgreSQL et commencé à écouter. Du trafic peut alors être envoyé trop tôt et les anciennes répliques retirées trop rapidement. Les logs Nginx et les événements seraient nécessaires pour attribuer précisément chaque échec, notamment pendant le retrait des anciennes instances.

Pour expliciter une stratégie prudente, on peut définir `maxUnavailable: 0` et `maxSurge: 1`. Avec nos deux répliques, les valeurs par défaut de 25 % produisent déjà ces limites après arrondi : les écrire seules ne corrigerait donc pas l'interruption constatée. Un `minReadySeconds` positif, par exemple 10, retarderait le retrait des anciennes répliques, mais n'empêcherait pas à lui seul un Service d'envoyer du trafic vers un nouveau Pod déjà déclaré prêt. Une sonde de disponibilité adaptée reste nécessaire pour vérifier l'aptitude à servir ; conformément au sujet, aucune sonde n'est ajoutée ici. Voir les [paramètres de Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy).

Le fichier API a été remis sur l'image `1.0.0`, puis `kubectl rollout undo deployment/api -n taskflow` a confirmé `rolled back`. La commande de suivi a ensuite annoncé la fin du rollout. La requête HTTP exécutée immédiatement après a toutefois reçu une page Nginx `502 Bad Gateway`.

`kubectl diff -f k8s/app/api.yaml` n'a affiché aucune différence. La commande de rollback a aussi averti qu'elle ne mettait pas à jour l'annotation `kubectl.kubernetes.io/last-applied-configuration`, utilisée par `kubectl apply`. Le manifeste en `1.0.0` a ensuite été réappliqué pour actualiser cette référence : le Deployment a retourné `configured` et le Service `unchanged`.

Les logs des Pods `api-76b6687769-8cb22` et `api-76b6687769-b2sjw` confirment le démarrage de TaskFlow API `1.0.0`, la base joignable sur `db:5432` avec le schéma vérifié, puis l'écoute sur `0.0.0.0:3000`. Une nouvelle requête à `/api/info` via le front a répondu `200 OK`, avec la version `1.0.0` et `X-Served-By: api-76b6687769-b2sjw`. Le retour arrière est donc confirmé et l'erreur HTTP précédente était transitoire.

Avant suppression du Pod de base, `GET /api/tasks` via le front a répondu `200 OK`, avec trois tâches : nº 1 « Tester les Services Kubernetes » terminée, nº 2 « test » non terminée et nº 3 « d » non terminée. La réponse provenait de `api-76b6687769-8cb22`.

Le Pod PostgreSQL `db-fc65bd6f6-vwj9k`, à l'adresse `10.42.0.22`, a ensuite été supprimé. Le Deployment l'a remplacé automatiquement par `db-fc65bd6f6-vq2zm`, affiché `1/1 Running`, sans redémarrage, à l'adresse `10.42.0.36` sur `k3s-lab`. Son âge était de 1 seconde au moment de la capture.

Le changement de nom et d'IP confirme le remplacement du Pod. La configuration de l'API n'a pas été modifiée (`DB_HOST=db`). La capture suivante montre `GET /api/tasks` en `500 Internal Server Error`, avec le corps `{"error":"Erreur interne"}` et `X-Served-By: api-76b6687769-8cb22`.

Les logs de cette instance montrent d'abord une connexion inactive interrompue lors de l'arrêt de PostgreSQL, puis l'erreur SQL `relation "tasks" does not exist` lors de la nouvelle lecture. Cette erreur prouve que l'API atteint de nouveau PostgreSQL : le problème est l'absence de la table, pas l'ancienne adresse IP. Le Service `db` permet donc de retrouver la base après remplacement de son Pod sans modifier `DB_HOST`. Les sorties de `pg_isready` et des EndpointSlices ne figurent pas dans cette capture.

Sans volume persistant, le nouveau Pod PostgreSQL repart avec une base initialisée sans notre schéma applicatif ni les trois tâches précédentes. Notre API crée la table dans `initDatabase()`, appelé uniquement au démarrage du serveur ; les processus API restés en place ne relancent pas cette initialisation lors d'une simple reconnexion.

La remise en état a été réalisée avec `kubectl rollout restart deployment/api -n taskflow`, puis le suivi du rollout a confirmé sa réussite. Les logs du nouveau Pod `api-7f85f86c7d-55bhd` montrent la version `1.0.0`, la base joignable sur `db:5432`, le schéma vérifié et l'écoute sur `0.0.0.0:3000`. Les erreurs affichées pour l'ancien Pod `api-76b6687769-8cb22` précèdent son arrêt propre par SIGTERM ; elles ne décrivent pas l'état de la nouvelle instance.

La vérification finale de `/api/tasks` via le front a répondu `200 OK` avec `[]`, depuis `api-7f85f86c7d-fn274`. L'application est rétablie, mais les trois tâches précédentes ont disparu. Le redémarrage de l'API a recréé le schéma, sans restaurer les données. Un stockage persistant sera nécessaire pour conserver celles-ci indépendamment du Pod PostgreSQL.

## Étape 6 — Notes et point de contrôle

Les observations sont consignées dans `docs/notes-kubernetes.md` : remplacement automatique des Pods, accès stable par les Services, interruptions pendant les rollouts sans sonde, piège de `DB_PORT` évité et perte des données de PostgreSQL sans volume persistant. Les manifestes ne contiennent que le mot de passe fictif `taskflow-lab-only`.

Si un Service LoadBalancer restait en `EXTERNAL-IP: <pending>`, il faudrait examiner les Pods `svclb-...` et leurs événements dans `kube-system`, notamment un conflit de port hôte empêchant leur placement. Si l'accès fonctionnait depuis la VM mais pas depuis le poste, il faudrait vérifier le pare-feu de la VM (notamment UFW) et la connectivité réseau. Ces problèmes n'ont pas été rencontrés pendant notre déploiement du front.

Les manipulations de l'étape 5 sont terminées. Le contrôle final avec `kubectl get deployments,services -n taskflow` donne :

```text
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/api    2/2     2            2           46m
deployment.apps/db     1/1     1            1           54m
deployment.apps/front  2/2     2            2           37m

NAME           TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
service/api    ClusterIP      10.43.19.25     <none>          3000/TCP         46m
service/db     ClusterIP      10.43.204.112   <none>          5432/TCP         54m
service/front  LoadBalancer   10.43.86.169    192.168.56.10   8080:31682/TCP   37m
```

Toutes les répliques attendues sont disponibles. Une nouvelle tâche « Tache test », non terminée, a été créée après remise en état. La capture finale montre l'URL `192.168.56.10:8080`, cette tâche et l'instance `api-7f85f86c7d-fn274` en pied de page.

![Point de contrôle final : TaskFlow sur Kubernetes](docs/point-controle-fil-rouge-4.png)

Éléments du rendu : la sortie de commande ci-dessus, la [capture navigateur](docs/point-controle-fil-rouge-4.png) et le [répertoire des manifestes sur GitHub](https://github.com/getill/taskflow/tree/main/k8s/app). Le dépôt du point de contrôle dans le fil du cours reste à effectuer.

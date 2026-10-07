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

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

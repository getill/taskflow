# Séance 1
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

# Séance 2 — TP Kubernetes

## Partie 1 — Fondamentaux et orchestration avec K3s

Périmètre demandé : étapes 1 à 7 incluses. L'étape 8 et les suivantes ne sont pas traitées.

### Étapes 1 à 3 — Préparation, installation et accès distant

On a préparé le cluster en suivant le guide d'installation avant de commencer les exercices du TP. La VM VirtualBox `k3s-lab` utilise Ubuntu Server 24.04.5 LTS, en architecture amd64. Elle possède deux cartes réseau :

| Interface dans la VM | Mode VirtualBox | Adresse | Usage |
| --- | --- | --- | --- |
| `enp0s3` | NAT | `10.0.2.15/24` | Accès à Internet |
| `enp0s8` | Réseau privé hôte (`vboxnet0`) | `192.168.56.10/24` | Accès depuis Zorin |

L'accès SSH par clé fonctionne avec `ssh illy@192.168.56.10`. Le système a été mis à jour, l'horloge est synchronisée et UFW est inactif. Un instantané `avant-k3s` a été pris avant l'installation du cluster.

On a créé `/etc/rancher/k3s/config.yaml` dans la VM avant de lancer le script d'installation :

```yaml
node-ip: 192.168.56.10
flannel-iface: enp0s8
tls-san:
  - 192.168.56.10
write-kubeconfig-mode: "0644"
```

Le nœud annonce ainsi l'adresse du réseau privé, également présente dans son certificat. Le mode `0644`, utilisé ici pour le laboratoire, rend le kubeconfig administrateur lisible par les utilisateurs de la VM.

K3s est installé en version `v1.36.5+k3s1`. Le service est actif et le nœud `k3s-lab` est `Ready`, avec `192.168.56.10` comme `INTERNAL-IP`. Les composants système sont `Running` et les tâches d'installation de Traefik sont `Completed`.

Sur Zorin, on utilise `kubectl v1.36.5` et `Helm v4.3.0`. Le kubeconfig est enregistré dans `~/.kube/k3s-lab.yaml`, avec des droits `600`. Contrairement au nom `~/.kube/config` proposé dans le TP, on utilise un fichier dédié, sélectionné par cette ligne dans `~/.zshrc` :

```sh
export KUBECONFIG="$HOME/.kube/k3s-lab.yaml"
```

Le contexte a été renommé `k3s-lab` et l'adresse du serveur remplacée par `https://192.168.56.10:6443`. Le kubeconfig reste hors du dépôt car il contient les accès administrateur.

Les vérifications suivantes ont été réalisées depuis Zorin :

| Vérification | Résultat observé |
| --- | --- |
| `kubectl config current-context` | `k3s-lab`, également dans un nouveau terminal |
| `kubectl get nodes -o wide` | Nœud `Ready`, adresse `192.168.56.10` |
| `kubectl cluster-info` | API accessible sur `https://192.168.56.10:6443` |
| `kubectl auth can-i '*' '*'` | `yes` : droits administrateur |
| Pod `test` avec `nginx:1.28-alpine`, puis `kubectl port-forward pod/test 8080:80` | Page « Welcome to nginx! » accessible sur `http://localhost:8080` |
| `curl -i http://192.168.56.10` | `404 Not Found` : Traefik répond, sans route applicative configurée |
| `helm list -A` | `traefik` et `traefik-crd` en état `deployed` dans `kube-system` |

Le Pod `test` utilisé pour valider l'installation a ensuite été supprimé. Il est distinct du Pod `pod-test` demandé à l'étape 4 du TP.

### Étape 4A — Création impérative d'un Pod

Depuis le terminal connecté en SSH à la VM, on a exécuté :

```sh
kubectl run pod-test --image=nginx:alpine
kubectl get pods -o wide
```

Le Pod est d'abord apparu en `ContainerCreating`, puis en `Running`, avec `1/1` conteneur prêt et aucun redémarrage. Son adresse IP interne observée est `10.42.0.10`, sur le nœud `k3s-lab`. La création est impérative : on donne directement l'ordre à Kubernetes de créer un Pod, sans fichier YAML. L'option `-o wide` permet notamment de voir son adresse IP interne et le nœud qui l'héberge.

Lors du passage au fichier YAML, la commande `cd` vers le projet a échoué dans la VM : ce dossier et `pod-web.yaml` se trouvent sur Zorin. Il faut donc exécuter la suite depuis le terminal du poste, avec le kubeconfig déjà configuré pour joindre le cluster distant.

### Étape 4B — Création déclarative d'un Pod

Le fichier `pod-web.yaml` décrit un Pod nommé `pod-web`, portant le label `component: frontend`. Il contient un conteneur `nginx-container` utilisant l'image `nginx:1.25`, avec le port 80 déclaré.

Depuis la racine du projet sur Zorin, on a exécuté :

```sh
kubectl apply -f pod-web.yaml
kubectl get pods -o wide
```

Kubernetes a répondu `pod/pod-web created`. Le premier affichage montrait `ContainerCreating`, juste après sa création. Contrairement à la méthode impérative, le fichier YAML conserve la description de l'état souhaité et peut être versionné avec le projet.

### Étape 4C — Inspection et diagnostic

On a inspecté les événements et les logs avec :

```sh
kubectl describe pod pod-web
kubectl logs pod-web
```

La section `Events` montre les étapes suivantes, toutes de type `Normal` :

| Événement | Observation et signification |
| --- | --- |
| `Scheduled` | Le scheduler a affecté `default/pod-web` au nœud `k3s-lab`. |
| `Pulling` | Le kubelet a lancé le téléchargement de `nginx:1.25`. |
| `Pulled` | L'image a été téléchargée avec succès en environ 5,3 secondes. |
| `Created` | Le conteneur a été créé. |
| `Started` | Le conteneur a démarré. |

Les logs affichent `Configuration complete; ready for start up`, puis `nginx/1.25.5` et `start worker processes`, suivi du lancement de deux processus workers. Ils confirment le démarrage de Nginx sans erreur visible. Le tag `nginx:1.25` utilisé dans cet exercice correspond donc ici à la version 1.25.5.

`get` résume l'état des Pods ; `describe` détaille leur configuration et les événements Kubernetes ; `logs` affiche les messages produits par le conteneur.

### Étape 5 — Erreur d'image et diagnostic

On a créé `pod-erreur.yaml` avec l'image volontairement inexistante `nginx:9.9.9`, puis exécuté :

```sh
kubectl apply -f pod-erreur.yaml
kubectl get pod pod-erreur -w
# Après Ctrl+C pour arrêter l'observation :
kubectl describe pod pod-erreur
```

Le Pod a été accepté et affecté au nœud `k3s-lab` (`Scheduled`), mais le téléchargement de l'image a échoué. Les événements affichent `ErrImagePull`, `ImagePullBackOff` et le message précis `docker.io/library/nginx:9.9.9: not found`, avec le code `NotFound`.

La cause est donc le tag inexistant, et non une erreur de syntaxe YAML ou d'ordonnancement. Kubernetes accepte la description du Pod avant que le kubelet tente de récupérer l'image. `ErrImagePull` signale l'échec du téléchargement ; `ImagePullBackOff` indique que Kubernetes attend avant une nouvelle tentative, en espaçant progressivement les essais. Les événements montrent déjà deux tentatives de téléchargement.

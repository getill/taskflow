# Fil rouge 2 — Premiers Pods TaskFlow sur Kubernetes

[Retour au sommaire](../../ANSWERS.md)

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


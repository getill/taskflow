# Premiers Pods TaskFlow — Limites observées

## Adresse de la base et disponibilité de l'application

Après suppression et recréation du Pod `db` avec le même manifeste, son adresse est passée de `10.42.0.18` à `10.42.0.20`. Le Pod API est resté `Running` à `10.42.0.19`, sans redémarrage, mais son `DB_HOST` contenait encore `10.42.0.18`.

L'API a continué à répondre `200 OK` sur `/healthz`, alors que `/api/tasks` renvoyait `500 Internal Server Error`. Le processus fonctionne, mais ne peut plus accéder à sa base à l'adresse configurée.

Dans cette configuration provisoire, il faut actualiser `DB_HOST` et recréer le Pod API. Un Service Kubernetes apportera une adresse stable et un nom DNS pour joindre la base sans inscrire l'IP éphémère d'un Pod dans la configuration de l'API. Une sonde de disponibilité utilisant `/readyz` permettra de distinguer une API capable de servir les requêtes d'un processus simplement vivant.

## Perte des données sans volume persistant

Le manifeste PostgreSQL ne contient aucun volume persistant. La tâche nº 1, « Tester TaskFlow sur Kubernetes », était présente avant la suppression du Pod `db`. Après recréation de l'API avec `DB_HOST=10.42.0.20`, `/healthz` et `/api/tasks` ont répondu `200 OK`, mais la liste des tâches était vide (`[]`). La perte des données est donc confirmée après rétablissement de l'accès à la base.

Un stockage persistant, monté via un PersistentVolumeClaim, permettra de conserver les données indépendamment du cycle de vie du Pod.

## Suppression d'un Pod sans contrôleur

Après `kubectl delete pod api -n taskflow`, plusieurs appels à `kubectl get pods -n taskflow` n'ont montré que `db`. L'API n'a pas été recréée automatiquement. Une nouvelle demande de suppression a renvoyé `NotFound`, car le Pod n'existait plus.

Le Pod a été créé directement, sans ReplicaSet ni Deployment. Aucun contrôleur ne maintient une réplique de l'API après sa suppression. Pour recréer l'API dans ce TP, il faut réappliquer son manifeste. Un Deployment, via son ReplicaSet, permettra ensuite de réconcilier le nombre de Pods avec l'état souhaité. Ce remplacement d'un Pod doit être distingué du redémarrage d'un conteneur au sein d'un Pod existant.

## Arrêt du processus et redémarrage du conteneur

La commande `kubectl exec -n taskflow api -- sh -c 'kill 1'` a envoyé `SIGTERM` au processus principal de l'API. Les logs précédents confirment un arrêt propre : `Signal SIGTERM reçu : arrêt en cours`, puis `Arrêt terminé`.

Le Pod a conservé le nom `api` et son âge a continué à augmenter. Son compteur `RESTARTS`, déjà à 1 avant cette manipulation, est passé à 2. Après les affichages transitoires `Completed` et `CrashLoopBackOff`, il est revenu à `Running`, avec `1/1` conteneur prêt.

Avec la politique par défaut `restartPolicy: Always`, le kubelet relance le conteneur dans le Pod existant, y compris après un arrêt réussi. Cette surveillance locale fonctionne déjà sans Deployment. Elle ne remplace pas le contrôleur nécessaire pour recréer un Pod supprimé. Voir le [cycle de vie des Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-restarts).

Dans notre `compose.yaml`, aucune politique `restart` n'est définie : le comportement Docker par défaut est `no`. Les politiques Docker `always` et `unless-stopped` permettent aussi de relancer un conteneur arrêté, mais pas de recréer un conteneur supprimé. Voir les [politiques de redémarrage Docker](https://docs.docker.com/engine/containers/start-containers-automatically/).

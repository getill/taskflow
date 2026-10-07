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

## État final vérifié du TP sur les Pods isolés

Le 7 octobre 2026, après les manipulations, le nœud `k3s-lab` est `Ready`. Dans le namespace `taskflow`, les Pods `db` (`10.42.0.20`) et `api` (`10.42.0.21`) sont `Running`, avec `1/1` conteneur prêt. Le compteur de redémarrages de l'API vaut 2 ; celui de la base vaut 0.

Le port-forward `18080:3000` permet de joindre `/healthz` et `/api/tasks`, qui répondent tous deux `200 OK`, avec `X-Served-By: api`. La liste des tâches est vide après le test de perte des données. Le manifeste API contient la bonne adresse de base ; il faudra la réactualiser si le Pod `db` est recréé avec une autre IP, jusqu'à l'introduction d'un Service.

## Fil rouge 4 — Services et accès de bout en bout

Les Pods isolés ont été remplacés par trois Deployments : deux répliques pour l'API, deux pour le front et une pour PostgreSQL. Les Services `api` et `db` sont de type `ClusterIP` ; le front utilise un `LoadBalancer` sur `192.168.56.10:8080`. L'API utilise désormais `DB_HOST=db`.

Les dix appels depuis un Pod temporaire au Service `api` ont atteint les deux instances (7 réponses et 3 réponses). Le navigateur permet de créer une tâche, de la terminer et de la retrouver après actualisation. Le pied de page affiche des instances API différentes.

Après suppression de `api-76b6687769-9cjth`, le ReplicaSet du Deployment a créé `api-76b6687769-6kqlz` automatiquement. Les deux répliques sont revenues à `1/1 Running` ; l'ancienne réplique `api-76b6687769-ztwfw` est restée en place. Aucune erreur n'a été observée dans le navigateur. Contrairement au Pod isolé supprimé dans le TP précédent, le nombre de répliques est maintenant maintenu par un contrôleur. Le nouveau nom et l'âge de `0s` distinguent ce remplacement d'un simple redémarrage de conteneur.

Le template Nginx a été adapté pour supprimer le DNS spécifique à Docker (`127.0.0.11`). L'image front `1.0.1` utilise le résolveur système au démarrage pour trouver `api`, dont le Service doit donc déjà exister.

Le piège des variables de Service est évité par la valeur explicite `DB_PORT="5432"`, déjà présente dans notre ancien manifeste. Sans elle, un Service `db` créé avant le Pod peut injecter `DB_PORT=tcp://<ClusterIP>:5432`, incompatible avec l'entier attendu par l'API. L'autre solution est `enableServiceLinks: false` dans la spécification du Pod. Aucun échec lié à ce piège n'a été observé dans notre déploiement.

Le test sur le front retrouve également deux Pods `1/1 Running`, sans redémarrage : `front-7ddf57fd6b-hsklm` âgé de 3 min 23 s et `front-7ddf57fd6b-klhql` âgé de 13 min. Cet écart d'âge correspond au remplacement demandé. Aucun problème visible n'a été signalé dans le navigateur. Le pied de page identifie uniquement l'API ; les noms des Pods front se consultent avec `kubectl`.

Les limites restent présentes : mot de passe fictif en clair dans les manifestes, aucune sonde de disponibilité et aucune persistance de PostgreSQL. Recharger une page ne supprime pas le Pod de base et ne teste donc pas la persistance de ses données.

## Mise à jour progressive : interruption observée

Pendant le passage de l'API `1.0.0` à `2.0.0`, les appels à `/api/info` via le front ont montré des réponses 200 en version 1, une réponse Nginx 502, un dépassement du délai curl de 3 secondes (`HTTP 000` : aucun statut reçu), puis des réponses 200 en version 2 depuis deux nouvelles instances. La capture ne montre pas d'alternance des deux versions servies.

Un Deployment disponible ne garantit pas une application prête à recevoir du trafic sans readinessProbe. Notre API initialise sa connexion à la base avant d'écouter : cette fenêtre de démarrage peut expliquer les erreurs, sans permettre de déterminer leur cause précise pour chaque requête à partir de la seule capture.

Avec deux répliques, les paramètres par défaut de RollingUpdate équivalent déjà à `maxUnavailable: 0` et `maxSurge: 1`. Les expliciter ne suffit donc pas à éliminer l'interruption. `minReadySeconds` peut ralentir le retrait des anciennes répliques, mais ne filtre pas le trafic d'un Service vers un Pod déclaré prêt. Une sonde de disponibilité, exclue de cette séance, permettrait de vérifier la capacité réelle à servir avant d'envoyer du trafic.

Le retour arrière avec `kubectl rollout undo` a été suivi d'une réponse 502 immédiate, puis d'un retour à `200 OK` en version `1.0.0`. Les logs des deux instances confirment la connexion à `db:5432` et l'écoute HTTP. Le manifeste a été remis sur `1.0.0` ; `kubectl diff` n'affichait aucune différence. Un nouvel `apply` a aussi actualisé l'annotation de configuration précédente que `rollout undo` ne met pas à jour.

## Remplacement de PostgreSQL : accès rétabli, schéma perdu

Avant suppression, `/api/tasks` répondait 200 avec trois tâches. Après suppression de `db-fc65bd6f6-vwj9k` (`10.42.0.22`), le Deployment a créé `db-fc65bd6f6-vq2zm` (`10.42.0.36`), affiché `1/1 Running`. L'API conserve `DB_HOST=db`.

La lecture suivante renvoie 500 ; les logs API contiennent `relation "tasks" does not exist`. La base est donc joignable via le Service malgré le changement d'IP, mais sa table a disparu en l'absence de stockage persistant. Le Deployment recrée le Pod et le Service maintient l'accès ; aucun des deux ne conserve les données.

L'API crée le schéma uniquement au démarrage. Après `kubectl rollout restart deployment/api -n taskflow`, le rollout a réussi et les nouvelles instances ont initialisé le schéma. `/api/tasks` a répondu `200 OK` avec `[]`, depuis `api-7f85f86c7d-fn274`. L'application fonctionne à nouveau, mais les trois anciennes tâches sont perdues. Un volume persistant, traité dans une séance ultérieure, répondra au problème de perte des données.

## Bilan des trois constats initiaux

| Constat avec les Pods isolés | Apport de Deployment et Service |
| --- | --- |
| Un Pod supprimé ne revient pas. | Le ReplicaSet géré par le Deployment rétablit le nombre de répliques ; observé pour l'API, le front et la base. |
| L'IP d'un Pod change après remplacement. | Le Service fournit une adresse et un nom stables ; l'API a rejoint la nouvelle base avec `DB_HOST=db` inchangé. |
| Un conteneur arrêté redémarre dans le même Pod. | Cela reste le rôle du kubelet et de `restartPolicy: Always`. Le Deployment complète ce mécanisme en remplaçant les Pods supprimés ; le Service dirige le trafic vers ses cibles disponibles. |

Ces mécanismes ne remplacent ni la persistance des données, ni une sonde de disponibilité applicative, ni la gestion des secrets. Le test de mise à jour a montré une interruption malgré deux répliques ; le test de suppression de PostgreSQL a montré une perte de données malgré sa recréation automatique.

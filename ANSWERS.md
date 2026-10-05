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

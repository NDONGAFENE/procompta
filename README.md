# Pro-Compta — Serveur réseau (version PostgreSQL)

Développé par **Pro-Technologie**

Cette version remplace le stockage fichier du squelette précédent par une vraie
base de données **PostgreSQL**. C'est la base sérieuse sur laquelle on construira
la suite (modules comptables, paie, etc.).

Elle démontre, avec du vrai code testé :
- l'**authentification** (login + mot de passe chiffré bcrypt) ;
- la **licence serveur pour un nombre exact d'utilisateurs** : le comptage des
  connexions se fait dans une **transaction verrouillée**, pour qu'aucune
  connexion simultanée ne dépasse le quota.

---

## Étape 1 — Installer PostgreSQL (une seule fois)

1. Téléchargez l'installateur Windows sur **https://www.postgresql.org/download/windows/**
   (bouton « Download the installer »).
2. Installez-le. Pendant l'installation :
   - notez bien le **mot de passe** que vous donnez à l'utilisateur `postgres` ;
   - laissez le port par défaut **5432**.
3. À la fin, l'outil **pgAdmin** est installé : il sert à gérer la base.

## Étape 2 — Créer la base et l'utilisateur de l'application

Ouvrez **pgAdmin** (ou l'outil « SQL Shell / psql ») et exécutez ces deux
commandes (adaptez le mot de passe si vous voulez) :

```sql
CREATE ROLE procompta LOGIN PASSWORD 'procompta';
CREATE DATABASE procompta OWNER procompta;
```

## Étape 3 — Configurer la connexion

Dans ce dossier, copiez le fichier **`.env.example`** en **`.env`** et vérifiez
les valeurs (elles correspondent à l'étape 2) :

```
PGHOST=localhost
PGPORT=5432
PGUSER=procompta
PGPASSWORD=procompta
PGDATABASE=procompta
PORT=3000
```

## Étape 4 — Lancer le serveur

Ouvrez une invite de commandes **dans ce dossier** (barre d'adresse de
l'explorateur → tapez `cmd` → Entrée), puis :

```
npm install
npm run init
npm run seed
npm start
```

- `npm install` installe les composants (Express, pg, bcrypt). Aucune compilation.
- `npm run init` crée les tables (exécute `schema.sql`).
- `npm run seed` insère les données de démonstration (licence de 3 utilisateurs + 4 comptes).
- `npm start` démarre le serveur sur http://localhost:3000

Vérifiez dans un navigateur : http://localhost:3000/api/health
(doit afficher `"db": "PostgreSQL"`).

## Comptes de test

| Login | Mot de passe | Rôle |
|-------|--------------|------|
| admin | admin123 | admin |
| compta1 | compta123 | comptable |
| paie1 | paie123 | paie |
| consult1 | consult123 | consultation |

Licence de démo : **3 utilisateurs simultanés**.

## Tester le comptage de licences

Laissez `npm start` tourner. Dans une **deuxième** invite de commandes (même
dossier) :

```
node tester-api.js
```

Résultat attendu : 3 connexions acceptées, la 4ᵉ refusée (« Licence pleine »),
puis la place réutilisée après une déconnexion.

---

## Les fichiers

| Fichier | Rôle |
|---------|------|
| `schema.sql` | Définition des tables (companies, licenses, users, sessions) |
| `db.js` | Connexion à PostgreSQL (lit `.env`) |
| `init-db.js` | Crée les tables (`npm run init`) |
| `seed.js` | Données de démonstration (`npm run seed`) |
| `server.js` | L'API (login, logout, license, me, sessions, health) |
| `tester-api.js` | Petit script de test du comptage de licences |
| `.env.example` | Modèle de configuration (à copier en `.env`) |

## Régler le nombre d'utilisateurs d'une licence

Dans `seed.js`, la valeur `3` (champ `max_users`) fixe le nombre d'utilisateurs.
En vente réelle, ce sera le palier acheté (5, 10, 25, 50…). On pourra aussi le
modifier directement en base via pgAdmin.

## Prochaines étapes

1. Brancher la **saisie d'écritures** sur le serveur (cœur comptable en réseau).
2. Relier les **modules métiers** (paie, facturation, stocks…).
3. **Administration** des licences et des utilisateurs (interface).
4. **Sécurité de production** : HTTPS, jetons signés (JWT), sauvegardes
   automatiques de la base, journal des accès.
5. **Déploiement** : serveur local (dans l'entreprise) ou cloud.

En cas d'erreur au démarrage, copiez le message affiché dans l'invite de
commandes : on le corrige ensemble. L'erreur la plus fréquente est une mauvaise
valeur dans `.env` (mot de passe ou nom de base).

---

## Synchronisation des données (v0.3.0)

Le serveur stocke désormais le **document comptable complet** de la société
(table `company_data`), partagé par tous les postes. Routes ajoutées :

| Méthode | Route | Rôle |
|---------|-------|------|
| GET | `/api/data` | Récupère les données partagées de la société |
| GET | `/api/data/version` | Version actuelle (vérification périodique) |
| POST | `/api/data` | Enregistre les données (verrou optimiste par version) |

Fonctionnement : chaque enregistrement incrémente un **numéro de version**. Si un
poste tente d'enregistrer avec une version périmée (parce qu'un autre poste a
enregistré entre-temps), le serveur renvoie **409 (conflit)** avec les données à
jour ; le poste se réaligne automatiquement. Cela empêche tout écrasement
silencieux. Les postes vérifient la version toutes les 10 secondes et se mettent
à jour quand quelqu'un a enregistré.

Limite : synchronisation « au niveau du document » (dernier enregistrement qui
gagne, protégé par version). Pour de très nombreux éditeurs simultanés sur les
mêmes écritures, un verrouillage plus fin viendra dans une version ultérieure.

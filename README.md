# Deploiement Eureka Clinique sur une EC2 distincte

Cette configuration prepare un environnement de staging. En l'absence de domaine et de TLS, le proxy est publie uniquement sur `127.0.0.1:8090` de l'EC2. L'application doit etre ouverte depuis le poste d'administration via un tunnel SSH, et non exposee directement sur Internet.

## Sources recuperees depuis GitHub

Compose construit le backend depuis `Sokevi/eureka-clinique-backend` et le frontend depuis `Sokevi/eureka-clinique`, en utilisant la branche `master` et l'acces SSH. Il recupere le contenu Git distant a chaque demande manuelle de build ; aucun deploiement automatique n'est configure.

Chaque depot prive utilise sa propre deploy key GitHub en lecture seule. Ajouter les cles publiques de l'EC2 comme deploy keys dans les parametres de securite de leurs depots respectifs. Ne jamais copier une cle privee dans le depot ni dans le compose.

Les Dockerfiles et les changements applicatifs destines au deploiement doivent etre commits et pushes sur `master` dans leurs depots respectifs. Un changement local non pousse n'est pas visible par le build distant.

## Construire et mettre a jour

Pour chaque deploiement manuel, charger les deux deploy keys dans l'agent SSH et lancer le build avant de redemarrer les services :

```sh
eval "$(ssh-agent -s)"
ssh-add /home/ubuntu/.ssh/eureka_clinique_backend_deploy
ssh-add /home/ubuntu/.ssh/eureka_clinique_frontend_deploy
docker compose build --ssh "default=$SSH_AUTH_SOCK" --pull
docker compose up -d
docker compose ps
```

Le build telecharge la version courante de `master`, puis reconstruit les images. Un push Git ne lance pas le build ni le redemarrage ; ils ne se produisent que lorsque ces commandes sont executees manuellement.

## Secrets

Creer un fichier `.env` local a partir de `.env.example`, puis remplacer toutes les valeurs d'exemple par des secrets propres a l'environnement. Ne jamais versionner ni partager ce fichier. `JWT_SECRET` doit contenir au moins 64 octets aleatoires pour HS512. Le profil Spring `prod` refuse de demarrer si ce secret ou les identifiants PostgreSQL sont absents.

La base PostgreSQL n'est pas publiee sur l'hote. pgAdmin est disponible uniquement sur `127.0.0.1:8081`.

Depuis le poste d'administration, ouvrir le tunnel SSH (remplacer l'adresse et l'utilisateur) :

```sh
ssh -L 8090:127.0.0.1:8090 -L 8081:127.0.0.1:8081 ubuntu@ADRESSE_IP_EC2
```

Ouvrir ensuite `http://localhost:8090` pour l'application et, si necessaire, `http://localhost:8081` pour pgAdmin. Dans le groupe de securite AWS, limiter SSH aux adresses IP d'administration ; ne pas ouvrir les ports 8090/8081 au public.

## Avant une mise en production

Cette configuration n'active pas TLS. Avant tout acces public ou traitement de donnees medicales, configurer un nom de domaine, HTTPS avec certificats renouvelables, sauvegardes/restauration PostgreSQL et une politique de mises a jour. Verifier aussi la capacite memoire de l'instance : `t3.small` peut etre insuffisante pour l'API, PostgreSQL et pgAdmin simultanement.

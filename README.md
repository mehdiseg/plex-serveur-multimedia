# Plex Media Server : installation, organisation et bonnes pratiques

[![Compose](https://github.com/mehdiseg/plex-serveur-multimedia/actions/workflows/verifier.yml/badge.svg)](https://github.com/mehdiseg/plex-serveur-multimedia/actions/workflows/verifier.yml)

Guide et fichier Docker Compose pour héberger un **serveur multimédia Plex** chez soi : films, séries, musique, accessibles depuis la télévision, le téléphone ou un navigateur. Le guide insiste sur ce qui est souvent négligé : **l'organisation des fichiers, la sécurité de l'accès distant et les sauvegardes**.

> **Statut : guide générique.** Le fichier `compose.yaml` est validé par `docker compose config` (syntaxe) à chaque `push`. Il décrit une installation type sur Linux ; ce dépôt ne détaille pas une configuration personnelle.

## Installation avec Docker

Sur un serveur Linux :

```bash
cp .env.exemple .env     # adapter PUID/PGID (commande : id), les dossiers, et coller le jeton https://www.plex.tv/claim
docker compose up -d
curl -s http://localhost:32400/identity   # répond en XML quand le serveur est prêt
```

Puis ouvrir `http://<adresse-du-serveur>:32400/web`, se connecter avec son compte Plex et ajouter les bibliothèques (dossiers `/movies`, `/tv`, `/music` vus depuis le conteneur).

Points de conception du `compose.yaml` :

| Choix | Raison |
|---|---|
| `network_mode: host` | la découverte des lecteurs du réseau local ne traverse pas le réseau Docker « bridge » (sous Windows ou macOS avec Docker Desktop, ce mode n'est pas équivalent : préférer une installation native) |
| médias montés en **lecture seule** (`:ro`) | Plex n'a pas besoin de modifier ni de pouvoir supprimer les fichiers |
| `PUID` / `PGID` | le conteneur lit les fichiers avec les droits d'un utilisateur normal, pas root |
| configuration dans un dossier à part (`/config`) | la base de données et les préférences survivent à une mise à jour de l'image |
| `.env` ignoré par git | le jeton de rattachement n'est jamais publié |

Installation native (Windows, NAS, Debian) : le principe est le même, avec le programme d'installation depuis [plex.tv/media-server-downloads](https://www.plex.tv/media-server-downloads/). Sous Windows, la configuration se trouve dans `%LOCALAPPDATA%\Plex Media Server`.

## Organiser ses fichiers

Plex reconnaît les médias grâce aux **noms de dossiers et de fichiers**. Un bon nommage évite les erreurs d'affiches et de résumés.

```text
films/
  Nom du film (2010)/
    Nom du film (2010).mkv
séries/
  Nom de la série (2015)/
    Season 01/
      Nom de la série (2015) - s01e01.mkv
      Nom de la série (2015) - s01e02.mkv
    Season 02/
musique/
  Artiste/
    Album (2019)/
      01 - Titre.flac
```

- L'**année entre parenthèses** lève les ambiguïtés entre deux films du même nom.
- Un dossier par film ou par série, jamais tous les fichiers à la racine.
- Les saisons s'appellent `Season 01` (avec le zéro), les épisodes `s01e01`.

Le détail des règles est dans la [documentation officielle](https://support.plex.tv/articles/naming-and-organizing-your-movie-media-files/).

## Sécurité

- **Ne pas ouvrir le port 32400 de la box « au hasard »** : Plex propose son propre accès distant (Paramètres → Accès à distance) ; si on ne s'en sert pas, le désactiver.
- Pour regarder ses médias à l'extérieur sans rien exposer, un **VPN maillé** comme [Tailscale](https://tailscale.com/) évite toute redirection de port : le serveur et le téléphone se voient comme sur le même réseau.
- Activer la **double authentification** du compte Plex.
- Dans Paramètres → Réseau, laisser vide la liste des « réseaux autorisés sans authentification » : tout ce qui y figure accède au serveur sans mot de passe.
- Restreindre le port au réseau local avec le pare-feu : `sudo ufw allow from 192.168.10.0/24 to any port 32400 proto tcp` (adapter le réseau).
- **Ne jamais publier** le dossier de configuration (`plex-config/`) : il contient des jetons d'accès (`Preferences.xml`). Le `.gitignore` de ce dépôt l'exclut.

## Sauvegardes

- Les **médias** se sauvegardent comme n'importe quels fichiers (règle 3-2-1 : trois copies, deux supports, une hors site).
- La **base Plex** (historique de lecture, bibliothèques, métadonnées) se trouve dans le dossier `/config`, donc `plex-config/` ici. Arrêter le conteneur (`docker compose stop`) avant de le copier, puis le relancer.
- Plex effectue aussi des copies automatiques de sa base (Paramètres → Tâches programmées) : elles protègent d'une corruption, pas d'un disque perdu.

## Transcodage

Le **transcodage** convertit un fichier vers un format que le lecteur sait lire. Il consomme beaucoup de processeur, sauf avec l'accélération matérielle (fonction réservée aux abonnés Plex Pass). Le fichier `compose.yaml` contient le passage de `/dev/dri` (processeur graphique Intel), en commentaire, à activer si la machine le permet. Préférer des lecteurs qui lisent directement le format (« lecture directe ») évite ce coût.

## Vérifier

```bash
docker compose ps                       # état du conteneur
docker compose logs -f plex             # journaux en temps réel
curl -s http://localhost:32400/identity # identité et version du serveur
```

## Pour aller plus loin

- Ajouter un suivi de l'activité (Tautulli) et l'alerte de disponibilité avec Uptime Kuma : voir [homelab-docker-services](https://github.com/mehdiseg/homelab-docker-services).
- Automatiser la sauvegarde de `plex-config/` avec un script planifié.
- Mettre le serveur derrière un onduleur et surveiller l'état des disques (SMART).

## Licence

[MIT](LICENSE)

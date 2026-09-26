# Yapapouaiye Player sur Cloudflare Pages + R2

Cette version utilise Cloudflare Pages pour le site et les Functions, puis Cloudflare R2 pour stocker les fichiers audio, les covers et l'index des titres.

## Application Android (APK)

Le projet `android/` ouvre le lecteur hébergé dans une Trusted Web Activity. Chrome ou un navigateur Android compatible est nécessaire. La connexion Google, la session, la lecture et les notifications du site utilisent ainsi le même moteur que la PWA. Une connexion Internet est nécessaire pour charger le lecteur.

Sur Android 13 et plus, l'application demande l'autorisation système des notifications au premier lancement. Pour recevoir les alertes des nouveaux morceaux, se connecter, ouvrir la cloche et toucher **Activer les alertes sur cet appareil** afin d'inscrire le téléphone au push du site. Les sorties utilisent le canal Android **Sorties Yapapouaiye** avec une importance élevée pour afficher une bannière. Si le téléphone propose un réglage séparé pour les notifications flottantes, l'activer dans les réglages de ce canal. Les alertes sont envoyées lors des nouvelles sorties, pas à chaque ouverture de l'application.

Pour reconstruire l'APK sur Windows, installer Java 17 ou plus et le SDK Android (plateforme API 36), puis définir `ANDROID_HOME` ou créer `android/local.properties` contenant `sdk.dir=C:/chemin/vers/Android/Sdk`. Conserver la clé de signature de façon sûre : perdre cette clé empêche d'installer les futures versions par-dessus l'application existante.

Placer la clé PKCS12 dans `android/.signing/release.p12` et créer `android/.signing/release.properties` :

```properties
storeFile=.signing/release.p12
storePassword=<mot-de-passe>
keyAlias=yapapouaiye
keyPassword=<mot-de-passe>
```

Puis lancer `npm run android:build`. L'APK signé se trouve dans `android/app/build/outputs/apk/release/`. Le fichier `.well-known/assetlinks.json` doit contenir l'empreinte SHA-256 de cette même clé et être déployé sur Pages pour que l'application s'ouvre en plein écran et que les liens du site puissent ouvrir l'application. Sans cette association, Android ouvre le lecteur avec la barre du navigateur.

À partir de la version 1.0.10, l'APK cherche une nouvelle version dans `/android-updates/latest.json` au lancement. La version 1.1.13 tente de télécharger et d'installer automatiquement les nouvelles versions sur Android 12 ou plus récent, si l'autorisation d'installer des APK est accordée. Android peut encore demander une confirmation système. Si l'installation automatique échoue, le lancement suivant propose l'installation manuelle. L'APK natif vérifie l'empreinte SHA-256, le paquet et la signature avant installation. La transition depuis une version antérieure vers 1.1.13 se confirme manuellement.

Pour publier une nouvelle version Android, augmenter `versionCode` et `versionName` dans `android/app/build.gradle`, puis lancer `npm run android:release`. Cette commande construit l'APK signé, prépare `android-updates/latest.json` et l'APK versionné, puis les déploie sur Cloudflare Pages. Ne pas modifier `latest.json` à la main et conserver la même clé de signature pour les mises à jour.

Sur iPhone, `/iphone.html` explique l'ajout de la version web à l'écran d'accueil depuis Safari. Le lecteur utilise l'icône Apple et le mode app web ; il n'existe pas d'APK iOS.

## Installation

```bash
npm install
npx wrangler login
npx wrangler r2 bucket create yapapouaiye-music
npx wrangler pages project create yapapouaiye-player --production-branch main
```

Si le projet Pages existe deja, garde-le et passe directement a l'etape du secret.

Ajoute ensuite les secrets admin. Ils protegent l'upload et la suppression depuis `/admin-login.html`.

```bash
npx wrangler pages secret put ADMIN_USERNAME --project-name yapapouaiye-player
npx wrangler pages secret put ADMIN_PASSWORD --project-name yapapouaiye-player
npx wrangler pages secret put STREAM_GUARD_SECRET --project-name yapapouaiye-player
```

`STREAM_GUARD_SECRET` doit etre une valeur aleatoire longue, differente du mot de passe admin. Il signe les jetons de session et limite le comptage des streams. En local, le mot de passe admin de test sert uniquement de repli.

Tu peux aussi definir un secret dedie aux sessions :

```bash
npx wrangler pages secret put SESSION_SECRET --project-name yapapouaiye-player
```

`SESSION_SECRET` est utilise en priorite pour signer les jetons de session (comptes et admin). S'il n'est pas defini, `STREAM_GUARD_SECRET` sert de repli.

## Notifications des nouvelles musiques

Les morceaux publics de YapapouaiyeMusic apparaissent par défaut dans la cloche de tous les comptes connectés. Les sorties des autres artistes suivis y apparaissent aussi. Les notifications push nécessitent l'accord du navigateur sur chaque appareil.

Pour activer l'envoi push en production, conserve les clés VAPID générées dans `.wrangler/vapid-keys.json` (ne publie jamais la clé privée), puis configure Cloudflare et déploie le site ainsi que le Worker programmé :

```bash
node Script/generate-vapid-keys.cjs
npm run push:deploy
node Script/configure-push-secrets.cjs
npm run deploy
```

Les morceaux rendus publics déclenchent aussitôt l'envoi push aux comptes éligibles. Le Worker `yapapouaiye-push-dispatcher` vérifie aussi les sorties toutes les 5 minutes pour rattraper les échecs et les sorties programmées ; les deux chemins partagent le suivi des envois pour éviter les doublons. Il utilise le bucket R2 `yapapouaiye-music`. Dans le lecteur, chaque utilisateur doit cliquer sur **Activer les alertes sur cet appareil** et accepter la demande du navigateur. Sur iPhone, le site doit d'abord être ajouté à l'écran d'accueil (iOS 16.4 ou plus récent). Sur Windows, l'application Electron affiche les alertes système tant qu'elle tourne, y compris lorsqu'elle est réduite dans la zone de notification.

La page `/admin-login` affiche d'abord un ecran de connexion. Tu dois saisir l'identifiant admin et le mot de passe admin pour acceder a l'espace d'upload.

Depuis cette page admin, tu peux aussi creer des playlists privees:

- choisis un nom de playlist;
- choisis un mot de passe de playlist;
- selectionne les musiques a inclure;
- copie le lien genere, sous la forme `/music.html?playlist=<id>`.

Les visiteurs qui ouvrent ce lien doivent saisir le mot de passe de la playlist pour afficher les musiques. Le mot de passe est stocke dans R2 sous forme de hash SHA-256 sale, jamais en clair.

Depuis l'admin, tu peux aussi creer des pages artistes:

- choisis le nom affiche, le slug du lien et le nom associe aux musiques;
- ajoute une banniere pleine page, un avatar, une bio, une couleur et des liens cliquables;
- copie le lien genere, sous la forme `/artist?artist=<slug>`.

Pour les liens artistes, ajoute un lien par ligne dans l'admin:

```text
YouTube | https://youtube.com/...
Instagram | https://instagram.com/...
```

Les musiques affichees sur une page artiste sont les musiques publiques dont le champ Artiste correspond au nom associe. Les musiques rangees dans une playlist privee restent masquees sur les pages artistes publiques.

## Aperçus vidéo dans Discord

Les liens habituels `/music.html?track=<id>` gardent leur destination. Pour les morceaux publics, Discord peut lire dans le message une vidéo MP4 à pochette fixe avec la musique et sa barre de progression native. La vidéo est générée séparément ; les titres privés ou non encore sortis ne sont jamais proposés à Discord. Les contrôles affichés sont ceux de Discord, pas ceux du lecteur web.

Après avoir installé FFmpeg et déployé le site, génère les vidéos manquantes avec :

```bash
npm run share:videos
```

La commande se relance après chaque ajout ou remplacement d'audio ou de pochette. Elle ignore les vidéos déjà présentes. Pour vérifier sans téléverser : `npm run share:videos -- --dry-run`. Pour un seul titre : `npm run share:videos -- --track <id>`. Si FFmpeg n'est pas dans le `PATH`, indique son exécutable avec `FFMPEG_PATH`.

## Developpement local

L'environnement Dev est identique a la prod mais totalement isole :

- il tourne sur une autre adresse (`http://127.0.0.1:8788`) que le site deploye (`yapapouaiye-player.pages.dev`) ;
- il utilise son propre bucket R2 (`yapapouaiye-music-dev`), jamais le bucket de prod (`yapapouaiye-music`) ;
- les identifiants admin de test sont dedies (`admin` / `admin`), differents de ceux de la prod.

```bash
npm run dev
```

Le player est disponible sur `http://127.0.0.1:8788`. La page admin est sur `/admin-login.html`.
Identifiants admin de test : `admin` / `admin`.

### Copier la base de donnees de la prod vers Dev

Pour que Dev contienne une copie de la base de prod (titres, playlists, comptes, artistes et medias), lance :

```bash
npm run db:sync
```

La commande telecharge depuis le bucket R2 de prod (`yapapouaiye-music`) :

- la base de donnees : `_metadata/tracks.json`, `_metadata/playlists.json`, `_metadata/artists.json`, `_metadata/users.json`, `_metadata/pending_tracks.json` ;
- les medias references par cette base : audios, covers, bannieres et avatars d'artistes.

Puis elle ecrit le tout dans l'etat R2 local de Dev (`.wrangler/dev-state`). Rien n'est modifie en prod.

Variantes :

```bash
npm run db:sync:metadata   # copie seulement la base, sans les medias
node Script/r2-sync.cjs --dry-run   # simule la copie sans rien ecrire
```

> Le compte Cloudflare connecte doit avoir acces au bucket de prod. Verifie avec `npx wrangler whoami` :
> si le login actuel n'est pas le compte qui possede le projet `yapapouaiye-player`, reconnecte-toi avec
> `npx wrangler login` (ou definis `CLOUDFLARE_API_TOKEN` et `CLOUDFLARE_ACCOUNT_ID`).

### Site Dev heberge et prive

Le projet Pages `yapapouaiye-player-dev` utilise le meme bucket R2 que la production :
`MUSIC_BUCKET` pointe vers `yapapouaiye-music`. Il lit et ecrit donc les **vraies donnees**
(comptes, playlists, musiques et metadonnees). Le developpement local `npm run dev`
reste independant et utilise toujours son bucket local.

Cloudflare Access protege le site Dev avant que les requetes atteignent Pages. L'application
`Yapapouaiye Player Dev` couvre `yapapouaiye-player-dev.pages.dev` et
`*.yapapouaiye-player-dev.pages.dev`. Sa politique `Yapapouaiye Dev owner` autorise
uniquement `donovanclerambaux9@gmail.com`. L'adresse seule ne donne donc pas acces au site.

Pour recreer la configuration sur un autre compte Cloudflare :

1. Cree le projet Pages `yapapouaiye-player-dev` avec la branche de production `main`.
2. Configure le binding R2 `MUSIC_BUCKET` vers `yapapouaiye-music` dans les environnements
   Production et Preview du projet Dev.
3. Dans Cloudflare One > Access controls > Applications, cree une application auto-hebergee
   pour les deux domaines ci-dessus avec une politique Allow limitee a l'adresse e-mail du
   testeur. Fais-le **avant** le premier deploiement : sans Access, l'URL Pages est publique.
4. Configure les secrets necessaires aux fonctions utilisees en Dev :
   `SESSION_SECRET`, `ADMIN_USERNAME`, `ADMIN_PASSWORD`, `SCW_ACCESS_KEY`,
   `SCW_SECRET_KEY` et `SCW_BUCKET`. La region Scaleway est `fr-par` par defaut.
5. Pour la connexion Google, cree un client OAuth Web dans Google Cloud avec l'URI de
   redirection `https://yapapouaiye-player-dev.pages.dev/api/auth/google/callback`,
   puis ajoute son ID et son secret sous `GOOGLE_CLIENT_ID` et `GOOGLE_CLIENT_SECRET`
   dans les secrets du projet Pages Dev. L'application Google `Yapapouaiye Player Dev`
   du projet `YapapouaiyeHub` est en mode Test, avec
   `donovanclerambaux9@gmail.com` comme utilisateur test.
6. Deploie avec `npm run deploy:dev`, puis ouvre
   `https://yapapouaiye-player-dev.pages.dev` et connecte-toi avec l'adresse autorisee.

Les secrets sont propres a chaque projet Pages. Les secrets de production ne sont pas
lisibles depuis Cloudflare : pour tester Stripe en Dev, il faut configurer ses cles et
son webhook pour le domaine Dev. Sans cela, les fonctions de paiement restent limitees.

## Deploiement

```bash
npm run deploy:dev  # publie uniquement sur le site Dev prive
npm run deploy      # publie sur le site de production
```

Le binding R2 est declare dans `wrangler.toml` sous le nom `MUSIC_BUCKET`. Si tu changes le nom du bucket, mets aussi a jour `bucket_name`.

### Abonnements Stripe

Deux abonnements mensuels sont disponibles :

- Plus a 5 EUR pour l'acces anticipe, les playlists privees et les contenus supporters ;
- Max a 10 EUR pour tous les avantages Plus et le lecteur premium sans publicite.

Configure les secrets Stripe dans Cloudflare Pages :

```bash
npx wrangler pages secret put STRIPE_SECRET_KEY --project-name yapapouaiye-player
npx wrangler pages secret put STRIPE_WEBHOOK_SECRET --project-name yapapouaiye-player
npx wrangler pages secret put STRIPE_PRICE_ID --project-name yapapouaiye-player
npx wrangler pages secret put STRIPE_MAX_PRICE_ID --project-name yapapouaiye-player
```

`STRIPE_PRICE_ID` correspond au tarif Plus et `STRIPE_MAX_PRICE_ID` au tarif Max. Active aussi le changement de formule dans le portail client Stripe pour permettre le passage de Plus vers Max.

## Comptage des streams

Le compteur n'augmente plus au simple demarrage du lecteur. Un stream n'est comptabilise que si :

- le visiteur est connecte et le lecteur envoie le jeton de session signe par le serveur (`x-account-token`) ;
- le titre est ecoutable (publique, ou accessible pour le plan du compte) ;
- le compte n'a pas depasse le plafond quotidien (500 ecoutes comptabilisees par jour).

Les requetes anonymes ou avec un jeton invalide reçoivent `401`. Le compteur est aussi protege par un rate limiting par IP (60 requetes/minute).

L'ancien `POST /api/tracks/<id>` ne compte plus rien et renvoie `410` : seul `POST /api/tracks/<id>/stream` peut mettre a jour le compteur.

Les compteurs journaliers sont stockes sous `_metadata/stream-usage/v1/<userId>/<date>.json`. Configure une regle de cycle de vie R2 qui supprime ce prefixe apres trois jours afin de limiter la retention. Cloudflare permet de cibler une regle de cycle de vie par prefixe : https://developers.cloudflare.com/r2/buckets/object-lifecycles/

## Securite

- Les comptes utilisent des jetons de session signes HMAC (`x-account-token`), jamais l'id du compte seul : un id forge ne donne aucun acces.
- Les mots de passe sont haches en PBKDF2-SHA256 (100 000 iterations). Les anciens hashes SHA-256 sont migres automatiquement a la prochaine connexion.
- La page admin utilise un jeton dedie (`POST /api/admin/login` renvoie un jeton utilise en `x-admin-token`, valable 2 heures) au lieu d'envoyer le mot de passe a chaque requete.
- Les endpoints sensibles (login, register, admin, mots de passe de playlists, streams) sont limites par IP.
- `/media/` ne sert que les medias publics (`tracks/`, `artists/`, `pending-tracks/`) : tout le reste du bucket R2 (dont `_metadata/`) renvoie `404`.
- Les audios des pistes sont proteges par des URLs signees HMAC a duree limitee (24 h) : `/media/tracks/<id>/audio.*` exige `?exp=<timestamp>&sig=<hmac>`. Les URLs signees ne sont generees par l'API (`/api/tracks`, `/api/playlists/:id`, `/api/artists/:slug`) que si le compte authentifie (jeton signe `x-account-token`) a le plan requis (Plus/Max, acces anticipe 7j/14j, evenement). Sans entitlement, `audioUrl`/`audioKey` ne sont jamais renvoyes. Les covers restent publiques.
- Le checkout Stripe n'accepte que `plan: plus|max` (liste blanche) et le webhook verifie que le plan et le montant payes correspondent avant d'accorder l'abonnement (anti price confusion Plus <-> Max).
- En-tetes de securite (CSP, X-Content-Type-Options, X-Frame-Options, HSTS, Permissions-Policy) ajoutes aux pages statiques (`_headers`) et aux reponses des fonctions (`functions/_middleware.js`).

## Application desktop Electron

L'application desktop charge par defaut le site deploye:

```bash
npm run desktop
```

Pour tester l'application desktop avec le serveur local Wrangler, lance d'abord `npm run dev`, puis dans un autre terminal:

```bash
npm run desktop:local
```

Pour generer les installateurs Windows NSIS et MSI ainsi que la version portable:

```bash
npm run desktop:build
```

Le fichier `desktop-dist/Yapapouaiye Player <version>.msi` contient un assistant en français. La page de destination permet de choisir le dossier d'installation. La page suivante propose un raccourci sur le bureau (activé), le lancement au démarrage de Windows en arrière-plan (activé) et une tentative d'épinglage à la barre des tâches (désactivée). Windows peut bloquer l'épinglage automatique ; dans ce cas, épingle l'application depuis le menu Démarrer. Le démarrage en arrière-plan garde l'application dans la zone de notification sans ouvrir sa fenêtre.

### Mises a jour automatiques

Les installations NSIS et MSI (à partir de la version 1.0.4) recherchent automatiquement la dernière [Release GitHub](https://github.com/YapapouaiyeDev/YapapouaiyePlayer-Software/releases/latest) publique au démarrage. Le téléchargement se fait en arrière-plan, puis l'application propose **Installer maintenant** ou **Plus tard**. La version MSI vérifie la taille et l'empreinte SHA-256 du fichier avant de fermer l'application et de lancer Windows Installer. Windows peut demander une autorisation ; une installation automatique et silencieuse sans accord du système n'est pas possible. La version portable se met à jour manuellement.

Le MSI 1.0.3 installé auparavant désactive lui-même les mises à jour : il faut installer **une fois** le MSI 1.0.4 depuis GitHub Releases. Les MSI suivants pourront se mettre à jour depuis l'application. Les installations NSIS 1.0.3 continuent de lire l'ancien canal Cloudflare pour recevoir 1.0.4, puis basculent sur GitHub Releases.

Pour publier une nouvelle version desktop:

1. Initialise le dépôt GitHub `YapapouaiyeDev/YapapouaiyePlayer-Software` avec un README si sa branche principale est encore vide. Crée un jeton GitHub autorisé à écrire dans ce dépôt (permission **Contents: Read and write**) et définis-le localement dans `GITHUB_TOKEN`. Ne l'ajoute jamais aux fichiers du projet. Incrémente la version avec `npm version <version> --no-git-tag-version`.
2. Genere les fichiers desktop:

```bash
npm run desktop:build
```

3. Publie `latest.yml`, le programme NSIS, son blockmap, le MSI et `latest-msi.json` dans une Release GitHub publique :

```bash
npm run desktop:publish-github
```

Le script prépare d'abord une Release brouillon et la rend publique uniquement lorsque tous les fichiers sont envoyés. Si un envoi échoue, relance la commande avec la même version ; elle reprend le brouillon. Une version déjà publiée doit rester immuable.
Si la publication GitHub réussit mais que l'étape Cloudflare échoue, reprends uniquement avec `npm run desktop:stage-updates` puis `npm run deploy`, sans reconstruire ni republier la Release.

4. Maintiens temporairement l'ancien canal Cloudflare pour les installations NSIS antérieures à 1.0.4 :

```bash
npm run desktop:stage-updates
```

5. Deploie le site et les fichiers de mise a jour:

```bash
npm run deploy
```

Tu peux aussi faire les quatre dernières étapes en une commande, après avoir défini `GITHUB_TOKEN` :

```bash
npm run desktop:release
```

Sur Windows, tu peux utiliser le script interactif:

```bash
Script\release-desktop.bat
```

Il demande la nouvelle version, met a jour `package.json`, construit l'app desktop, publie la Release GitHub et maintient le canal Cloudflare des anciennes versions.

## Bases de stockage des fichiers

Les fichiers medias (audio, covers) peuvent etre stockes sur deux bases, choisies a la publication :

| Choix | Nom | Backend | Usage |
|:---|:---|:---|:---|
| Auto | (defaut) | R2 puis Scaleway en secours | Base 1 prioritaire, bascule automatique sur la Base 2 si la Base 1 est indisponible ou en echec. |
| Base 1 | Cloudflare | Cloudflare R2 (`MUSIC_BUCKET`) | Stockage principal historique. |
| Base 2 | Scaleway | Scaleway Object Storage (S3) | Stockage secondaire / secours. |

- Le choix est disponible dans le formulaire admin d'upload (`Auto` / `Base 1` / `Base 2`) et dans le formulaire d'edition d'un morceau.
- Les publications faites **en tant qu'artiste** (`espace-artiste.html`) sont **toujours en `Auto`** : le serveur ignore tout `storageTarget` envoye par le client artiste.
- Les **metadonnees** (`_metadata/*.json`) restent toujours dans la **Base 1 (R2)** : c'est la source de verite unique du catalogue. Seuls les binaires sont routes vers la base choisie.
- Les fichiers stockes en Base 2 portent le prefixe `scw/` dans les metadonnees (ex. `scw/tracks/<id>/audio.mp3`) : le handler `/media/` deduit la base depuis ce prefixe, sans consultation d'index. Les URLs signees et le support HTTP `Range` fonctionnent a l'identique sur les deux bases.

### Configuration de la Base 2 (Scaleway)

La Base 2 est optionnelle. Elle est active des que ces variables sont definies :

| Variable | Description | Defaut |
|:---|:---|:---|
| `SCW_ACCESS_KEY` | Cle d'acces Scaleway | - |
| `SCW_SECRET_KEY` | Cle secrete Scaleway | - |
| `SCW_BUCKET` | Nom du bucket Object Storage | - |
| `SCW_REGION` | Region Scaleway | `fr-par` |
| `SCW_ENDPOINT` | Endpoint S3 personnalise | `https://<bucket>.s3.<region>.scw.cloud` |

Definir les secrets en production :

```bash
wrangler pages secret put SCW_ACCESS_KEY
wrangler pages secret put SCW_SECRET_KEY
wrangler pages secret put SCW_BUCKET
```

En developpement local, ajoute ces variables dans `.dev.vars`.

> Sans ces 3 variables, tout upload en **Base 2 echoue** : l'API renvoie une
> erreur JSON `501 Base 2 (Scaleway) non configuree` (le formulaire admin
> l'affiche au lieu de planter).

## Likes et commentaires

Les interactions sont stockées dans Scaleway Object Storage, dans des objets
JSON sous le préfixe `social/`. L'API utilise les identifiants `SCW_ACCESS_KEY`,
`SCW_SECRET_KEY`, `SCW_BUCKET` et `SCW_REGION` déjà utilisés par le stockage
Scaleway. Aucun SQL ni binding Hyperdrive n'est nécessaire. Les likes ont une
clé unique par morceau et compte; les commentaires sont enregistrés dans des
objets distincts.

## Structure R2

- `tracks/<id>/audio.*`: fichier musique.
- `tracks/<id>/cover.*`: cover optionnelle.
- `artists/<id>/hero-*`: banniere de page artiste.
- `artists/<id>/avatar-*`: avatar de page artiste.
- `_metadata/tracks.json`: liste des titres, artistes et URLs publiques servies par `/media/...`.
- `_metadata/playlists.json`: liste des playlists privees, leurs titres, leurs morceaux et les hashes de mots de passe.
- `_metadata/artists.json`: fiches artistes, slugs, textes, couleurs et images.
- `_metadata/stream-token-uses/v1/`: sessions de stream deja validees, a expirer automatiquement.

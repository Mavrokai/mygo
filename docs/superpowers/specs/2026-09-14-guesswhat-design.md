# Conception de GuessWhat

Date : 14 septembre 2026
Statut : validée

## Objectif

Construire le jeu GuessWhat décrit dans `guesswhat.pdf` : un équivalent simplifié de Pictionary où plusieurs navigateurs partagent une partie en temps réel. Un joueur dessine un mot secret et le premier autre joueur qui le trouve gagne des points selon sa rapidité.

Le dépôt part d'une base vide. Le livrable utilise Go, HTML5, CSS et JavaScript natif. Il fonctionne avec un seul binaire Go qui sert l'interface web et les connexions WebSocket.

## Périmètre

Le périmètre obligatoire comprend :

- création d'une room et obtention d'un code ;
- liste des rooms publiques en attente ;
- entrée dans une room par son code ;
- administrateur de room et démarrage réservé à celui-ci ;
- liste en temps réel des joueurs connectés ;
- chat avec historique ;
- dessin Canvas synchronisé en temps réel ;
- mot aléatoire envoyé au seul dessinateur ;
- une lettre du mot, choisie aléatoirement, révélée à tous dès le début ;
- détection du premier joueur qui trouve le mot ;
- score proportionnel au temps restant ;
- rotation aléatoire des dessinateurs ;
- classement final et fermeture de la room.

Les comptes, la base de données, l'historique entre redémarrages, les équipes, les rooms privées et l'administration des mots sont exclus.

## Technologies

- Go 1.27.1, dernière version stable vérifiée au 14 septembre 2026 ;
- bibliothèque maintenue `github.com/coder/websocket` pour les WebSockets ;
- serveur standard `net/http`, journalisation `log/slog` et fichiers embarqués avec `embed` ;
- HTML sémantique, CSS Grid et JavaScript natif en modules ES ;
- APIs navigateur `WebSocket`, `CanvasRenderingContext2D` et Pointer Events ;
- aucune dépendance JavaScript d'exécution, aucun bundler et aucun serveur front séparé.

Le poste actuel possède Go 1.22.2. Le module déclarera la toolchain Go 1.27.1 afin que Go puisse récupérer automatiquement la version requise.

## Architecture

### Processus unique

Le programme démarre un serveur HTTP unique. Il sert les ressources embarquées, expose l'API des rooms et accepte les connexions WebSocket. Les parties résident uniquement en mémoire.

### Registre des rooms

Un registre concurrent conserve les rooms actives par code. Il est responsable de leur création, de leur recherche, de leur publication dans la liste publique et de leur suppression finale. Il ne modifie pas directement leur état métier.

### Boucle événementielle par room

Chaque room possède une goroutine qui est seule autorisée à modifier ses joueurs, son chat, son dessin et l'état de sa partie. Les connexions lui transmettent des événements par channels. Ce modèle évite les accès concurrents complexes et isole les rooms.

Chaque client possède une file d'envoi bornée. Une connexion lente est fermée plutôt que de bloquer la room entière. L'annulation par contexte arrête les goroutines et timers associés.

### Organisation prévue

- `cmd/guesswhat` : chargement de la configuration, signaux, démarrage et code de sortie ;
- `internal/game` : registre, room, joueur, manche, mots, score et transitions ;
- `internal/realtime` : client WebSocket, décodage et encodage du protocole ;
- `internal/server` : handlers HTTP, validation et composition des services ;
- `web` : ressources embarquées, pages, styles, modules JavaScript et Canvas ;
- `data` : liste de mots française embarquée.

## Cycle de jeu

Une room accepte de 2 à 12 joueurs. Les pseudos sont uniques dans la room. L'administrateur est le créateur ; s'il quitte le lobby, le rôle passe au joueur présent le plus ancien.

Au démarrage, le serveur fige les joueurs actifs et mélange leur ordre de dessin. Chaque joueur obtient exactement un tour. Un joueur arrivé après le démarrage voit l'état et le dessin courant comme spectateur, mais n'entre pas dans l'ordre de cette partie.

Pour chaque manche :

1. le serveur choisit un mot et un dessinateur ;
2. le Canvas et son historique sont remis à zéro ;
3. le mot complet est envoyé au seul dessinateur ;
4. un emplacement alphabétique non vide est choisi aléatoirement et cette occurrence est révélée dans le masque envoyé aux autres ;
5. la manche dure 90 secondes ;
6. le dessinateur envoie ses opérations de dessin et les autres joueurs proposent des réponses ;
7. la première bonne réponse termine la manche et rapporte `100 + secondes restantes` au gagnant ;
8. à défaut de gagnant, le timer termine la manche sans attribuer de point ;
9. le serveur révèle le mot et annonce le résultat avant de passer au dessinateur suivant.

Après le dernier tour, le serveur envoie le classement final, ferme les connexions proprement et supprime la room. Si le dessinateur part, la manche se termine sans gagnant. S'il reste moins de deux joueurs actifs, la partie est annulée et la room est fermée.

Les réponses sont comparées après suppression des espaces superflus, passage en minuscules et normalisation des accents. Une proposition incorrecte apparaît dans le chat. Une bonne réponse n'y révèle jamais le mot.

## API HTTP

- `POST /api/rooms` avec un pseudo crée une room et retourne son code, un jeton de session temporaire et le rôle administrateur ;
- `POST /api/rooms/{code}/join` valide le pseudo et retourne un jeton de session temporaire ;
- `GET /api/rooms` retourne les rooms publiques qui attendent leur démarrage ;
- `GET /room/{code}` sert l'application sur l'URL partageable ;
- `GET /ws?room={code}&token={token}` transforme la connexion en WebSocket ;
- `GET /healthz` indique si le processus est prêt.

Les codes de room et jetons sont générés avec `crypto/rand`. Les jetons restent en mémoire et ne sont jamais écrits dans les journaux.

## Protocole WebSocket

Chaque paquet est un objet JSON :

```json
{
  "type": "chat_message",
  "payload": {}
}
```

Le serveur lit d'abord `type`, puis décode le paquet dans la structure Go dédiée. Les champs inconnus ou mal formés sont rejetés par une erreur protocolaire explicite.

Messages du client :

- `start_game` ;
- `chat_message` ;
- `draw_stroke` ;
- `draw_shape` ;
- `erase_stroke` ;
- `clear_canvas` ;
- `ping`.

Messages du serveur :

- `room_state` ;
- `player_joined` et `player_left` ;
- `game_started` ;
- `round_started` ;
- `chat_message` ;
- `draw_stroke`, `draw_shape`, `erase_stroke` et `canvas_cleared` ;
- `round_ended` ;
- `game_ended` ;
- `error` ;
- `pong`.

Le serveur est l'unique source de vérité. Il vérifie l'état de la room, le rôle du joueur, les coordonnées, les couleurs, les tailles et les transitions avant toute diffusion.

## Dessin

Le Canvas est l'élément principal de l'interface desktop. Les coordonnées sont normalisées entre 0 et 1 pour produire le même dessin quelle que soit la résolution.

Outils :

- crayon libre ;
- gomme libre à taille réglable ;
- trait ;
- rectangle ;
- cercle ;
- triangle ;
- épaisseur du contour ;
- remplissage activable pour les formes ;
- couleurs de contour et de remplissage indépendantes ;
- suppression totale avec confirmation.

Le sélecteur de couleur comprend un carré saturation/luminosité, une barre de teinte, un aperçu, une valeur hexadécimale et des champs rouge, vert et bleu de 0 à 255.

Les mouvements libres sont regroupés en petits lots de points pour limiter la fréquence réseau. Le serveur conserve un historique borné des opérations. Un client rejoignant une partie en cours reçoit cet historique et reconstruit le dessin. La gomme est une opération rejouable utilisant le mode de composition d'effacement du Canvas.

## Interface desktop

L'écran de jeu comporte :

- une barre supérieure avec logo, manche, mot masqué et timer ;
- une colonne gauche fixe avec code de room, joueurs, rôles et scores ;
- un grand Canvas central qui absorbe toute largeur disponible ;
- une barre d'outils compacte sous le Canvas avec icônes SVG ;
- une colonne de chat secondaire à droite.

Les autres écrans sont l'accueil, le lobby et le classement final. L'accueil permet de saisir son pseudo, créer une room, entrer un code et parcourir les rooms publiques.

La cible principale est un navigateur desktop récent à partir de 1280 px. L'interface utilise une structure sémantique, des libellés lisibles, la navigation clavier, des contrastes suffisants et des annonces ARIA pour le timer et les résultats.

## Erreurs et limites

Les erreurs fatales de configuration, chargement des mots ou écoute réseau sont écrites sur `stderr`, puis le processus termine avec le code `84`. Un arrêt normal, y compris après un signal traité proprement, termine avec `0`.

Les handlers HTTP renvoient un statut adapté et une erreur JSON stable. Les erreurs WebSocket sont envoyées sous la forme `error` avec un code machine et un message lisible. Une erreur client ne doit jamais arrêter une room ou le serveur.

Le serveur impose :

- longueur maximale des pseudos et messages ;
- limite de lecture WebSocket ;
- maximum de 12 joueurs ;
- historique de chat et de dessin borné ;
- validation stricte des couleurs RGB, épaisseurs et coordonnées ;
- fréquence maximale des opérations de dessin ;
- délais de lecture et d'écriture ;
- contrôle same-origin lors de l'ouverture WebSocket.

## Tests et validation

Les tests Go couvrent :

- création et suppression des rooms ;
- contraintes sur les pseudos et joueurs ;
- démarrage réservé à l'administrateur ;
- transitions de manche et ordre des dessinateurs ;
- masque du mot et lettre aléatoire ;
- normalisation des réponses et calcul du score ;
- déconnexions de l'administrateur et du dessinateur ;
- décodage des paquets valides et invalides ;
- synchronisation HTTP/WebSocket avec `httptest` ;
- nettoyage des goroutines et timers.

Les modules JavaScript purs sont testés avec le runner intégré de Node, sans dépendance d'exécution. Les contrôles finaux sont :

```text
go test ./...
go test -race ./...
go vet ./...
node --test
```

Un test manuel multiclient vérifie enfin la création, l'entrée, le dessin simultané, le chat, la victoire, le changement de dessinateur et la fermeture finale.

## Critères d'acceptation

Le projet est accepté lorsque :

- il se compile en un seul binaire et sert seul toute l'application ;
- deux à douze navigateurs peuvent jouer dans une même room ;
- plusieurs rooms fonctionnent sans partager leur état ;
- le mot secret n'est jamais envoyé aux joueurs qui devinent ;
- le dessin, les formes, la gomme, les couleurs RGB et le chat sont synchronisés ;
- une lettre aléatoire est visible dès le début de chaque manche ;
- le gagnant, le score, la rotation et le classement final sont corrects ;
- les erreurs fatales utilisent `stderr` et le code `84` ;
- les tests Go, le détecteur de courses, `go vet` et les tests JavaScript passent.

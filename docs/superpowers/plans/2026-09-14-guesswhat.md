# GuessWhat Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Livrer un jeu GuessWhat multijoueur complet, servi par un binaire Go unique et synchronisé en WebSocket.

**Architecture:** Un registre conserve les rooms actives et chaque room possède une goroutine propriétaire de son état. Le serveur `net/http` expose l'API, sert les ressources embarquées et relie les clients WebSocket aux rooms ; le front JavaScript natif rend les états et produit des opérations Canvas normalisées.

**Tech Stack:** Go 1.27.1, `net/http`, `github.com/coder/websocket`, `golang.org/x/text`, HTML5, CSS Grid, modules ES, Canvas 2D, Pointer Events et runner de tests Node intégré.

---

## Arborescence cible

```text
.
├── .gitignore
├── go.mod
├── go.sum
├── package.json
├── cmd/guesswhat/
│   ├── main.go
│   └── main_test.go
├── data/
│   ├── embed.go
│   └── words_fr.txt
├── internal/
│   ├── config/
│   │   ├── config.go
│   │   └── config_test.go
│   ├── game/
│   │   ├── game.go
│   │   ├── game_test.go
│   │   ├── registry.go
│   │   ├── registry_test.go
│   │   ├── room.go
│   │   ├── room_test.go
│   │   ├── types.go
│   │   ├── words.go
│   │   └── words_test.go
│   ├── realtime/
│   │   ├── client.go
│   │   ├── protocol.go
│   │   └── protocol_test.go
│   └── server/
│       ├── handlers.go
│       ├── handlers_test.go
│       ├── server.go
│       ├── websocket.go
│       └── websocket_test.go
└── web/
    ├── embed.go
    ├── index.html
    └── assets/
        ├── css/app.css
        └── js/
            ├── api.js
            ├── app.js
            ├── canvas-engine.js
            ├── canvas-model.js
            ├── canvas-model.test.js
            ├── color.js
            ├── color.test.js
            ├── socket.js
            ├── store.js
            ├── store.test.js
            └── views.js
```

### Task 1: Initialiser le module et la configuration

**Files:**
- Create: `.gitignore`
- Create: `go.mod`
- Create: `internal/config/config.go`
- Test: `internal/config/config_test.go`

- [ ] **Step 1: Écrire les tests de configuration**

```go
func TestLoadDefaults(t *testing.T) {
    t.Setenv("GUESSWHAT_ADDR", "")
    cfg, err := Load()
    if err != nil {
        t.Fatal(err)
    }
    if cfg.Address != ":8080" || cfg.RoundDuration != 90*time.Second {
        t.Fatalf("unexpected defaults: %#v", cfg)
    }
}

func TestLoadRejectsInvalidAddress(t *testing.T) {
    t.Setenv("GUESSWHAT_ADDR", "not-an-address")
    if _, err := Load(); err == nil {
        t.Fatal("expected invalid address error")
    }
}
```

- [ ] **Step 2: Initialiser le module et vérifier l'échec attendu**

Run:

```bash
go mod init github.com/EpitechWebAcademiePromo2026/W-WEB-404-LIL-4-1-mygo-3
go mod edit -go=1.27.0 -toolchain=go1.27.1
go test ./internal/config
```

Expected: FAIL parce que `Load` et `Config` n'existent pas.

- [ ] **Step 3: Implémenter la configuration**

```go
type Config struct {
    Address       string
    RoundDuration time.Duration
    ResultDelay   time.Duration
    MaxPlayers    int
}

func Load() (Config, error) {
    address := strings.TrimSpace(os.Getenv("GUESSWHAT_ADDR"))
    if address == "" {
        address = ":8080"
    }
    if _, err := net.ResolveTCPAddr("tcp", address); err != nil {
        return Config{}, fmt.Errorf("invalid GUESSWHAT_ADDR: %w", err)
    }
    return Config{
        Address:       address,
        RoundDuration: 90 * time.Second,
        ResultDelay:   4 * time.Second,
        MaxPlayers:    12,
    }, nil
}
```

Créer `.gitignore` avec :

```gitignore
/guesswhat
/coverage.out
/.superpowers/
/guesswhat.pdf
```

- [ ] **Step 4: Vérifier et committer**

Run: `go test ./internal/config && gofmt -w internal/config`

Expected: PASS.

```bash
git add .gitignore go.mod internal/config
git commit -m "chore: initialize Go project configuration"
```

### Task 2: Embarquer les mots et normaliser les réponses

**Files:**
- Create: `data/embed.go`
- Create: `data/words_fr.txt`
- Create: `internal/game/words.go`
- Test: `internal/game/words_test.go`
- Modify: `go.mod`
- Create: `go.sum`

- [ ] **Step 1: Écrire les tests des mots**

```go
func TestNormalizeGuess(t *testing.T) {
    if got := NormalizeGuess("  Éléphant   BLEU "); got != "elephant bleu" {
        t.Fatalf("got %q", got)
    }
}

func TestMaskWordRevealsOnePosition(t *testing.T) {
    got, err := MaskWord("maison", 2)
    if err != nil {
        t.Fatal(err)
    }
    if got != "_ _ i _ _ _" {
        t.Fatalf("got %q", got)
    }
}

func TestNewWordBankRejectsEmptyInput(t *testing.T) {
    if _, err := NewWordBank(strings.NewReader("\n")); err == nil {
        t.Fatal("expected empty bank error")
    }
}
```

- [ ] **Step 2: Installer la dépendance Unicode et constater l'échec**

Run:

```bash
go get golang.org/x/text@latest
go test ./internal/game -run 'TestNormalizeGuess|TestMaskWord|TestNewWordBank'
```

Expected: FAIL parce que les fonctions n'existent pas.

- [ ] **Step 3: Implémenter le domaine des mots**

```go
type WordBank struct {
    words []string
}

func NewWordBank(r io.Reader) (*WordBank, error) {
    scanner := bufio.NewScanner(r)
    words := make([]string, 0, 128)
    for scanner.Scan() {
        word := strings.TrimSpace(scanner.Text())
        if word != "" {
            words = append(words, word)
        }
    }
    if err := scanner.Err(); err != nil {
        return nil, fmt.Errorf("read words: %w", err)
    }
    if len(words) == 0 {
        return nil, errors.New("word bank is empty")
    }
    return &WordBank{words: words}, nil
}

func NormalizeGuess(value string) string {
    decomposed := norm.NFD.String(strings.ToLower(strings.TrimSpace(value)))
    cleaned := strings.Map(func(r rune) rune {
        if unicode.Is(unicode.Mn, r) {
            return -1
        }
        return r
    }, decomposed)
    return strings.Join(strings.Fields(cleaned), " ")
}

func MaskWord(word string, reveal int) (string, error) {
    runes := []rune(word)
    if reveal < 0 || reveal >= len(runes) || !unicode.IsLetter(runes[reveal]) {
        return "", errors.New("invalid reveal position")
    }
    masked := make([]string, len(runes))
    for i, r := range runes {
        switch {
        case i == reveal:
            masked[i] = string(r)
        case unicode.IsSpace(r):
            masked[i] = " "
        default:
            masked[i] = "_"
        }
    }
    return strings.Join(masked, " "), nil
}
```

Dans `data/embed.go`, exposer les mots embarqués :

```go
package data

import (
    "embed"
    "io/fs"
)

//go:embed words_fr.txt
var files embed.FS

func OpenWords() (fs.File, error) {
    return files.Open("words_fr.txt")
}
```

Ajouter dans `words_fr.txt` au moins 100 mots français simples, sans doublon après normalisation, un par ligne.

- [ ] **Step 4: Vérifier et committer**

Run: `go test ./internal/game && gofmt -w data internal/game`

Expected: PASS.

```bash
git add data internal/game/words.go internal/game/words_test.go go.mod go.sum
git commit -m "feat: add embedded French word bank"
```

### Task 3: Implémenter la machine d'état pure du jeu

**Files:**
- Create: `internal/game/types.go`
- Create: `internal/game/game.go`
- Test: `internal/game/game_test.go`

- [ ] **Step 1: Écrire les tests des règles**

```go
func TestGameRequiresAdminAndTwoPlayers(t *testing.T) {
    g := newTestGame("admin", "guest")
    if err := g.Start("guest"); !errors.Is(err, ErrForbidden) {
        t.Fatalf("expected forbidden, got %v", err)
    }
    if err := g.Start("admin"); err != nil {
        t.Fatal(err)
    }
    if g.State() != StatePlaying || g.CurrentDrawerID() == "" {
        t.Fatalf("unexpected state: %#v", g.SnapshotFor("guest"))
    }
}

func TestCorrectGuessAwardsTimeBasedScore(t *testing.T) {
    g := startedTestGame(t, "maison")
    winner := firstGuesser(g)
    result, correct, err := g.SubmitGuess(winner, " MAÏSON ", 37*time.Second)
    if err != nil || !correct {
        t.Fatalf("guess failed: %v", err)
    }
    if result.Points != 137 {
        t.Fatalf("got %d points", result.Points)
    }
}

func TestDrawerCannotGuess(t *testing.T) {
    g := startedTestGame(t, "maison")
    if _, _, err := g.SubmitGuess(g.CurrentDrawerID(), "maison", time.Minute); !errors.Is(err, ErrForbidden) {
        t.Fatalf("expected forbidden, got %v", err)
    }
}
```

- [ ] **Step 2: Lancer les tests rouges**

Run: `go test ./internal/game -run 'TestGame|TestCorrect|TestDrawer'`

Expected: FAIL parce que `Game` n'existe pas.

- [ ] **Step 3: Définir les types et transitions**

```go
type State string

const (
    StateLobby       State = "lobby"
    StatePlaying     State = "playing"
    StateRoundResult State = "round_result"
    StateFinished    State = "finished"
)

var (
    ErrForbidden       = errors.New("action forbidden")
    ErrInvalidState    = errors.New("invalid game state")
    ErrNotEnoughPlayer = errors.New("at least two players are required")
)

type Player struct {
    ID        string `json:"id"`
    Nickname  string `json:"nickname"`
    Score     int    `json:"score"`
    Admin     bool   `json:"admin"`
    Connected bool   `json:"connected"`
    Spectator bool   `json:"spectator"`
}

type RoundResult struct {
    Word       string `json:"word"`
    WinnerID   string `json:"winnerId,omitempty"`
    WinnerName string `json:"winnerName,omitempty"`
    Points     int    `json:"points"`
}
```

Implémenter `Game.AddPlayer`, `RemovePlayer`, `Start`, `BeginRound`, `SubmitGuess`, `EndRound`, `AdvanceRound`, `SnapshotFor` et `FinalRanking`. Injecter `*rand.Rand` pour rendre l'ordre et la lettre révélée déterministes en test. `SnapshotFor` ne doit inclure `Word` que pour le dessinateur.

- [ ] **Step 4: Vérifier toutes les transitions**

Run: `go test ./internal/game`

Expected: PASS avec tests couvrant ordre aléatoire, lettre masquée, transfert admin, spectateur et classement.

- [ ] **Step 5: Commit**

```bash
git add internal/game/types.go internal/game/game.go internal/game/game_test.go
git commit -m "feat: implement GuessWhat game rules"
```

### Task 4: Construire l'acteur de room et le registre

**Files:**
- Create: `internal/game/room.go`
- Create: `internal/game/registry.go`
- Test: `internal/game/room_test.go`
- Test: `internal/game/registry_test.go`

- [ ] **Step 1: Tester la sérialisation des événements**

```go
func TestRoomBroadcastsJoinAndState(t *testing.T) {
    room := newTestRoom(t)
    session, err := room.Join(t.Context(), "Lina")
    if err != nil {
        t.Fatal(err)
    }
    sink := newRecordingSink(session.PlayerID)
    playerID, err := room.Attach(t.Context(), session.Token, sink)
    if err != nil {
        t.Fatal(err)
    }
    if playerID != session.PlayerID {
        t.Fatalf("got player %q, want %q", playerID, session.PlayerID)
    }
    event := sink.WaitFor(t, EventRoomState)
    if len(event.Payload.(RoomSnapshot).Players) != 1 {
        t.Fatalf("unexpected payload: %#v", event.Payload)
    }
}

func TestRegistryRemovesClosedRoom(t *testing.T) {
    registry := newTestRegistry(t)
    created, err := registry.Create(t.Context(), "Admin")
    if err != nil {
        t.Fatal(err)
    }
    registry.Close(created.Code)
    if _, ok := registry.Get(created.Code); ok {
        t.Fatal("closed room still registered")
    }
}
```

- [ ] **Step 2: Vérifier l'échec**

Run: `go test ./internal/game -run 'TestRoom|TestRegistry'`

Expected: FAIL parce que `Room` et `Registry` n'existent pas.

- [ ] **Step 3: Implémenter la boucle propriétaire**

```go
type Sink interface {
    Send(Event) bool
    Close(reason string)
}

type command struct {
    kind    commandKind
    actorID string
    payload any
    reply   chan commandResult
}

type Room struct {
    code     string
    commands chan command
    done     chan struct{}
}

func (r *Room) run(state *roomState) {
    defer close(r.done)
    for {
        select {
        case cmd := <-r.commands:
            if state.handle(cmd) {
                return
            }
        case <-state.roundTimer:
            state.finishRound("")
        case <-state.resultTimer:
            state.advanceOrClose()
        }
    }
}
```

Les méthodes publiques envoient une commande et attendent la réponse avec le contexte appelant. `Attach(ctx, token, sink)` retourne l'identifiant du joueur associé au jeton. Seule `run` modifie `roomState`. Les événements de dessin et le chat sont bornés ; un `Sink.Send` faux provoque le détachement du client.

Le registre génère un code de six caractères avec `crypto/rand`, refuse les collisions, expose `Create`, `Join`, `Get`, `ListWaiting` et `Close`, puis ferme toutes les rooms lors de l'arrêt. Son constructeur reçoit :

```go
type Options struct {
    RoundDuration time.Duration
    ResultDelay   time.Duration
    MaxPlayers    int
}

registry := NewRegistry(bank, Options{
    RoundDuration: 90 * time.Second,
    ResultDelay:   4 * time.Second,
    MaxPlayers:    12,
})
```

- [ ] **Step 4: Vérifier concurrence et nettoyage**

Run:

```bash
go test ./internal/game
go test -race ./internal/game
```

Expected: PASS sans course ni goroutine bloquée.

- [ ] **Step 5: Commit**

```bash
git add internal/game/room.go internal/game/registry.go internal/game/room_test.go internal/game/registry_test.go
git commit -m "feat: add concurrent room registry"
```

### Task 5: Définir le protocole JSON et le client WebSocket

**Files:**
- Create: `internal/realtime/protocol.go`
- Create: `internal/realtime/client.go`
- Test: `internal/realtime/protocol_test.go`
- Modify: `go.mod`
- Modify: `go.sum`

- [ ] **Step 1: Tester le décodage strict**

```go
func TestDecodeClientMessage(t *testing.T) {
    msg, err := DecodeClientMessage([]byte(`{"type":"chat_message","payload":{"message":"bonjour"}}`))
    if err != nil {
        t.Fatal(err)
    }
    chat, ok := msg.(ChatMessage)
    if !ok || chat.Message != "bonjour" {
        t.Fatalf("unexpected message: %#v", msg)
    }
}

func TestDecodeRejectsUnknownType(t *testing.T) {
    if _, err := DecodeClientMessage([]byte(`{"type":"hack","payload":{}}`)); !errors.Is(err, ErrUnknownType) {
        t.Fatalf("got %v", err)
    }
}
```

- [ ] **Step 2: Installer WebSocket et lancer les tests**

Run:

```bash
go get github.com/coder/websocket@v1.8.15
go test ./internal/realtime
```

Expected: FAIL parce que le protocole n'existe pas.

- [ ] **Step 3: Implémenter l'enveloppe**

```go
type envelope struct {
    Type    string          `json:"type"`
    Payload json.RawMessage `json:"payload"`
}

type ChatMessage struct {
    Message string `json:"message"`
}

type DrawShape struct {
    Shape       string  `json:"shape"`
    StartX      float64 `json:"startX"`
    StartY      float64 `json:"startY"`
    EndX        float64 `json:"endX"`
    EndY        float64 `json:"endY"`
    StrokeColor string  `json:"strokeColor"`
    FillColor   string  `json:"fillColor,omitempty"`
    Fill        bool    `json:"fill"`
    Width       float64 `json:"width"`
}

func DecodeClientMessage(raw []byte) (any, error) {
    var header envelope
    if err := strictDecode(raw, &header); err != nil {
        return nil, err
    }
    switch header.Type {
    case "start_game":
        var value StartGame
        if err := strictDecode(header.Payload, &value); err != nil {
            return nil, err
        }
        return value, nil
    case "chat_message":
        var value ChatMessage
        if err := strictDecode(header.Payload, &value); err != nil {
            return nil, err
        }
        return value, nil
    case "draw_shape":
        var value DrawShape
        if err := strictDecode(header.Payload, &value); err != nil {
            return nil, err
        }
        return value, nil
    default:
        return nil, ErrUnknownType
    }
}
```

Compléter les structures pour `draw_stroke`, `erase_stroke`, `clear_canvas` et `ping`, puis valider type de forme, RGB hexadécimal, coordonnées entre 0 et 1, largeur entre 1 et 64, message entre 1 et 280 caractères.

- [ ] **Step 4: Implémenter le transport client**

`Client` implémente `game.Sink`, possède une file `send chan game.Event` de 64 éléments, une méthode `ReadLoop(ctx, handler)` et une méthode `WriteLoop(ctx)`. Appliquer `SetReadLimit(64 << 10)`, un délai d'écriture de cinq secondes, `Ping` toutes les 25 secondes et une fermeture WebSocket explicite.

- [ ] **Step 5: Vérifier et committer**

Run: `go test ./internal/realtime && gofmt -w internal/realtime`

Expected: PASS.

```bash
git add internal/realtime go.mod go.sum
git commit -m "feat: define typed WebSocket protocol"
```

### Task 6: Servir le front et l'API HTTP

**Files:**
- Create: `web/embed.go`
- Create: `web/index.html`
- Create: `internal/server/server.go`
- Create: `internal/server/handlers.go`
- Test: `internal/server/handlers_test.go`

- [ ] **Step 1: Écrire les tests HTTP**

```go
func TestCreateAndJoinRoom(t *testing.T) {
    app := newTestServer(t)
    created := postJSON(t, app, "/api/rooms", `{"nickname":"Lina"}`)
    if created.Code == "" || created.Token == "" || !created.Admin {
        t.Fatalf("unexpected response: %#v", created)
    }
    joined := postJSON(t, app, "/api/rooms/"+created.Code+"/join", `{"nickname":"Yanis"}`)
    if joined.Token == "" || joined.Admin {
        t.Fatalf("unexpected join: %#v", joined)
    }
}

func TestListOnlyWaitingRooms(t *testing.T) {
    app := newTestServer(t)
    createRoom(t, app, "Lina")
    response := getJSON[[]roomResponse](t, app, "/api/rooms")
    if len(response) != 1 {
        t.Fatalf("got %d rooms", len(response))
    }
}
```

- [ ] **Step 2: Créer un HTML minimal et vérifier l'échec API**

Dans `web/index.html`, créer les racines `#app`, `#notifications` et charger `/assets/js/app.js` en `type="module"`.

Run: `go test ./internal/server -run 'TestCreate|TestList'`

Expected: FAIL car les handlers n'existent pas.

- [ ] **Step 3: Implémenter le routeur**

```go
func New(registry *game.Registry, assets fs.FS, logger *slog.Logger) http.Handler {
    app := &App{registry: registry, assets: assets, logger: logger}
    mux := http.NewServeMux()
    mux.HandleFunc("GET /healthz", app.health)
    mux.HandleFunc("GET /api/rooms", app.listRooms)
    mux.HandleFunc("POST /api/rooms", app.createRoom)
    mux.HandleFunc("POST /api/rooms/{code}/join", app.joinRoom)
    mux.HandleFunc("GET /ws", app.websocket)
    mux.HandleFunc("GET /room/{code}", app.index)
    mux.Handle("GET /assets/", app.static())
    mux.HandleFunc("GET /{$}", app.index)
    return securityHeaders(recoverer(logger, mux))
}
```

Décoder les corps avec `DisallowUnknownFields`, limiter à 4 KiB, répondre en JSON, utiliser `404` pour room absente, `409` pour pseudo pris et `422` pour données invalides. Poser `Content-Security-Policy`, `X-Content-Type-Options` et `Referrer-Policy`.

- [ ] **Step 4: Vérifier et committer**

Run: `go test ./internal/server`

Expected: PASS.

```bash
git add web/embed.go web/index.html internal/server/server.go internal/server/handlers.go internal/server/handlers_test.go
git commit -m "feat: add room HTTP API and embedded frontend"
```

### Task 7: Relier WebSocket, protocole et rooms

**Files:**
- Create: `internal/server/websocket.go`
- Test: `internal/server/websocket_test.go`

- [ ] **Step 1: Écrire le test multiclient**

```go
func TestWebSocketBroadcastsChatAndRejectsDrawerGuess(t *testing.T) {
    app := newTestTLSServer(t)
    admin := createAndConnect(t, app, "Lina")
    guest := joinAndConnect(t, app, admin.Code, "Yanis")

    admin.Write(t, "start_game", map[string]any{})
    guest.WaitFor(t, "round_started")
    guest.Write(t, "chat_message", map[string]any{"message": "bonjour"})

    got := admin.WaitFor(t, "chat_message")
    if got.Payload["message"] != "bonjour" {
        t.Fatalf("unexpected payload: %#v", got)
    }
}
```

- [ ] **Step 2: Lancer le test rouge**

Run: `go test ./internal/server -run TestWebSocket`

Expected: FAIL car `App.websocket` n'est pas relié.

- [ ] **Step 3: Implémenter le handler**

```go
func (a *App) websocket(w http.ResponseWriter, r *http.Request) {
    room, ok := a.registry.Get(strings.ToUpper(r.URL.Query().Get("room")))
    if !ok {
        writeAPIError(w, http.StatusNotFound, "room_not_found", "Room introuvable.")
        return
    }
    conn, err := websocket.Accept(w, r, &websocket.AcceptOptions{})
    if err != nil {
        a.logger.Warn("websocket rejected", "error", err)
        return
    }
    client := realtime.NewClient(conn)
    playerID, err := room.Attach(r.Context(), r.URL.Query().Get("token"), client)
    if err != nil {
        client.Close("Session invalide.")
        return
    }
    defer room.Detach(context.WithoutCancel(r.Context()), playerID)
    client.Run(r.Context(), func(message any) error {
        return room.Dispatch(r.Context(), playerID, message)
    })
}
```

Même origine reste le comportement par défaut. Mapper chaque message entrant vers la commande de room correspondante et chaque `game.Event` vers son enveloppe JSON. Ne jamais journaliser le jeton.

- [ ] **Step 4: Vérifier intégration et courses**

Run:

```bash
go test ./internal/server
go test -race ./internal/server
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/server/websocket.go internal/server/websocket_test.go
git commit -m "feat: connect rooms through WebSocket"
```

### Task 8: Construire l'état et la navigation du front

**Files:**
- Create: `package.json`
- Create: `web/assets/js/store.js`
- Create: `web/assets/js/store.test.js`
- Create: `web/assets/js/api.js`
- Create: `web/assets/js/socket.js`
- Create: `web/assets/js/views.js`
- Create: `web/assets/js/app.js`
- Modify: `web/index.html`

- [ ] **Step 1: Tester le store**

```js
import test from "node:test";
import assert from "node:assert/strict";
import { createStore } from "./store.js";

test("room_state replaces players without leaking the secret word", () => {
  const store = createStore();
  store.dispatch({
    type: "room_state",
    payload: { code: "ABC123", players: [{ id: "1", nickname: "Lina" }], maskedWord: "_ a _" },
  });
  assert.equal(store.getState().room.code, "ABC123");
  assert.equal(store.getState().room.players[0].nickname, "Lina");
  assert.equal(store.getState().room.word, undefined);
});
```

- [ ] **Step 2: Configurer Node et vérifier l'échec**

Créer :

```json
{
  "private": true,
  "type": "module",
  "scripts": {
    "test": "node --test \"web/assets/js/*.test.js\"",
    "check": "node --check web/assets/js/*.js"
  }
}
```

Run: `npm test`

Expected: FAIL parce que `createStore` n'existe pas.

- [ ] **Step 3: Implémenter le store et les adaptateurs réseau**

```js
export function createStore() {
  let state = {
    screen: "home",
    session: null,
    room: null,
    chat: [],
    drawing: [],
    error: null,
  };
  const listeners = new Set();
  return {
    getState: () => structuredClone(state),
    subscribe(listener) {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
    dispatch(event) {
      state = reduce(state, event);
      listeners.forEach((listener) => listener(state));
    },
  };
}
```

`api.js` expose `listRooms`, `createRoom` et `joinRoom` avec contrôle de `response.ok`. `socket.js` construit automatiquement `ws:` ou `wss:`, reconnecte uniquement avant le démarrage avec un délai borné et distribue les enveloppes au store.

- [ ] **Step 4: Implémenter les vues**

`views.js` crée avec DOM API les écrans accueil, lobby, jeu et classement. Tous les textes issus des joueurs utilisent `textContent`. `app.js` relie formulaires, historique du navigateur, API, socket et store.

- [ ] **Step 5: Vérifier et committer**

Run:

```bash
npm test
npm run check
go test ./internal/server
```

Expected: PASS.

```bash
git add package.json web
git commit -m "feat: add vanilla JavaScript game client"
```

### Task 9: Implémenter le moteur Canvas et les outils

**Files:**
- Create: `web/assets/js/canvas-model.js`
- Create: `web/assets/js/canvas-model.test.js`
- Create: `web/assets/js/canvas-engine.js`
- Modify: `web/assets/js/app.js`
- Modify: `web/assets/js/views.js`

- [ ] **Step 1: Tester normalisation et formes**

```js
test("normalizes pointer coordinates", () => {
  assert.deepEqual(normalizePoint({ clientX: 150, clientY: 100 }, {
    left: 50, top: 50, width: 200, height: 100,
  }), { x: 0.5, y: 0.5 });
});

test("clamps RGB and widths", () => {
  assert.equal(rgbToHex({ r: 104, g: 71, b: 237 }), "#6847ED");
  assert.equal(clampWidth(100), 64);
});

test("builds a filled rectangle operation", () => {
  const shape = createShape("rectangle", { x: 0.1, y: 0.2 }, { x: 0.5, y: 0.7 }, {
    strokeColor: "#6847ED", fillColor: "#EF5E87", fill: true, width: 8,
  });
  assert.equal(shape.fill, true);
  assert.equal(shape.endX, 0.5);
});
```

- [ ] **Step 2: Vérifier l'échec**

Run: `npm test`

Expected: FAIL parce que les fonctions Canvas n'existent pas.

- [ ] **Step 3: Implémenter le modèle pur**

```js
export function normalizePoint(event, rect) {
  return {
    x: Math.max(0, Math.min(1, (event.clientX - rect.left) / rect.width)),
    y: Math.max(0, Math.min(1, (event.clientY - rect.top) / rect.height)),
  };
}

export function rgbToHex({ r, g, b }) {
  return `#${[r, g, b]
    .map((value) => Math.max(0, Math.min(255, Math.round(value))).toString(16).padStart(2, "0"))
    .join("")
    .toUpperCase()}`;
}

export function createShape(shape, start, end, style) {
  return {
    shape,
    startX: start.x,
    startY: start.y,
    endX: end.x,
    endY: end.y,
    ...style,
  };
}
```

- [ ] **Step 4: Implémenter le rendu interactif**

`CanvasEngine` gère Pointer Events, conversion HiDPI, aperçu sur un calque temporaire, lots de points toutes les 16 ms et replay complet. Implémenter crayon, gomme avec `destination-out`, trait, rectangle, cercle et triangle. Les opérations reçues du serveur sont l'unique historique validé ; une opération locale est affichée immédiatement puis réconciliée.

- [ ] **Step 5: Vérifier et committer**

Run:

```bash
npm test
npm run check
go test ./...
```

Expected: PASS.

```bash
git add web/assets/js/canvas-model.js web/assets/js/canvas-model.test.js web/assets/js/canvas-engine.js web/assets/js/app.js web/assets/js/views.js
git commit -m "feat: add synchronized Canvas drawing tools"
```

### Task 10: Réaliser l'interface desktop et le sélecteur RGB

**Files:**
- Create: `web/assets/css/app.css`
- Create: `web/assets/js/color.js`
- Create: `web/assets/js/color.test.js`
- Modify: `web/index.html`
- Modify: `web/assets/js/views.js`
- Modify: `web/assets/js/app.js`

- [ ] **Step 1: Tester les conversions du sélecteur**

```js
test("round trips RGB and HSV", () => {
  const hsv = rgbToHsv({ r: 104, g: 71, b: 237 });
  const rgb = hsvToRgb(hsv);
  assert.deepEqual(rgb, { r: 104, g: 71, b: 237 });
});

test("parses strict hexadecimal colors", () => {
  assert.deepEqual(hexToRgb("#EF5E87"), { r: 239, g: 94, b: 135 });
  assert.equal(hexToRgb("javascript:alert(1)"), null);
});
```

- [ ] **Step 2: Lancer les tests rouges**

Run: `npm test`

Expected: FAIL parce que `rgbToHsv`, `hsvToRgb` et `hexToRgb` n'existent pas.

- [ ] **Step 3: Implémenter le sélecteur**

`color.js` crée un panneau avec carré saturation/luminosité, curseur de teinte, aperçu, valeur hexadécimale et champs numériques R/G/B. Les valeurs sont bornées, synchronisées dans les deux sens et appliquées séparément à `strokeColor` et `fillColor`.

- [ ] **Step 4: Implémenter la mise en page validée**

Créer les variables et la grille principales :

```css
:root {
  --ink: #151127;
  --panel: #1b172e;
  --panel-raised: #29233f;
  --line: #352e50;
  --paper: #fffdf8;
  --accent: #ffd23f;
  --purple: #6847ed;
  --danger: #ff506f;
}

.game-layout {
  min-width: 1100px;
  height: 100dvh;
  display: grid;
  grid-template-columns: 220px minmax(560px, 1fr) 260px;
  grid-template-rows: 70px minmax(0, 1fr);
}

.canvas-stage {
  min-width: 0;
  min-height: 0;
  display: grid;
  grid-template-rows: minmax(0, 1fr) 64px;
}

.drawing-surface {
  width: 100%;
  height: 100%;
  background: var(--paper);
  border-radius: 16px;
}
```

La gauche affiche toujours code, joueurs, rôles et scores. Le centre réserve toute la hauteur restante au Canvas. Le chat reste compact à droite. Utiliser les icônes SVG crayon, gomme, trait, carré, rond et triangle validées. Séparer visuellement « Tout effacer » et demander confirmation.

- [ ] **Step 5: Vérifier et committer**

Run:

```bash
npm test
npm run check
go test ./...
```

Expected: PASS.

```bash
git add web/index.html web/assets
git commit -m "feat: build desktop drawing interface"
```

### Task 11: Ajouter l'entrée du programme et les erreurs fatales

**Files:**
- Create: `cmd/guesswhat/main.go`
- Test: `cmd/guesswhat/main_test.go`
- Modify: `internal/server/server.go`

- [ ] **Step 1: Tester les codes de sortie**

```go
func TestRunReturns84OnInvalidConfig(t *testing.T) {
    t.Setenv("GUESSWHAT_ADDR", "invalid")
    var stderr bytes.Buffer
    if code := run(t.Context(), &stderr); code != 84 {
        t.Fatalf("got %d", code)
    }
    if !strings.Contains(stderr.String(), "GUESSWHAT_ADDR") {
        t.Fatalf("missing error: %s", stderr.String())
    }
}
```

- [ ] **Step 2: Vérifier l'échec**

Run: `go test ./cmd/guesswhat`

Expected: FAIL parce que `run` n'existe pas.

- [ ] **Step 3: Implémenter l'entrée**

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()
    os.Exit(run(ctx, os.Stderr))
}

func run(ctx context.Context, stderr io.Writer) int {
    logger := slog.New(slog.NewTextHandler(stderr, nil))
    cfg, err := config.Load()
    if err != nil {
        logger.Error("configuration failed", "error", err)
        return 84
    }
    wordsFile, err := data.OpenWords()
    if err != nil {
        logger.Error("word bank failed", "error", err)
        return 84
    }
    defer wordsFile.Close()
    bank, err := game.NewWordBank(wordsFile)
    if err != nil {
        logger.Error("word bank failed", "error", err)
        return 84
    }
    registry := game.NewRegistry(bank, game.Options{
        RoundDuration: cfg.RoundDuration,
        ResultDelay:   cfg.ResultDelay,
        MaxPlayers:    cfg.MaxPlayers,
    })
    defer registry.CloseAll()
    if err := server.ListenAndServe(ctx, cfg.Address, registry, web.Assets(), logger); err != nil {
        logger.Error("server failed", "error", err)
        return 84
    }
    return 0
}
```

`ListenAndServe` démarre le serveur dans une goroutine, attend le contexte ou une erreur d'écoute, puis appelle `Shutdown` avec cinq secondes de délai. `http.ErrServerClosed` après un signal est un arrêt normal.

- [ ] **Step 4: Vérifier et committer**

Run:

```bash
go test ./cmd/guesswhat
go build -o guesswhat ./cmd/guesswhat
```

Expected: PASS et binaire créé.

```bash
git add cmd/guesswhat internal/server/server.go
git commit -m "feat: add graceful GuessWhat server entrypoint"
```

### Task 12: Durcir et valider le produit complet

**Files:**
- Modify: `internal/game/room_test.go`
- Modify: `internal/realtime/protocol_test.go`
- Modify: `internal/server/websocket_test.go`
- Modify: `web/assets/js/*.test.js`

- [ ] **Step 1: Ajouter les scénarios d'acceptation automatisés**

Ajouter des tests nommés :

```text
TestRoomClosesAfterEveryInitialPlayerDrew
TestDrawerDisconnectEndsRound
TestAdminDisconnectTransfersOwnership
TestLateJoinerIsSpectatorAndReceivesDrawingHistory
TestSlowClientIsDisconnectedWithoutBlockingRoom
TestProtocolRejectsOversizedAndOutOfRangeDrawing
TestWebSocketNeverSendsSecretToGuessers
```

Chaque test doit vérifier les événements exacts, l'état final et la fermeture des ressources.

- [ ] **Step 2: Exécuter toute la validation**

Run:

```bash
gofmt -w cmd data internal web/embed.go
go vet ./...
go test ./...
go test -race ./...
npm test
npm run check
go build -o guesswhat ./cmd/guesswhat
```

Expected: toutes les commandes réussissent.

- [ ] **Step 3: Effectuer le test manuel multiclient**

Run: `GUESSWHAT_ADDR=:8080 ./guesswhat`

Ouvrir trois fenêtres sur `http://localhost:8080`, puis vérifier dans cet ordre :

1. création de room et entrée de deux joueurs ;
2. démarrage réservé à l'administrateur ;
3. mot complet visible uniquement par le dessinateur ;
4. une lettre aléatoire visible chez les joueurs ;
5. crayon, gomme, formes, contour, remplissage et RGB synchronisés ;
6. proposition incorrecte publiée, bonne réponse masquée ;
7. points, changement de dessinateur et classement ;
8. fermeture de la room après le dernier tour.

Expected: aucun message d'erreur console ou serveur et comportement conforme à chaque point.

- [ ] **Step 4: Vérifier la livraison et committer**

Run:

```bash
git status --short
git ls-files
```

Confirmer qu'aucun binaire, fichier temporaire ou contenu `.superpowers` n'est suivi.

```bash
git add internal web
git commit -m "test: cover realtime game acceptance flows"
```

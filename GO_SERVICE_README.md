# Go Demo Parser Microservice

This Go microservice uses `demoinfocs-golang` to parse CS2 demo files and send the parsed data back to your EliteScout application.

## Prerequisites

- Go 1.24 or higher
- Access to your Supabase storage bucket
- The `GO_SERVICE_SECRET` you configured

## Setup

1. Install dependencies:
```bash
go mod init demo-parser
go get -u github.com/markus-wa/demoinfocs-golang/v5/pkg/demoinfocs
```

2. Create `main.go` with the service code below

3. Set environment variables:
```bash
export SUPABASE_URL="https://bmojcirhkifjtyfsnqve.supabase.co"
export SUPABASE_SERVICE_ROLE_KEY="your-service-role-key"
export GO_SERVICE_SECRET="your-secret"
export WEBHOOK_URL="https://bmojcirhkifjtyfsnqve.supabase.co/functions/v1/demo-webhook"
```

4. Run the service:
```bash
go run main.go
```

## Deployment Options

### Option 1: Railway
1. Push code to GitHub
2. Connect to Railway
3. Set environment variables in Railway dashboard
4. Deploy

### Option 2: Google Cloud Run
```bash
gcloud run deploy demo-parser \
  --source . \
  --platform managed \
  --region us-central1 \
  --set-env-vars SUPABASE_URL=...,SUPABASE_SERVICE_ROLE_KEY=...,GO_SERVICE_SECRET=...,WEBHOOK_URL=...
```

### Option 3: Docker
```bash
docker build -t demo-parser .
docker run -p 8080:8080 \
  -e SUPABASE_URL=... \
  -e SUPABASE_SERVICE_ROLE_KEY=... \
  -e GO_SERVICE_SECRET=... \
  -e WEBHOOK_URL=... \
  demo-parser
```

## Service Code

Create `main.go`:

\`\`\`go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"time"

	demoinfocs "github.com/markus-wa/demoinfocs-golang/v5/pkg/demoinfocs"
	common "github.com/markus-wa/demoinfocs-golang/v5/pkg/demoinfocs/common"
	events "github.com/markus-wa/demoinfocs-golang/v5/pkg/demoinfocs/events"
)

type ParseRequest struct {
	DemoID   string \`json:"demoId"\`
	FilePath string \`json:"filePath"\`
}

type ParsedData struct {
	DemoID   string      \`json:"demoId"\`
	Players  []Player    \`json:"players"\`
	Rounds   []Round     \`json:"rounds"\`
	Events   []GameEvent \`json:"events"\`
	Metadata Metadata    \`json:"metadata"\`
}

type Player struct {
	SteamID string                 \`json:"steamId"\`
	Name    string                 \`json:"name"\`
	Team    string                 \`json:"team"\`
	Kills   int                    \`json:"kills"\`
	Deaths  int                    \`json:"deaths"\`
	Assists int                    \`json:"assists"\`
	ADR     float64                \`json:"adr"\`
	HSP     float64                \`json:"hsp"\`
	KAST    float64                \`json:"kast"\`
	Rating  float64                \`json:"rating"\`
	Stats   map[string]interface{} \`json:"stats"\`
}

type Round struct {
	RoundNumber     int                    \`json:"roundNumber"\`
	WinnerSide      string                 \`json:"winnerSide"\`
	WinReason       string                 \`json:"winReason"\`
	CTScore         int                    \`json:"ctScore"\`
	TScore          int                    \`json:"tScore"\`
	DurationSeconds int                    \`json:"durationSeconds"\`
	RoundData       map[string]interface{} \`json:"roundData"\`
}

type GameEvent struct {
	EventType   string                 \`json:"eventType"\`
	Tick        int                    \`json:"tick"\`
	RoundNumber int                    \`json:"roundNumber"\`
	EventData   map[string]interface{} \`json:"eventData"\`
}

type Metadata struct {
	MapName  string \`json:"mapName"\`
	Duration int    \`json:"duration"\`
	TickRate int    \`json:"tickRate"\`
}

func main() {
	http.HandleFunc("/parse", parseHandler)
	http.HandleFunc("/health", healthHandler)

	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	log.Printf("Demo parser service starting on port %s", port)
	if err := http.ListenAndServe(":"+port, nil); err != nil {
		log.Fatal(err)
	}
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("OK"))
}

func parseHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	var req ParseRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	log.Printf("Parsing demo: %s", req.DemoID)

	// Download demo from Supabase Storage
	demoData, err := downloadDemo(req.FilePath)
	if err != nil {
		log.Printf("Error downloading demo: %v", err)
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	// Parse demo
	parsedData, err := parseDemo(req.DemoID, demoData)
	if err != nil {
		log.Printf("Error parsing demo: %v", err)
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	// Send to webhook
	if err := sendToWebhook(parsedData); err != nil {
		log.Printf("Error sending to webhook: %v", err)
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string]string{"status": "success"})
}

func downloadDemo(filePath string) ([]byte, error) {
	supabaseURL := os.Getenv("SUPABASE_URL")
	serviceRoleKey := os.Getenv("SUPABASE_SERVICE_ROLE_KEY")

	url := fmt.Sprintf("%s/storage/v1/object/demos/%s", supabaseURL, filePath)
	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		return nil, err
	}

	req.Header.Set("Authorization", "Bearer "+serviceRoleKey)

	client := &http.Client{Timeout: 2 * time.Minute}
	resp, err := client.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("failed to download demo: status %d", resp.StatusCode)
	}

	return io.ReadAll(resp.Body)
}

func parseDemo(demoID string, data []byte) (*ParsedData, error) {
	parsed := &ParsedData{
		DemoID:  demoID,
		Players: []Player{},
		Rounds:  []Round{},
		Events:  []GameEvent{},
		Metadata: Metadata{},
	}

	// Player stats tracking
	playerStats := make(map[uint64]*Player)
	currentRound := 0
	startTick := 0

	p := demoinfocs.NewParser(bytes.NewReader(data))

	// Register event handlers
	p.RegisterEventHandler(func(e events.RoundStart) {
		currentRound++
		startTick = p.GameState().IngameTick()
	})

	p.RegisterEventHandler(func(e events.RoundEnd) {
		gs := p.GameState()
		parsed.Rounds = append(parsed.Rounds, Round{
			RoundNumber:     currentRound,
			WinnerSide:      e.Winner.String(),
			WinReason:       e.Reason.String(),
			CTScore:         gs.TeamCounterTerrorists().Score(),
			TScore:          gs.TeamTerrorists().Score(),
			DurationSeconds: (p.GameState().IngameTick() - startTick) / 128,
			RoundData:       map[string]interface{}{},
		})
	})

	p.RegisterEventHandler(func(e events.Kill) {
		if e.Killer != nil {
			killer := getOrCreatePlayer(playerStats, e.Killer)
			killer.Kills++
		}
		if e.Victim != nil {
			victim := getOrCreatePlayer(playerStats, e.Victim)
			victim.Deaths++
		}
		if e.Assister != nil {
			assister := getOrCreatePlayer(playerStats, e.Assister)
			assister.Assists++
		}

		// Store kill event
		eventData := map[string]interface{}{
			"killer":      getPlayerName(e.Killer),
			"victim":      getPlayerName(e.Victim),
			"weapon":      e.Weapon.String(),
			"isHeadshot":  e.IsHeadshot,
			"penetrated":  e.PenetratedObjects,
		}

		parsed.Events = append(parsed.Events, GameEvent{
			EventType:   "kill",
			Tick:        p.GameState().IngameTick(),
			RoundNumber: currentRound,
			EventData:   eventData,
		})
	})

	// Parse the demo
	if err := p.ParseToEnd(); err != nil {
		return nil, err
	}

	// Convert player stats
	for _, player := range playerStats {
		parsed.Players = append(parsed.Players, *player)
	}

	// Set metadata
	header := p.Header()
	parsed.Metadata = Metadata{
		MapName:  header.MapName,
		Duration: int(p.Header().PlaybackTime.Seconds()),
		TickRate: int(p.Header().FrameRate()),
	}

	return parsed, nil
}

func getOrCreatePlayer(stats map[uint64]*Player, p *common.Player) *Player {
	if p == nil {
		return nil
	}

	if player, exists := stats[p.SteamID64]; exists {
		return player
	}

	team := "spectator"
	if p.Team == common.TeamTerrorists {
		team = "T"
	} else if p.Team == common.TeamCounterTerrorists {
		team = "CT"
	}

	player := &Player{
		SteamID: fmt.Sprintf("%d", p.SteamID64),
		Name:    p.Name,
		Team:    team,
		Stats:   make(map[string]interface{}),
	}

	stats[p.SteamID64] = player
	return player
}

func getPlayerName(p *common.Player) string {
	if p == nil {
		return "unknown"
	}
	return p.Name
}

func sendToWebhook(data *ParsedData) error {
	webhookURL := os.Getenv("WEBHOOK_URL")
	secret := os.Getenv("GO_SERVICE_SECRET")

	jsonData, err := json.Marshal(data)
	if err != nil {
		return err
	}

	req, err := http.NewRequest("POST", webhookURL, bytes.NewBuffer(jsonData))
	if err != nil {
		return err
	}

	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Authorization", "Bearer "+secret)

	client := &http.Client{Timeout: 30 * time.Second}
	resp, err := client.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		body, _ := io.ReadAll(resp.Body)
		return fmt.Errorf("webhook returned status %d: %s", resp.StatusCode, string(body))
	}

	return nil
}
\`\`\`

## Dockerfile

Create `Dockerfile`:

\`\`\`dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /app
COPY go.* ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o demo-parser .

FROM alpine:latest
RUN apk --no-cache add ca-certificates

WORKDIR /root/
COPY --from=builder /app/demo-parser .

EXPOSE 8080
CMD ["./demo-parser"]
\`\`\`

## How It Works

1. Your app uploads a demo to Supabase Storage
2. The `analyze-demo` edge function calls this Go service with the demo ID and file path
3. This service:
   - Downloads the demo from Supabase Storage
   - Parses it using demoinfocs-golang
   - Extracts kills, deaths, player stats, rounds, and events
   - Sends the parsed data to the webhook endpoint
4. The webhook stores all data in your database
5. The AI analysis function uses this rich data for insights

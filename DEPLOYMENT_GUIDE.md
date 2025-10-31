# EliteScout Go Microservice Deployment Guide

## Architecture Overview

```
User uploads demo → Supabase Storage
      ↓
Edge Function (analyze-demo) → Go Microservice
      ↓                              ↓
AI Analysis ←─────────── Parsed Data (webhook)
      ↓
Database (players, rounds, events)
```

## Step 1: Get Your Supabase Credentials

You'll need these environment variables for the Go service:

### 1. SUPABASE_URL
Already available: `https://bmojcirhkifjtyfsnqve.supabase.co`

### 2. SUPABASE_SERVICE_ROLE_KEY
This is the secret service role key that allows the Go service to download demos from storage.

**To get it:**
<lov-actions>
  <lov-open-backend>Open Backend</lov-open-backend>
</lov-actions>

Then navigate to: Settings → API → Project API keys → Copy the `service_role` key (NOT the anon key)

### 3. GO_SERVICE_SECRET
You just set this! Use the same value you entered.

### 4. WEBHOOK_URL
```
https://bmojcirhkifjtyfsnqve.supabase.co/functions/v1/demo-webhook
```

## Step 2: Deploy the Go Service

### Option A: Railway (Recommended - Easiest)

1. Create account at [railway.app](https://railway.app)
2. Click "New Project" → "Deploy from GitHub repo"
3. Create a new GitHub repo with the Go service code from `GO_SERVICE_README.md`
4. Connect the repo to Railway
5. Add environment variables:
   - `SUPABASE_URL`
   - `SUPABASE_SERVICE_ROLE_KEY`
   - `GO_SERVICE_SECRET`
   - `WEBHOOK_URL`
6. Railway will auto-deploy and give you a URL like: `https://your-service.railway.app`

### Option B: Render

1. Create account at [render.com](https://render.com)
2. Create "New Web Service"
3. Connect your GitHub repo
4. Set build command: `go build -o demo-parser .`
5. Set start command: `./demo-parser`
6. Add the same environment variables
7. Deploy

### Option C: Google Cloud Run

```bash
gcloud run deploy demo-parser \
  --source . \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars SUPABASE_URL=https://bmojcirhkifjtyfsnqve.supabase.co,SUPABASE_SERVICE_ROLE_KEY=your-key,GO_SERVICE_SECRET=your-secret,WEBHOOK_URL=https://bmojcirhkifjtyfsnqve.supabase.co/functions/v1/demo-webhook
```

## Step 3: Configure Your App

Once deployed, add the Go service URL as a secret:

1. Copy your deployed service URL (e.g., `https://your-service.railway.app`)
2. I'll add it as a secret called `GO_SERVICE_URL`

Then the flow will work:
- User uploads demo
- Edge function calls your Go service
- Go service parses demo with demoinfocs-golang
- Sends detailed data back to webhook
- AI generates insights using the parsed data

## Testing

Test the Go service health endpoint:
```bash
curl https://your-service-url/health
```

Should return: `OK`

Test parsing (after deployment):
```bash
curl -X POST https://your-service-url/parse \
  -H "Content-Type: application/json" \
  -d '{"demoId":"test-id","filePath":"test-path"}'
```

## What Gets Parsed

The Go service extracts:
- **Player stats**: Kills, deaths, assists, ADR, headshot %, KAST, rating
- **Round data**: Winner, win reason, scores, duration
- **Events**: All kills with weapon, headshot status, wallbangs
- **Metadata**: Map name, demo duration, tick rate

All this data is stored in your database and available for:
- Detailed AI analysis
- Player dashboards
- Performance tracking
- Heatmap generation
- Recruitment insights

## Cost Considerations

- Railway: Free tier available, then ~$5-10/month
- Render: Free tier with limitations, then $7/month
- Google Cloud Run: Pay per use, ~$0.40 per million requests

For most use cases, the free tiers are sufficient to start!

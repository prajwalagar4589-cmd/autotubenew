# One Smart Minute: autonomous YouTube Shorts pipeline

One original, fact-checked, AI-produced YouTube Short every day at 6 PM IST. Zero running cost.

## How a daily run works (GitHub Actions, no human)
1. **Pick the pillar** by weekday: Mon AI · Tue Money · Wed Engineering · Thu Politics · Fri Technology · Sat Education · Sun General knowledge.
   Rotate 5 formats (news explained, myth vs fact, number story, how it works, compare), never repeating the last 3.
2. **Ingest.** Pull the pillar's RSS feeds (last 48h, widening to 120h), skipping stories already covered.
   Evergreen days, and thin-news days, use a Wikipedia article as the source instead.
3. **Select.** Nemotron picks one story, reading recent titles (no repeats) and the weekly performance memo.
4. **Write.** The script uses only the fetched source text: 5–8 segments, 110–145 words, a hook first, plus on-screen headlines and big-number cards.
5. **Fact-check gate.** Two checks run on every draft:
   - Hard filters: banned advice phrases and political language.
   - A second model compares every claim with the sources, checks money scripts for advice, and checks politics scripts for neutrality.
   
   It gets up to 3 rewrite rounds. If it still fails, **nothing is posted** and you get an alert.
6. **Voice.** Kokoro TTS (open source, CPU).
7. **Render.** Pexels portrait B-roll (with gradient fallback), word-highlighted captions, headline and stat overlays, and an "AI-generated" tag. Music is ducked under the voice, audio is normalised to −14 LUFS, output is 1080×1920.
8. **QA gate.** Checks resolution, duration, audio presence and loudness, and file size.
9. **Upload.** Via a Pipedream webhook, using Pipedream's YouTube connection (or directly through the YouTube API if you add Google Cloud credentials later). Sources, AI disclosure and disclaimers go in the description.
10. **Log + alert.** State is committed to `data/state.db`, the video is kept as an artifact for 7 days, and a Telegram message is sent.

Weekly (Sunday), `src/analytics.py` pulls views and retention, then writes `data/insights.md`, which softly steers topic and format choice.

## Cost
All ₹0:
- NVIDIA and OpenRouter free models
- Kokoro (open source)
- Pexels API
- Wikipedia API
- Pipedream free plan (upload), YouTube RSS (analytics)
- GitHub Actions free minutes: about 12–15 min per day, well within the free allowance for a private repo
- Telegram

## Your one-time checklist (~75 minutes)

### A. Channel (15 min)
1. Sign in to YouTube with the Google account that will own the channel and create a channel named **One Smart Minute**. Set the handle; `@onesmartminute` or the closest free one.
2. Verify the channel's phone number at youtube.com/verify.
3. In YouTube Studio → Customization:
   - Upload `branding/avatar.png` and `branding/banner.png`.
   - Paste the description from `docs/channel_about.md`.

### B. GitHub (15 min)
4. Create a **private** repo, e.g. `autotube`, and push this folder:
   ```
   git init && git add . && git commit -m init
   git branch -M main && git remote add origin https://github.com/<you>/autotube.git && git push -u origin main
   ```
   Make sure `.github/workflows/` is in the repo; the Actions tab should list 4 workflows.
5. Create a small **public** repo, e.g. `one-smart-minute-site`, containing the two files from `site/`.
   - First replace `CONTACT_EMAIL` and `CHANNEL_HANDLE` in them.
   - Then go to Settings → Pages → Deploy from branch → `main` / root.
   - Note the URL, e.g. `https://<you>.github.io/one-smart-minute-site/`.

### C. Free API keys (15 min)
6. **NVIDIA:** go to build.nvidia.com, sign in, open the Nemotron 3 Ultra page and click "Generate API Key".
7. **OpenRouter:** go to openrouter.ai → Keys → Create key. This is the free fallback.
   - Optional: choose a free non-NVIDIA model at openrouter.ai/models?max_price=0 and put it first under `llm.checker` in `config.yaml`, so the fact-checker is a different model family from the writer.
8. **Pexels:** go to pexels.com/api and request a key. It's instant and free.
9. **Telegram:**
   - Message @BotFather, send `/newbot`, and copy the token.
   - Send your new bot any message.
   - Open `https://api.telegram.org/bot<TOKEN>/getUpdates` and copy `chat.id`.

### D. Pipedream: uploads to YouTube, no Google Cloud needed (15 min)
Pipedream is a free automation service with its own YouTube connection. GitHub renders the video and sends it to a Pipedream webhook. Pipedream then uploads it to your channel.

10. Sign up at pipedream.com (you can use "Sign in with GitHub"). The free plan is enough: this uses 1 workflow, 1 connected account, and about 1–2 credits per day.
11. Create the trigger:
    - Go to **New workflow → Trigger: HTTP / Webhook → New Requests**.
    - Under *Authorization*, pick **Custom token** and set a long random string. That string is your `PIPEDREAM_TOKEN`.
    - Click Save. Copy the endpoint URL (`https://….m.pipedream.net`); that's your `PIPEDREAM_WEBHOOK_URL`.
12. Add both as GitHub secrets (step 14), then run **Daily Short** once with *dry run* unticked and *force* ticked. The video arrives in Pipedream as a test event. Nothing is uploaded yet.
13. Back in Pipedream, click **+** under the trigger and search **YouTube Data API → Upload Video**. Then:
    - Click **Connect account** and sign in with the channel's Google account. **Pick the One Smart Minute channel.**
    - Fill the fields by clicking into the trigger event:
      - Title → `{{steps.trigger.event.body.title}}`
      - Description → `{{steps.trigger.event.body.description}}`
      - File Path or URL → the video's **url** field (e.g. `{{steps.trigger.event.body.video.url}}`; click it in the event explorer so the path is exact)
      - Privacy Status → `public`
      - Tags → `{{steps.trigger.event.body.tags.split(",")}}`
    - Click **Test**. The video should appear on your channel.
    - Click **Deploy**.
14. Put the channel ID in `config.yaml` → `publish.channel_id`. It starts with `UC`; find it in YouTube Studio → Settings → Channel → Advanced settings. This powers free analytics via the channel's public RSS feed.

### E. Secrets + first runs (10 min)
15. In the repo, go to Settings → Secrets and variables → Actions and add:
    - `NVIDIA_API_KEY`
    - `OPENROUTER_API_KEY`
    - `PEXELS_API_KEY`
    - `PIPEDREAM_WEBHOOK_URL`
    - `PIPEDREAM_TOKEN`
    - `TELEGRAM_BOT_TOKEN`
    - `TELEGRAM_CHAT_ID`
16. Actions → **Doctor** → Run. Everything should say "set" or "ok". Delete any feed URL marked FAILED from `config.yaml`.
17. Actions → **Daily Short** → Run with *dry run* ticked. Download the `video` artifact and watch it.
18. Do step 12–13 if you haven't. From now on it runs by itself every day at 6 PM IST.

### Optional
- Add 10–20 tracks from the YouTube Audio Library (filter: *attribution not required*) to `assets/music/`. Without them, videos are voice-only.
- Change the voice in `config.yaml` (`tts.voice`), or the posting time in `.github/workflows/daily.yml`.

## Editing the channel
Everything editorial is in `config.yaml`:
- channel name
- weekday pillars
- formats
- RSS sources
- models
- voice
- banned phrases
- politics rules

Prompts live in `src/writer.py`.

## Optional: direct YouTube API instead of Pipedream
If Pipedream ever changes its free plan, the code still supports the direct YouTube Data API.
1. Create a Google Cloud project, enable YouTube Data API v3 and YouTube Analytics API, set the OAuth consent screen to **In production**, and create a Desktop OAuth client.
2. Run `python scripts/get_refresh_token.py client_secret.json`.
3. Add `YT_CLIENT_ID`, `YT_CLIENT_SECRET` and `YT_REFRESH_TOKEN` as secrets, and remove `PIPEDREAM_WEBHOOK_URL`.

Uploads from your own project stay private until you pass the YouTube API audit (`docs/youtube_audit_answers.md`).

## Troubleshooting
- **Uploads land as "Private (locked)" via Pipedream.** Pipedream's YouTube app would then be unaudited; use the direct API option above and submit the audit.
- **Pipedream run failed.** Open the workflow's event history in Pipedream; the error is shown on the Upload Video step. Its file URL expires after about 30 minutes, so re-run the GitHub job rather than replaying an old event.
- **"Skipped today" alert.** A gate did its job; the reason is in the message and in `data/state.db`.
- **All models failed.** Free tiers change. Swap model IDs in `config.yaml` (see openrouter.ai/models?max_price=0).
- **Upload 401/invalid_grant (direct API only).** The refresh token was revoked, or the consent screen was left in Testing.

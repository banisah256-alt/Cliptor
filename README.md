# ⬇️ Cliptor

A self-hosted web app to download YouTube videos (up to 4K), extract MP3 audio, make 30-second Shorts, and cut clips out of long videos. Built with Flask and [yt-dlp](https://github.com/yt-dlp/yt-dlp).

> **For personal use only.** Download only content you own, that is licensed for reuse, or that you have permission to download. See [Disclaimer](#disclaimer).

## Features

- 🎬 **Video download** in 4K, 2K, 1080p, 720p or 480p (MP4)
- 🎵 **MP3 audio** extraction (192 kbps)
- ⚡ **30-second Shorts**: pick a start time, get a 30 s clip
- ✂️ **Cliptor Cutter**: cut any start/end range from a long video
- ⏱️ Videos longer than **1 hour** are rejected (configurable in `app.py`)
- 🛡️ Per-IP rate limit, concurrent-job limit, auto-cleanup of temp files
- 📱 Responsive, light/dark UI. No database, no accounts, no ads

## Quick start

**Requirements:** [Docker](https://docs.docker.com/get-docker/) (includes ffmpeg, Redis and workers)

```bash
git clone https://github.com/<your-username>/cliptor.git
cd cliptor
docker compose up -d --build
```

Open http://localhost

Scale up: `docker compose up -d --scale worker=6` (each worker handles one download at a time).

## Architecture

```
Browser ─► web (Flask + gunicorn, stateless) ─► Redis queue ─► workers (yt-dlp + ffmpeg)
                    ▲                                                │
                    └──── /api/status (polling) ◄── shared volume ◄──┘  files auto-deleted
```

- Web containers only validate and queue, so they stay fast under load
- Workers do the heavy download/convert jobs; add more workers for more capacity
- Identical requests (same link, type, quality, range) are de-duplicated and served from one job
- Per-IP rate limit and queue cap protect the server; users see their position in the queue

## Configuration

Set these as environment variables:

| Variable | Default | Description |
|---|---|---|
| `PORT` | `5000` | Port to listen on |
| `MAX_QUEUE` | `50` | New requests are rejected when the queue is this long |
| `FILE_TTL` | `1800` | Seconds before finished files are deleted |
| `REDIS_URL` | compose default | Redis connection |
| `PROXY_URL` | none | HTTP proxy for yt-dlp (helps with IP blocks) |
| `RATE_LIMIT` | `10` | Downloads per IP per hour |
| `COOKIES_FILE` | none | Path to a `cookies.txt`, useful if YouTube asks to "confirm you're not a bot" |

The 1-hour limit is the `MAX_SECONDS` constant in `app.py`.

## Deploying

| Host | Works? | Notes |
|---|---|---|
| Your laptop / home server | ✅ | Easiest. Use Cloudflare Tunnel (`cloudflared tunnel --url http://localhost:5000`) to share a link |
| VPS (Hetzner, DigitalOcean, Oracle Always Free) | ✅ | 2 CPU / 4 GB RAM is comfortable for 4K |
| Render / Railway (Docker) | ⚠️ | Free tiers have low RAM; 4K and long videos may fail |
| Vercel / Netlify / GitHub Pages | ❌ | No ffmpeg, no long-running processes |

The real limits when traffic grows are **bandwidth**, **CPU** and **YouTube blocking your server IP**, not Flask. Cloud IPs are often blocked by YouTube. If downloads fail with a bot check, set `COOKIES_FILE`.

## How it works

```
Browser ──► /api/info     check link, read title & duration
        ──► /api/prepare  yt-dlp + ffmpeg download, merge, convert or cut
        ──► /api/file/<token>  download the finished file (deleted after 10 min)
```

The server runs with **one worker and multiple threads** (see `Dockerfile`) because download tokens are kept in memory.

## Project structure

```
cliptor/
├── app.py            Flask web app (API)
├── tasks.py          yt-dlp/ffmpeg worker logic
├── docker-compose.yml web + redis + workers
├── static/index.html Frontend
├── requirements.txt
├── Dockerfile
└── LICENSE
```

## Roadmap

- [ ] Live progress bar
- [ ] Subtitle download
- [ ] Playlist support
- [ ] Auto-scaling workers

Pull requests are welcome.

## Disclaimer

This project is provided for educational and personal use. Downloading content from YouTube may violate [YouTube's Terms of Service](https://www.youtube.com/t/terms) and copyright law in your country. You are solely responsible for how you use this software and for what you download. The authors do not host or distribute any media and are not affiliated with YouTube or Google. If you run a public instance, you are responsible for handling takedown requests.

## License

MIT, see [LICENSE](LICENSE).

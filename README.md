# 🐾 OnlyPanther — Gamified GSU Class Scraper

> *Choose your classes like a quest. Collect professor cards. Earn XP. Be the top Panther.*

OnlyPanther scrapes **live class data** from GSU's Banner (GoSolar/PAWS) registration system and enriches it with **RateMyProfessors** ratings, then presents everything as a gamified Pokémon-meets-Duolingo experience built for Georgia State University students.

---

## Features

| Feature | Description |
|---|---|
| **Live Banner Scraper** | Pulls all sections from GoSolar for Fall/Summer 2026 via the Ellucian Banner SSB public API |
| **RMP Integration** | Fetches prof ratings, difficulty, "would take again" from RateMyProfessors GraphQL |
| **Class Rarity System** | COMMON → UNCOMMON → RARE → EPIC → LEGENDARY based on seat availability |
| **Professor Cards** | Pokémon-style holographic cards ranked by RMP rating |
| **XP & Levels** | Earn XP by viewing classes, adding to schedule, completing challenges |
| **Daily Challenges** | 5 rotating daily quests with XP rewards |
| **Streaks** | Duolingo-style 7-day streak tracker |
| **Schedule Builder** | Drag-drop-style schedule with weekly calendar view and CSV export |
| **Leaderboard** | Global XP rankings |
| **SQLite Cache** | Classes cached for 1 hour, prof ratings cached for 24 hours |

---

## Quick Start

### Prerequisites

- Python 3.10+
- pip

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. Run the server

```bash
uvicorn app:app --reload --host 0.0.0.0 --port 8000
```

### 3. Open in browser

```
http://localhost:8000
```

The app immediately starts scraping GSU Banner in the background. The first load takes **1–2 minutes** to populate the database with all sections. A progress banner appears in the UI while scraping is active.

---

## API Reference

| Endpoint | Description |
|---|---|
| `GET /api/terms` | Available Banner terms |
| `GET /api/subjects?term=202608` | Subjects for a term |
| `GET /api/classes?term=&subject=&query=&rarity=&open_only=&page=&size=` | Paginated, filtered class list |
| `GET /api/prof/{name}` | RMP data for one professor |
| `POST /api/profs/batch` | Batch RMP data `{"names":[...]}` |
| `POST /api/scrape?term=&force=` | Trigger fresh scrape (background) |
| `GET /api/gamification/badges` | Badge definitions |
| `GET /api/gamification/challenges` | Daily challenge definitions |
| `GET /api/gamification/leaderboard` | Top players |
| `GET /api/health` | Scrape status |

---

## Architecture

```
┌─────────────────────────────────────┐
│           index.html (SPA)          │  Tailwind CSS · Vanilla JS
│  Explore · Quests · Profs · Schedule│  Particles · Animations
└──────────────┬──────────────────────┘
               │ HTTP / same-origin
┌──────────────▼──────────────────────┐
│          FastAPI (app.py)           │  Python 3.10+
│  /api/classes  /api/prof  /api/...  │  uvicorn
└──────┬───────────────┬──────────────┘
       │               │
┌──────▼─────┐  ┌──────▼──────────────┐
│  scraper.py │  │  onlypanther.db     │
│  Banner SSB │  │  SQLite cache       │
│  RMP GraphQL│  │  (classes + profs)  │
└─────────────┘  └─────────────────────┘
```

### Scraper Flow

1. `GET /classSearch/getTerms` — discover available terms
2. `GET /classSearch/get_subject` — list all subject codes
3. `POST /term/search` — activate term in Banner session
4. `GET /searchResults/searchResults` — paginate through all sections (500/request)
5. RMP GraphQL `newSearch.teachers` query per professor name

---

## Configuration

All configuration is done via environment variables or by editing the top of `scraper.py`:

| Variable | Default | Description |
|---|---|---|
| `GSU_BANNER_BASE` | `https://registration.gosolar.gsu.edu/StudentRegistrationSsb/ssb` | Banner base URL |
| `TERM_PRIORITIES` | `["202608","202605","202601"]` | Fall 2026, Summer 2026, Spring 2026 term codes |

**Term codes** follow Banner's YYYYTT format:
- `202608` = Fall 2026
- `202605` = Summer 2026
- `202601` = Spring 2026

---

## Deployment

### Docker

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
docker build -t onlypanther .
docker run -p 8000:8000 -v $(pwd)/onlypanther.db:/app/onlypanther.db onlypanther
```

### Systemd

```ini
[Unit]
Description=OnlyPanther GSU Class App

[Service]
WorkingDirectory=/opt/onlypanther
ExecStart=/usr/bin/uvicorn app:app --host 0.0.0.0 --port 8000 --workers 2
Restart=always

[Install]
WantedBy=multi-user.target
```

---

## Gamification Design

### Class Rarity (by seat availability)

| Rarity | Condition | Color |
|---|---|---|
| LEGENDARY | 0 seats left | 🟡 Gold |
| EPIC | ≤5% seats remaining | 🟣 Purple |
| RARE | ≤15% seats remaining | 🔵 Blue |
| UNCOMMON | ≤40% seats remaining | 🟢 Green |
| COMMON | >40% seats remaining | ⬜ Gray |

### Professor Rarity (by RMP avg rating)

| Rarity | Rating | Color |
|---|---|---|
| LEGENDARY | ≥4.5 | 🟡 Gold |
| EPIC | ≥4.0 | 🟣 Purple |
| RARE | ≥3.5 | 🔵 Blue |
| UNCOMMON | ≥3.0 | 🟢 Green |
| COMMON | <3.0 | ⬜ Gray |

### XP Formula

```python
xp = 50 (base) + fill_percent * 100 (rarity bonus) + credit_hours * 10
```

### Level Formula

```
XP to reach Level N = N² × 100
```

---

## Notes

- No GSU login is required — all data comes from Banner's **public** class search endpoint.
- RateMyProfessors data is fetched via their public GraphQL API with GSU's school ID (360).
- The SQLite database (`onlypanther.db`) is created automatically on first run.
- Player progress (XP, streaks, schedule) is saved in browser `localStorage`.

---

*Built with ❤️ for Georgia State University Panthers · Not affiliated with GSU or Ellucian*

# På spåret

A real-time, host-controlled quiz platform inspired by the Swedish TV show *På spåret*. One admin drives the entire game flow while players connect from their own devices to submit answers and view the live leaderboard. Designed for group gatherings, parties, or remote play over screen share.

---

## Architecture Overview

```
Browser (Player/Admin/Viewer)
        │
        │  HTTP + WebSocket (Socket.IO)
        ▼
Flask App (app.py)
        │
        ├── game_data.json  ← live game state (questions, answers, users, scores)
        └── /static/images/ ← question images
```

The backend is a single Flask application backed by a JSON file (`game_data.json`) that acts as the live game state. All connected clients receive real-time updates via Socket.IO events. There is no database — the JSON file is read and written on every relevant action, with an automatic backup to `game_data.backup.json` before each write.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Backend | Python, Flask 3, Flask-SocketIO 5 |
| Real-time | Socket.IO (Eventlet async mode) |
| Frontend | HTML5, Bootstrap 5.3, JavaScript (ES6+) |
| Mapping | Leaflet.js with CartoDB Positron tiles (no labels) |
| Notifications | SweetAlert2 |
| Production server | Gunicorn + Eventlet (Render-compatible) |

---

## Game Modes

### Player Mode (`/`)
Players connect from their own phones or laptops. After logging in they see:
- The current active question (text, clues, or interactive map)
- An answer input suited to the question type
- A live leaderboard sidebar
- All visible answers submitted by other players (shown after the admin makes them visible)

### Admin Mode (`/admin`)
A separate password-protected panel that controls the entire game. The admin:
- Selects which question to make active
- Reveals clues one-by-one (for decreasing-point questions)
- Shows or hides individual player answers
- Marks answers as correct (which awards points)
- Manually adjusts or removes player scores
- Triggers the correct-location reveal on map questions
- Clears all answers between rounds

### Viewer Mode (`/viewer`)
A read-only big-screen display intended for a projector or shared screen. It shows:
- The current question and any images on the left
- A live leaderboard on the right
- An interactive map with all player markers and (after reveal) the correct location marker in gold

---

## Question Types

Questions are defined in `game_data.json`. The `type` field controls how a question is rendered and scored.

### Standard (default)
A straightforward text answer. Can have multiple `parts`, each getting its own input field.

```json
{
  "id": 0,
  "name": "Capitals",
  "parts": ["What is the capital of France?", "What is the capital of Japan?"]
}
```

### Decreasing Points (`"type": "decreasing"`)
Players answer from a list of progressively easier clues. The later a player waits for more clues, the fewer points they can earn. The admin clicks "Next Clue" to reveal clues one at a time.

**Points formula:** `10 − (2 × clue_index)` → 10, 8, 6, 4, 2 pts

```json
{
  "id": 1,
  "type": "decreasing",
  "name": "Mystery Destination",
  "clues": [
    "I am the most visited paid monument in the world.",
    "I was built between 1887 and 1889.",
    "I stand 330 meters tall.",
    "I am in Western Europe.",
    "I am in Paris."
  ]
}
```

### Map (`"type": "map"`)
Players drop a pin on an interactive, label-free map. The system calculates Haversine distance (km) between the player's pin and the target coordinates. The admin can reveal the correct location to all viewers.

```json
{
  "id": 8,
  "type": "map",
  "name": "Where are we?",
  "text": "Frihetsgudinnan",
  "target_lat": 40.6892,
  "target_lon": -74.0445,
  "images": ["statue_hint.jpg"]
}
```

---

## `game_data.json` Structure

This file is the single source of truth for all live game state.

```json
{
  "questions": [ /* all questions loaded from setup */ ],
  "answers": [
    {
      "id": 1,
      "question_id": 0,
      "text": "Paris",
      "username": "Alice",
      "avatar": "/static/avatars/1.webp",
      "votes": [{ "username": "Bob", "avatar": "/static/avatars/2.webp" }],
      "potential_points": 8,
      "coordinates": { "lat": 48.8566, "lon": 2.3522 },
      "visible": false,
      "is_correct": false,
      "random_num": 342
    }
  ],
  "users": [
    {
      "username": "Alice",
      "avatar": "/static/avatars/1.webp",
      "score": 18
    }
  ],
  "current_question": { /* the currently active question object */ },
  "reveal": false
}
```

Key fields:
- `answers[].visible` — controls whether players can see an answer (admin toggles this)
- `answers[].is_correct` — marks an answer correct and awards `potential_points` to the player
- `answers[].potential_points` — locked in at submission time (decreasing questions) or set by admin (standard)
- `answers[].random_num` — used to randomize display order so players can't tell who answered first
- `reveal` — when `true`, all answers are fully revealed to players

---

## Admin Game Flow (Round by Round)

A typical round runs like this:

1. **Select question** — Admin picks from the dropdown and clicks "Set Next Question". This broadcasts `next_question` to all clients, clears old answers, and resets the reveal flag.
2. **Players answer** — Players submit their answer. For map questions they drop a pin; for decreasing questions the admin clicks "Next Clue" to reveal clues (broadcasting `clue_update`), each reducing the available points.
3. **Moderate answers** — Admin sees all answers in real time. They click "Show" to make individual answers visible to players, then "Mark Correct" to award points.
4. **Reveal** (optional) — For map questions, "Reveal Correct Location" broadcasts `show_target_location`, plotting the gold marker on the viewer map.
5. **Clear round** — "Clear Votes and Answers" resets all answers while preserving scores and questions, ready for the next round.

---

## Real-Time Socket.IO Events

| Event | Triggered by | Effect on clients |
|---|---|---|
| `next_question` | Admin sets next question | All views update to new question |
| `clue_update` | Admin clicks Next Clue | New clue appears; points value drops |
| `show_target_location` | Admin reveals map location | Gold marker appears on all maps |
| `answer_visibility_changed` | Admin shows/hides answer | Answer appears or disappears for players |
| `updated_answers` | Admin marks correct/incorrect | Correct answer highlighted in green |
| `votes_revealed` | Admin reveals votes | Votes become visible |
| `game_reset` | Admin clears round | All answers removed from all views |

---

## Scoring

- Each user starts at 0 points.
- Points are awarded when the admin marks an answer correct — the value is whatever `potential_points` was at submission time.
- For **decreasing** questions, `potential_points` = `10 − (2 × current_clue_index)` at the moment of submission.
- For **map** questions, `potential_points` is set manually by the admin before marking correct.
- The admin can also manually nudge scores with +1 / −1 buttons at any time.
- The leaderboard is sorted by score descending across all views.

---

## Authentication

There are three separate logins:

| Role | Route | Credential env var |
|---|---|---|
| All users (gate) | `/login` | `USER_PASSWORD` |
| Player profile | `/userlogin` | *(username + avatar, no password)* |
| Admin | `/admin_login` | `ADMIN_PASSWORD` |

Sessions persist for 7 days. Cookies are `HttpOnly`, `Secure`, and `SameSite=Lax`.

---

## API Endpoints

### Game state
| Method | Route | Description |
|---|---|---|
| GET | `/get_game_state` | Full game data (questions, answers, users, reveal flag) |
| POST | `/set_next_question` | Load a question and reset the round |
| POST | `/next_clue` | Reveal next clue (decreasing questions) |
| POST | `/clear_data` | Clear answers/votes, preserve scores |

### Answers
| Method | Route | Description |
|---|---|---|
| POST | `/submit_answer` | Player submits answer (handles all question types) |
| POST | `/toggle_answer_visibility` | Admin shows/hides an answer |
| POST | `/mark_correct` | Admin toggles correct flag and awards points |

### Scoring & users
| Method | Route | Description |
|---|---|---|
| POST | `/change_score` | Admin +1 or −1 a player's score |
| POST | `/delete_user` | Remove a player from the game |

---

## Prerequisites

- Python 3.8+
- pip

---

## Installation & Local Setup

```bash
# 1. Clone
git clone https://github.com/Kingof3O/quiz-game.git
cd quiz-game

# 2. Create and activate virtual environment
python -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Create .env file
SECRET_KEY=your_secret_key_here
ADMIN_PASSWORD=your_admin_password
USER_PASSWORD=your_user_password

# 5. Run
python app.py
```

The app runs on `http://localhost:5000` by default. Set `PORT` in `.env` to override.

---

## Adding Questions

Edit `game_data.json` directly. Add objects to the `"questions"` array following the structures shown in the [Question Types](#question-types) section. Place any question images in `static/images/` and reference them by filename in the `"images"` array.

---

## Deployment (Render)

The app is configured for [Render](https://render.com) out of the box:
- Gunicorn + Eventlet handles WebSocket connections in production
- The `PORT` environment variable is respected automatically
- Set `SECRET_KEY`, `ADMIN_PASSWORD`, and `USER_PASSWORD` as environment variables in the Render dashboard

---

## License

MIT License — see [LICENSE](LICENSE)

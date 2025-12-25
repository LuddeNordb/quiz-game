# På spåret

A real-time, interactive på spåret quiz platform featuring dynamic question types, geographic challenges.

---

## 🚀 Key Features

### 🎭 Diverse Question Types
* **Standard/Multi-Part:** Traditional Q&A or multi-step challenges with dynamic input boxes.
* **Decreasing Points:** High-stakes rounds where point values drop as more clues are revealed.
* **Geographic Map Challenges:** Interactive map questions where players drop pins. The system automatically calculates the Haversine distance (km) from the target coordinates.

### 📡 Real-Time Interaction
* **Live Viewer Page:** A dedicated, non-interactive "Big Screen" view showing the live leaderboard, current question, and clues.
* **Instant Updates:** Powered by **Socket.IO** for zero-latency clue reveals, answer submissions, and score updates.
* **Dynamic Image Support:** Questions can feature a gallery of images displayed across all player and viewer screens.

### 🗺️ Map Integration
* **Interactive Markers:** Players use a label-free map (CartoDB Positron) to prevent cheating via city names.
* **The Reveal:** Admin can trigger a "True Location" reveal that plots everyone's markers and the correct location (Gold Marker) simultaneously on the viewer map.

---

## 🛠️ Technology Stack

* **Backend:** Python Flask & Flask-SocketIO
* **Frontend:** HTML5, CSS3, JavaScript (ES6+), Bootstrap 5
* **Mapping:** Leaflet.js
* **Production Server:** Gunicorn with Eventlet (optimized for WebSockets on Render)

---

## 🎮 Game Modes

### 1. Player Mode (`/`)
- Choose a unique avatar and join the lobby.
- Submit answers via text or interactive map pins.
- Points are calculated based on submission timing or distance accuracy.

### 2. Admin Mode (`/admin`)
- Full control over the game flow and question selection.
- Reveal clues one-by-one for decreasing-point rounds.
- Moderate answers and trigger global "Reveal" events.
- Special "Reveal Correct Location" button for map-based questions.

### 3. Viewer Mode (`/viewer`)
- Designed for projectors or Twitch/Discord screen sharing.
- Split-screen layout: Questions/Map on the left, Scoreboard on the right.
- Visualizes "Distance Lines" between player guesses and the truth.

---

## Prerequisites

- Python 3.8+
- pip

## 📦 Installation & Local Setup

1. Clone the repository
   ```bash
   git clone https://github.com/Kingof3O/quiz-game.git
   cd quiz-game
   ```

2. Create virtual environment
   ```bash
   python -m venv venv
   source venv/bin/activate  # Windows: venv\Scripts\activate
   ```

3. Install dependencies
   ```bash
   pip install -r requirements.txt
   ```

4. Configure environment variables
   Create a `.env` file with:
   ```
   SECRET_KEY=your_secret_key
   ADMIN_PASSWORD=your_admin_password
   USER_PASSWORD=your_user_password
   ```

## Running the Application

```bash
python app.py
```

## Security Notes

- Use strong, unique passwords
- Keep `.env` file private
- Do not share credentials


## License

MIT License - see [LICENSE](LICENSE) file

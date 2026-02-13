# Rock Paper Scissors — Arduino UNO Q

A real-time Rock-Paper-Scissors game running on the Arduino UNO Q using an Edge Impulse object detection model and the Arduino App Lab bricks system.

<img width="1624" height="1056" alt="Playing Rock Paper Scissors against Arduino UNO Q" src="https://github.com/user-attachments/assets/4a424e48-33d7-4b16-a3ca-eb47b1708bb3" />

The camera detects your hand gesture (rock, paper, or scissors) via machine learning inference, while the Arduino picks a random move. A web UI served on port 5001 lets you play rounds, track scores, and view move history.

## How It Works

### Architecture

```
video_object_detection brick (Docker container)
  └── Manages USB camera + runs Edge Impulse .eim model
  └── Sends detection callbacks: {label: confidence}
        │
        ▼
main.py (Python on Linux MPU)
  ├── Receives brick callbacks → updates detection state
  ├── Game logic → scores, rounds, history
  └── Flask (port 5001) → serves web UI + REST API
        │
        ▼
App.run() → activates all bricks
```

The `video_object_detection` brick runs in a Docker container and has exclusive access to the USB camera. It performs inference using the Edge Impulse model and calls back into Python with detection results. There is no OpenCV or direct camera access from Python — the brick handles everything.

### Game Flow

1. Show your hand gesture (rock, paper, or scissors) to the camera
2. The detection panel on the left shows what the model sees in real time
3. Click **Play Round** — your gesture is **locked in** at that moment
4. A 3-2-1 countdown plays (detection is paused during the round)
5. The Arduino reveals its random move and the winner is shown
6. After 3 seconds the round ends and live detection resumes

### Detection Lock

When you press Play Round, the app snapshots whatever gesture the model currently detects. During the countdown and result display, new detections are ignored. This means:

- You must hold your gesture **before** clicking Play
- The panel turns blue to show your move is locked
- Detection resumes automatically after the result is shown

## How to Play

1. Deploy the app to your Arduino UNO Q (see below)
2. Open `http://<device-ip>:5001` in a browser on the same network
3. Point the USB camera at your hand
4. Wait for the detection panel to show your gesture (green background)
5. Click **Play Round**
6. See the result — scores update automatically
7. Click **Reset Game** to start over

## Project Structure

```
RSP-game/
├── app.yaml                    # App Lab config: brick + model path + ports
├── README.md                   # This file
└── python/
    ├── main.py                 # Flask server + game logic + brick callback
    ├── requirements.txt        # Python dependencies (flask)
    └── templates/
        └── index.html          # Game web UI (HTML/CSS/JS)
```

## Deployment

### Prerequisites

- Arduino UNO Q with Arduino App Lab
- USB camera connected to the board
- Edge Impulse `.eim` model trained to detect `rock`, `paper`, and `scissors`

### Step 1: Transfer the App

Copy the entire `RSP-game` folder to the board:

```bash
scp -r RSP-game/ arduino@<device-ip>:/home/arduino/ArduinoApps/RSP-game
```

### Step 2: Deploy the Model

Copy your Edge Impulse model and make it executable:

```bash
scp rcp-model.eim arduino@<device-ip>:/home/arduino/.arduino-bricks/ei-models/
ssh arduino@<device-ip> "chmod +x /home/arduino/.arduino-bricks/ei-models/rcp-model.eim"
```

### Step 3: Start the App

```bash
arduino-app-cli app start user:RSP-game
```

### Step 4: Open the Web UI

Navigate to `http://<device-ip>:5001` in your browser.

### Viewing Logs

```bash
arduino-app-cli app logs user:RSP-game --all
```

## Configuration

All settings are in [python/main.py](python/main.py) at the top:

| Setting | Default | Description |
|---------|---------|-------------|
| `CONFIDENCE_THRESHOLD` | `0.6` | Minimum confidence to accept a detection |
| `PORT` | `5001` | Flask web server port (also set `FLASK_PORT` env var) |
| `COUNTDOWN_SECS` | `3` | Countdown duration before evaluating |
| `RESULT_HOLD_SECS` | `3` | How long the result stays on screen |

### Changing the Model

Edit `app.yaml` and update the `EI_OBJ_DETECTION_MODEL` path:

```yaml
bricks:
- arduino:video_object_detection: {
    variables: {
      EI_OBJ_DETECTION_MODEL: /home/arduino/.arduino-bricks/ei-models/your-model.eim
    }
  }
```

### Changing the Detection Sensitivity

In [python/main.py](python/main.py), adjust two values:

- `CONFIDENCE_THRESHOLD` — minimum confidence to accept a gesture (0.0 to 1.0)
- `VideoObjectDetection(confidence=0.6, debounce_sec=0.0)` — the brick's own threshold

### Adding or Changing Gestures

The valid gestures are defined in `VALID_LABELS`:

```python
VALID_LABELS = {'rock', 'paper', 'scissors'}
```

The win logic is in `WINS`:

```python
WINS = {'Rock': 'Scissors', 'Scissors': 'Paper', 'Paper': 'Rock'}
```

To change gestures, update both of these and retrain your Edge Impulse model.

The UI emoji mapping is in `templates/index.html`:

```javascript
var EMOJIS = {
    Rock:     '\uD83E\uDEA8',
    Paper:    '\uD83D\uDCC4',
    Scissors: '\u2702\uFE0F'
};
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Web UI |
| `/api/state` | GET | Current game state (polled every 500ms) |
| `/api/play` | POST | Start a round (returns 409 if busy) |
| `/api/reset` | POST | Reset scores and history |

## Running Without Hardware

On a laptop (without the Arduino UNO Q), you can run the Flask server standalone:

```bash
cd python
pip install flask
python3 main.py
```

The game starts but detection won't work (the brick is unavailable). Every round will show "No gesture detected". This is useful for testing the UI.

## Troubleshooting

**"No gesture detected" every round:**
- Check that the brick is initialized: look for `[BRICK] VideoObjectDetection initialized` in logs
- Check that `App.run()` is active: look for `[MODE] App runner: yes` in logs
- Look for `[BRICK-RAW]` lines — if absent, the brick callback isn't firing
- Ensure your model labels match `rock`, `paper`, `scissors` (lowercase)

**"App runner: no" in logs:**
- The `App` class couldn't be imported. Make sure you're running via `arduino-app-cli app start`, not `python3 main.py` directly

**Model not found:**
- Verify the `.eim` file exists at the path in `app.yaml`
- Ensure the file is executable: `chmod +x /home/arduino/.arduino-bricks/ei-models/rcp-model.eim`

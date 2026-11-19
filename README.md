# Glucose Monitoring System

Final-year B.Tech project. An Arduino reads a PPG (pulse) signal and heart rate, sends them over USB serial, and a Flask app estimates the blood glucose level from those two readings with a Random Forest model. The dashboard refreshes with the latest reading and a short suggestion (low, normal or high).

> **Student prototype, not a medical device.** The model has not been clinically validated and this README makes no accuracy claims. Do not use its output for any health or treatment decision; use a certified glucometer.

## How it works

1. `preprocess_data.py` loads `dataset/data.csv` (PPG signal, heart rate, measured glucose), scales the two inputs with MinMaxScaler and writes train/test splits.
2. `extract_feature.py` adds two derived features: PPG / heart rate and PPG × heart rate.
3. `train_model.py` tunes a `RandomForestRegressor` with `RandomizedSearchCV` on a 10% sample, retrains on the full training set and saves `model/glucose_model.pkl`.
4. `predict.py` starts the Flask app, reads `*<ppg>*<hr>#` lines from the Arduino in a background thread, predicts glucose and serves the latest values at `/data`. The page at `/` polls it.
5. `append_data.py` adds new labelled rows to the dataset: it reads one sensor sample and asks you for the glucose value measured with a glucometer.

The suggestion uses fixed thresholds written into the code (below 80 mg/dL low, above 140 mg/dL high). They are placeholders for the demo, not medical guidance.

## Project layout

```
dataset/data.csv          training data (about 845k rows)
model/glucose_model.pkl   trained model
templates/index.html      dashboard
static/style.css
sensor_data.ini           last reading, written by predict.py
PORT                      last serial port used, written by append_data.py
```

## Running it

Requires Python 3.9+ and an Arduino sending data in the `*<ppg>*<hr>#` format at 9600 baud.

```bash
pip install -r requirements.txt

python preprocess_data.py
python extract_feature.py
python train_model.py      # only needed to retrain; a trained model is included

python predict.py          # then open http://127.0.0.1:5000
```

The serial port is detected automatically; if no Arduino is found it falls back to `COM3` (Windows). Set `FLASK_DEBUG=1` to run Flask in debug mode.

## Known limitation: prediction and scaling

The live predictions are not reliable in the current code, for this reason:

- `preprocess_data.py` scales `PPG_Signal` and `Heart_Rate` to the 0–1 range with `MinMaxScaler` before training, so the model only ever sees scaled inputs.
- The fitted scaler is not saved.
- `predict.py` builds its features from the raw sensor values (for example a PPG reading of 500) and passes them straight to the model.

The model is therefore asked about values far outside the range it was trained on, and its output for live readings should not be trusted. The fix is to save the scaler with the model, apply it in `predict.py` before computing the derived features, and retrain. That change has not been made yet.

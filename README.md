

# === backend/app.py ===

from flask import Flask, request, jsonify
from flask_cors import CORS
import requests

app = Flask(__name__)
CORS(app)

children_data = [
    {"id": 1, "age": 4, "disability": True, "malnutrition": False, "location_risk": "high"},
    {"id": 2, "age": 6, "disability": False, "malnutrition": True, "location_risk": "medium"},
    {"id": 3, "age": 2, "disability": False, "malnutrition": True, "location_risk": "high"},
    {"id": 4, "age": 9, "disability": True, "malnutrition": False, "location_risk": "low"},
]

def fetch_weather(lat, lon, api_key):
    url = f"https://api.openweathermap.org/data/2.5/weather?lat={lat}&lon={lon}&appid={api_key}&units=metric"
    res = requests.get(url)
    res.raise_for_status()
    return res.json()

def calculate_vulnerability_score(child):
    score = 0
    if child['age'] < 5: score += 2
    if child['disability']: score += 2
    if child['malnutrition']: score += 3
    if child['location_risk'] == 'high': score += 3
    elif child['location_risk'] == 'medium': score += 1
    return score

def assess_risks(weather, children):
    wind = weather.get('wind', {}).get('speed', 0)
    rain = weather.get('rain', {}).get('1h', 0)
    risks = []
    high_wind = wind > 10
    heavy_rain = rain > 20
    for child in children:
        score = calculate_vulnerability_score(child)
        if high_wind or heavy_rain:
            if score >= 5:
                risks.append({"id": child["id"], "risk": "High", "score": score})
            elif score >= 3:
                risks.append({"id": child["id"], "risk": "Medium", "score": score})
    return risks

@app.route("/weather-risk", methods=["POST"])
def get_risk():
    data = request.json
    lat = data.get("lat", 10.5)
    lon = data.get("lon", 7.4)
    api_key = data.get("api_key")
    try:
        weather = fetch_weather(lat, lon, api_key)
        risks = assess_risks(weather, children_data)
        return jsonify({
            "weather": {
                "wind_speed": weather.get('wind', {}).get('speed', 0),
                "rain_1h": weather.get('rain', {}).get('1h', 0)
            },
            "risks": risks
        })
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == "__main__":
    app.run(debug=True)


# === backend/requirements.txt ===
flask
flask-cors
requests


# === backend/README.md ===
# Flask Backend

Run with:
```bash
pip install -r requirements.txt
flask run
```

# === frontend/package.json ===

{
  "name": "weather-risk-frontend",
  "version": "1.0.0",
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "axios": "^1.4.0"
  },
  "scripts": {
    "start": "parcel public/index.html --port 3000"
  },
  "devDependencies": {
    "parcel": "^2.9.3"
  }
}


# === frontend/src/App.jsx ===

import React, { useState } from 'react';
import axios from 'axios';

const App = () => {
  const [wind, setWind] = useState(null);
  const [rain, setRain] = useState(null);
  const [risks, setRisks] = useState([]);

  const fetchData = async () => {
    const res = await axios.post("http://localhost:5000/weather-risk", {
      lat: 10.5,
      lon: 7.4,
      api_key: "YOUR_OPENWEATHER_API_KEY"
    });
    setWind(res.data.weather.wind_speed);
    setRain(res.data.weather.rain_1h);
    setRisks(res.data.risks);
  };

  return (
    <div style={{ padding: "2rem" }}>
      <h1>🌦️ Weather Risk Dashboard</h1>
      <button onClick={fetchData}>Fetch Risk Data</button>
      <h2>Weather</h2>
      <p>Wind Speed: {wind} m/s</p>
      <p>Rainfall: {rain} mm</p>
      <h2>Risks</h2>
      {risks.length === 0 ? <p>No risks.</p> : risks.map(r =>
        <div key={r.id} style={{
          padding: "1rem", margin: "0.5rem 0",
          background: r.risk === "High" ? "red" : "orange",
          color: "white"
        }}>
          Child ID {r.id}: {r.risk} Risk (Score: {r.score})
        </div>
      )}
    </div>
  );
};

export default App;


# === frontend/src/index.jsx ===

import React from "react";
import { createRoot } from "react-dom/client";
import App from "./App";
const root = createRoot(document.getElementById("root"));
root.render(<App />);


# === frontend/src/styles.css ===
body { font-family: sans-serif; background: #f5f5f5; }

# === frontend/public/index.html ===

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Weather Risk Dashboard</title>
</head>
<body>
  <div id="root"></div>
</body>
</html>

# Agricultural-Production-and-Transport-Management-System-IAPTMS-
Traditionally, agricultural production planning and transportation logistics operate as isolated systems. This information asymmetry leads to significant post-harvest losses, inflated fuel emissions, 




Open your terminal or command prompt inside your project folder and run these three quick commands:Start the database engines:bashdocker-compose up -d
Use code with caution.Install node dependencies and boot up your API:bashnpm install && npm start
Use code with caution.Open another terminal window and start your field hardware simulator:bashpython emulator.py






New-Item -ItemType Directory -Path "agrilogix-platform"; Set-Location "agrilogix-platform"

@'
version: '3.8'
services:
  postgres_db:
    image: postgres:15-alpine
    container_name: agrilogix_postgres
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret_agri_password
      POSTGRES_DB: agrilogix
    ports:
      - "5432:5432"
  influx_db:
    image: influxdb:2.7-alpine
    container_name: agrilogix_influx
    restart: always
    ports:
      - "8086:8086"
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=super_secret_influx_password
      - DOCKER_INFLUXDB_INIT_ORG=agrilogix-corp
      - DOCKER_INFLUXDB_INIT_BUCKET=telemetry_data
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=your-super-secret-auth-token
'@ | Out-File -Encoding UTF8 docker-compose.yml

@'
{
  "name": "agrilogix-api",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "cors": "^2.8.5",
    "express": "^4.18.2",
    "socket.io": "^4.6.1"
  },
  "scripts": {
    "start": "node server.js"
  }
}
'@ | Out-File -Encoding UTF8 package.json

@'
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const cors = require('cors');
const app = express();
app.use(cors());
app.use(express.json());
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: "*", methods: ["GET", "POST"] } });
io.on('connection', (socket) => { console.log(`🔌 Dashboard interface linked: ${socket.id}`); });
const MOISTURE_CRITICAL_LOW = 20.0;
app.post('/api/telemetry/ingest', (req, res) => {
    try {
        const { deviceId, type, metrics } = req.body;
        io.emit('telemetry_stream', { deviceId, type, metrics, time: new Date().toLocaleTimeString() });
        if (type === 'moisture' && metrics.moisture < MOISTURE_CRITICAL_LOW) {
            io.emit('system_alert', {
                timestamp: new Date().toLocaleTimeString(),
                module: 'Production',
                severity: 'Critical',
                message: `Field Sector [${deviceId}] moisture dropped to ${metrics.moisture}%! Direct irrigation needed.`
            });
        }
        res.status(202).json({ status: 'Data routed successfully.' });
    } catch (err) { res.status(500).json({ error: err.message }); }
});
server.listen(5000, () => console.log('🚀 AgriLogix backend pipeline active on port 5000'));
'@ | Out-File -Encoding UTF8 server.js

@'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>AgriLogix Live Hub</title>
    <script src="https://jsdelivr.net"></script>
    <link rel="stylesheet" href="https://unpkg.com" />
    <script src="https://unpkg.com"></script>
    <script src="https://jsdelivr.net"></script>
    <style>
        :root { --bg-dark: #121212; --bg-card: #1E1E1E; --text-main: #E0E0E0; }
        body { background: var(--bg-dark); color: var(--text-main); font-family: sans-serif; display: flex; height: 100vh; margin: 0; }
        aside { width: 240px; background: var(--bg-card); padding: 20px; border-right: 1px solid #333; }
        main { flex-grow: 1; padding: 30px; overflow-y: auto; }
        .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 20px; margin-bottom: 20px; }
        .card { background: var(--bg-card); padding: 20px; border-radius: 8px; border: 1px solid #333; }
        table { width: 100%; border-collapse: collapse; margin-top: 15px; color: #fff; }
        th, td { padding: 12px; text-align: left; border-bottom: 1px solid #333; }
        #map { height: 350px; border-radius: 8px; margin-top: 20px; }
        .badge-alert { background: #f44336; color: white; padding: 4px 8px; border-radius: 4px; font-size: 0.8rem; }
    </style>
</head>
<body>
    <aside><h2>🌾 AgriLogix</h2><p style="color:#888;">Real-Time Control Panel</p></aside>
    <main>
        <div class="grid">
            <div class="card"><h3>System Status</h3><h1 style="color:#4CAF50;">Operational</h1></div>
            <div class="card"><h3>Telemetry Channel</h3><h1 id="stream-count" style="color:#2196F3;">Listening...</h1></div>
        </div>
        <div class="card">
            <h3>Live Telemetry Alerts & Updates</h3>
            <table>
                <thead><tr><th>Time</th><th>Module</th><th>Severity</th><th>Notification Event Message</th></tr></thead>
                <tbody id="alert-zone"><tr><td colspan="4" style="color:#666; text-align:center;">Waiting for simulated farm sensors to start streaming...</td></tr></tbody>
            </table>
        </div>
        <div id="map"></div>
    </main>
    <script>
        const map = L.map('map').setView([20.5937, 78.9629], 5);
        L.tileLayer('https://{s}://{z}/{x}/{y}{r}.png').addTo(map);
        const markers = {};
        const socket = io('http://localhost:5000');
        socket.on('telemetry_stream', (data) => {
            document.getElementById('stream-count').innerText = `Active Link`;
            if (data.type === 'gps') {
                const { lat, lon } = data.metrics;
                if (markers[data.deviceId]) { markers[data.deviceId].setLatLng([lat, lon]); }
                else { markers[data.deviceId] = L.marker([lat, lon]).addTo(map).bindPopup(`<b>${data.deviceId}</b>`); }
            }
        });
        socket.on('system_alert', (alert) => {
            const tbody = document.getElementById('alert-zone');
            if(tbody.rows && tbody.rows.cells.length === 1) tbody.innerHTML = '';
            const row = tbody.insertRow(0);
            row.style.background = 'rgba(244, 67, 54, 0.1)';
            row.innerHTML = `<td>${alert.timestamp}</td><td>${alert.module}</td><td><span class="badge-alert">${alert.severity}</span></td><td>${alert.message}</td>`;
        });
    </script>
</body>
</html>
'@ | Out-File -Encoding UTF8 index.html

@'
import time, random, json, http.client
API_HOST, API_PORT = "localhost", 5000
devices = {
    "FIELD-SEC-03": {"type": "moisture", "base": 19.0},
    "TRK-088": {"type": "gps", "lat": 20.5937, "lon": 78.9629}
}
print("🚜 Starting Edge Device Transmitter Loop...")
while True:
    for dev_id, config in devices.items():
        payload = {"deviceId": dev_id, "type": config["type"], "metrics": {}}
        if config["type"] == "moisture":
            config["base"] += random.uniform(-0.8, 0.5)
            payload["metrics"] = {"moisture": round(max(0.0, config["base"]), 2)}
        elif config["type"] == "gps":
            config["lat"] += random.uniform(-0.02, 0.02)
            config["lon"] += random.uniform(-0.02, 0.02)
            payload["metrics"] = {"lat": config["lat"], "lon": config["lon"]}
        try:
            conn = http.client.HTTPConnection(API_HOST, API_PORT, timeout=1)
            conn.request("POST", "/api/telemetry/ingest", json.dumps(payload), {'Content-Type': 'application/json'})
            conn.getresponse().read()
            conn.close()
            print(f"📡 Sent data for {dev_id}")
        except Exception: print("❌ Backend server offline. Check node application.")
    time.sleep(2)
'@ | Out-File -Encoding UTF8 emulator.py

Write-Host "✅ Project built successfully inside folder: agrilogix-platform" -ForegroundColor Green

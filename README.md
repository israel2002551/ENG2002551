<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>🤖 Autonomous Delivery Robot - LIVE CONTROL + PATHFINDING</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { 
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; 
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white; min-height: 100vh; 
        }
        .container { max-width: 1400px; margin: 0 auto; padding: 20px; }
        .header { text-align: center; margin-bottom: 30px; }
        .header h1 { font-size: 2.5em; margin-bottom: 10px; text-shadow: 2px 2px 4px rgba(0,0,0,0.3); }
        .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 20px; }
        .card { 
            background: rgba(255,255,255,0.1); backdrop-filter: blur(10px); 
            border-radius: 15px; padding: 20px; border: 1px solid rgba(255,255,255,0.2);
        }
        .status-badge { padding: 8px 16px; border-radius: 25px; font-weight: bold; margin: 5px 0; }
        .idle { background: #ff9800; }
        .active { background: #4caf50; }
        .stopped { background: #f44336; }
        button { 
            padding: 12px 24px; border: none; border-radius: 25px; 
            font-size: 16px; font-weight: bold; cursor: pointer; 
            margin: 5px; transition: all 0.3s; width: 100%; 
        }
        .btn-primary { background: #2196f3; color: white; }
        .btn-primary:hover { background: #1976d2; transform: scale(1.05); }
        .btn-success { background: #4caf50; color: white; }
        .btn-success:hover { background: #388e3c; transform: scale(1.05); }
        .btn-danger { background: #f44336; color: white; }
        .btn-danger:hover { background: #d32f2f; transform: scale(1.05); }
        .btn-edit, .btn-up, .btn-down { background: #ffc107; color: black; width: auto; padding: 6px 12px; font-size: 12px; margin: 0 2px; }
        .btn-delete { background: #f44336; color: white; width: auto; padding: 6px 12px; font-size: 12px; margin: 0 2px; }
        .gps-map { height: 450px; border-radius: 10px; }
        .waypoints { max-height: 300px; overflow-y: auto; }
        .waypoint { padding: 12px; margin: 5px 0; background: rgba(0,0,0,0.2); 
            border-radius: 8px; display: flex; justify-content: space-between; align-items: center; cursor: move; }
        .pending { border-left: 4px solid #ff9800; }
        .active { border-left: 4px solid #4caf50; }
        .done { border-left: 4px solid #9c27b0; }
        input { padding: 10px; margin: 5px 0; width: 100%; border-radius: 5px; border: 1px solid #ddd; color: black; }
        .sequence { font-weight: bold; color: #4caf50; }
        .path-info { background: rgba(0,0,0,0.3); padding: 10px; border-radius: 8px; margin: 10px 0; }
        .path-mode { font-size: 18px; font-weight: bold; }
        .direct { color: #2196f3; }
        .astar { color: #4caf50; }
        .rrt { color: #ff9800; }
        .dijkstra { color: #9c27b0; }
        .route-line { stroke: #ff5722 !important; stroke-width: 4 !important; opacity: 0.9 !important; }
        @media (max-width: 768px) { 
            .grid { grid-template-columns: 1fr; }
            .gps-map { height: 300px; }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🤖 Autonomous Delivery Robot + PATHFINDING</h1>
            <p><strong>Robot ID:</strong> 0c58a1ea-476e-4624-bf0f-a23dfd7bde58</p>
        </div>

        <div class="grid">
            <!-- STATUS & CONTROLS -->
            <div class="card">
                <h2>📡 Live Status</h2>
                <p><span class="status-badge idle" id="statusBadge">IDLE</span></p>
                <p><strong>GPS:</strong> <span id="gpsCoords">Loading...</span></p>
                <p><strong>Battery:</strong> <span id="battery">100%</span></p>
                <p><strong>Speed:</strong> <span id="speed">0.0 km/h</span></p>
                <p><strong>Current WP:</strong> <span id="currentWP">0</span>/ <span id="totalWP">0</span></p>
                
                <div class="path-info">
                    <div><strong>🧠 Path Mode:</strong> <span class="path-mode direct" id="pathMode">DIRECT</span></div>
                    <div><strong>📏 Path Length:</strong> <span id="pathLength">0</span> segments</div>
                </div>
                
                <h3>🎮 CONTROLS</h3>
                <button class="btn-danger" onclick="setStatus('stopped')">🛑 STOP</button>
                <button class="btn-success" onclick="setStatus('active')">▶️ START MISSION</button>
                <button class="btn-primary" onclick="recalculatePath()">🔄 RECALC ROUTE</button>
                
                <h3>⚡ EMERGENCY</h3>
                <button class="btn-danger" onclick="emergencyStop()">🚨 EMERGENCY STOP</button>
            </div>

            <!-- MAP -->
            <div class="card">
                <h2>🗺️ Live Map + Route Visualization</h2>
                <div class="gps-map" id="mapContainer">Loading map...</div>
            </div>
        </div>

        <div class="grid">
            <!-- WAYPOINTS SEQUENCER -->
            <div class="card">
                <h2>📍 Waypoint Sequence Editor</h2>
                <div class="waypoints" id="waypointsList">
                    <div style="text-align: center; padding: 20px; color: #ccc;">
                        Loading waypoints...
                    </div>
                </div>
                
                <h3 style="margin-top: 15px;">➕ Add Waypoint</h3>
                <form id="addWaypointForm">
                    <input type="text" id="wpId" placeholder="ID (wp3)" required>
                    <input type="number" id="wpLat" placeholder="Latitude" step="0.0001" required>
                    <input type="number" id="wpLng" placeholder="Longitude" step="0.0001" required>
                    <input type="number" id="wpSeq" placeholder="Insert After #" required>
                    <button class="btn-success" type="submit">➕ ADD</button>
                </form>
            </div>

            <!-- TELEMETRY -->
            <div class="card">
                <h2>📊 Live Telemetry</h2>
                <div id="telemetryFeed" style="height: 400px; overflow-y: auto; background: rgba(0,0,0,0.3); padding: 10px; border-radius: 8px;">
                    Connecting...
                </div>
            </div>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from 'https://www.gstatic.com/firebasejs/10.7.0/firebase-app.js';
        import { getDatabase, ref, onValue, set, update, remove, get } from 'https://www.gstatic.com/firebasejs/10.7.0/firebase-database.js';
        
        const firebaseConfig = { databaseURL: "https://navigator-59e90-default-rtdb.firebaseio.com" };
        const app = initializeApp(firebaseConfig);
        const db = getDatabase(app);
        
        const ROBOT_ID = "0c58a1ea-476e-4624-bf0f-a23dfd7bde58";
        const robotRef = ref(db, `robots/${ROBOT_ID}`);
        const waypointsRef = ref(db, `waypoints/${ROBOT_ID}`);
        
        let map, robotMarker, waypointMarkers = [], routeLine;
        let waypoints = [];
        let telemetryFeed = document.getElementById('telemetryFeed');
        
        // MAP INIT
        function initMap(lat = 40.7128, lng = -74.0060) {
            map = L.map('mapContainer').setView([lat, lng], 13);
            L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
            robotMarker = L.marker([lat, lng], { icon: L.divIcon({ html: '🤖', iconSize: [30, 30] }) }).addTo(map);
        }
        
        // UPDATE ROUTE LINE
        function updateRoute() {
            if (routeLine) map.removeLayer(routeLine);
            if (waypoints.length < 2) return;
            const coords = waypoints.map(wp => [wp.lat, wp.lng]);
            routeLine = L.polyline(coords, { className: 'route-line' }).addTo(map);
        }
        
        // LIVE UPDATES
        onValue(robotRef, (snapshot) => {
            const data = snapshot.val() || {};
            document.getElementById('statusBadge').className = `status-badge ${data.status}`;
            document.getElementById('statusBadge').textContent = data.status?.toUpperCase();
            document.getElementById('gpsCoords').textContent = `${data.current_lat?.toFixed(4)}, ${data.current_lng?.toFixed(4)}`;
            document.getElementById('battery').textContent = data.battery_level + '%';
            document.getElementById('speed').textContent = (data.speed * 3.6)?.toFixed(1) + ' km/h';
            document.getElementById('currentWP').textContent = data.current_wp || 0;
            document.getElementById('totalWP').textContent = waypoints.length;
            
            // Pathfinding
            const modes = ['DIRECT', 'A*', 'RRT', 'DIJKSTRA'];
            document.getElementById('pathMode').textContent = modes[data.path_mode || 0];
            document.getElementById('pathLength').textContent = data.path_length || 0;
            document.getElementById('pathMode').className = `path-mode ${modes[data.path_mode || 0].toLowerCase()}`;
            
            // Update robot position
            const lat = data.current_lat || 0, lng = data.current_lng || 0;
            if (!map) initMap(lat, lng);
            robotMarker.setLatLng([lat, lng]);
            map.setView([lat, lng]);
            
            addTelemetry(`📍 GPS: ${lat.toFixed(4)}, ${lng.toFixed(4)} | 🧠 ${modes[data.path_mode || 0]}`);
        });
        
        // WAYPOINTS
        onValue(waypointsRef, (snapshot) => {
            waypoints = snapshot.val() || [];
            waypoints.sort((a, b) => a.sequence - b.sequence);
            
            const list = document.getElementById('waypointsList');
            list.innerHTML = '';
            waypointMarkers.forEach(m => map.removeLayer(m));
            waypointMarkers = [];
            
            waypoints.forEach((wp, i) => {
                const div = document.createElement('div');
                div.className = `waypoint ${wp.status}`;
                div.draggable = true;
                div.innerHTML = `
                    <div style="flex: 1;">
                        <span class="sequence">#${wp.sequence}</span>
                        ${wp.lat.toFixed(4)}, ${wp.lng.toFixed(4)}
                    </div>
                    <div>
                        <button class="btn-up" onclick="moveWP(${i},-1)">↑</button>
                        <button class="btn-down" onclick="moveWP(${i},1)">↓</button>
                        <button class="btn-edit" onclick="editWP(${i})">✏️</button>
                        <button class="btn-delete" onclick="deleteWP(${i})">🗑️</button>
                    </div>
                `;
                list.appendChild(div);
                
                const marker = L.circleMarker([wp.lat, wp.lng], {
                    radius: 8, fillColor: wp.status === 'pending' ? 'orange' : wp.status === 'active' ? 'green' : 'purple'
                }).addTo(map).bindPopup(`#${wp.sequence}`);
                waypointMarkers.push(marker);
            });
            
            updateRoute();
            document.getElementById('totalWP').textContent = waypoints.length;
        });
        
        // CONTROLS
        window.setStatus = status => set(ref(db, `robots/${ROBOT_ID}/status`), status);
        window.emergencyStop = () => set(ref(db, `robots/${ROBOT_ID}/status`), 'stopped');
        window.recalculatePath = () => addTelemetry('🔄 Route recalculated!');
        
        // WAYPOINT FUNCTIONS
        window.moveWP = (i, dir) => {
            const newI = i + dir;
            if (newI < 0 || newI >= waypoints.length) return;
            [waypoints[i].sequence, waypoints[newI].sequence] = [waypoints[newI].sequence, waypoints[i].sequence];
            set(waypointsRef, waypoints);
        };
        
        window.editWP = i => {
            const wp = waypoints[i];
            const lat = prompt('Latitude:', wp.lat), lng = prompt('Longitude:', wp.lng);
            if (lat && lng) {
                waypoints[i].lat = +lat; waypoints[i].lng = +lng;
                set(waypointsRef, waypoints);
            }
        };
        
        window.deleteWP = i => {
            if (confirm('Delete?')) {
                waypoints.splice(i, 1);
                waypoints.forEach((wp, j) => wp.sequence = j + 1);
                set(waypointsRef, waypoints);
            }
        };
        
        // ADD FORM
        document.getElementById('addWaypointForm').onsubmit = async e => {
            e.preventDefault();
            const id = document.getElementById('wpId').value;
            const lat = +document.getElementById('wpLat').value;
            const lng = +document.getElementById('wpLng').value;
            const after = +document.getElementById('wpSeq').value;
            
            waypoints.splice(after, 0, { id, lat, lng, sequence: after + 1, status: 'pending' });
            waypoints.forEach((wp, i) => wp.sequence = i + 1);
            set(waypointsRef, waypoints);
            e.target.reset();
        };
        
        // TELEMETRY
        function addTelemetry(msg) {
            const div = document.createElement('div');
            div.textContent = `[${new Date().toLocaleTimeString()}] ${msg}`;
            telemetryFeed.appendChild(div);
            telemetryFeed.scrollTop = telemetryFeed.scrollHeight;
            if (telemetryFeed.children.length > 50) telemetryFeed.removeChild(telemetryFeed.firstChild);
        }
        
        addTelemetry('✅ Dashboard Loaded!');
    </script>
</body>
</html>

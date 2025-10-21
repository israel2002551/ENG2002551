<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>🤖 Autonomous Delivery Robot - GitHub Integration</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
          integrity="sha256-p4NxAoJBhIIN+hmNHrzRCf9tD/miZyoHS5obTRR9BMY=" crossorigin=""/>
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
            integrity="sha256-20nQCchB9co0qIjJZRGuk2/Z9VM+kNiyxNV1lvTlZBo=" crossorigin=""></script>
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
        .status-badge { padding: 8px 16px; border-radius: 25px; font-weight: bold; display: inline-block; margin: 5px 0; }
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
        #mapContainer {
            height: 450px; width: 100%; border-radius: 10px; background: #f0f0f0;
            position: relative;
        }
        .waypoints { max-height: 300px; overflow-y: auto; }
        .waypoint {
            padding: 12px; margin: 5px 0; background: rgba(0,0,0,0.2);
            border-radius: 8px; display: flex; justify-content: space-between; align-items: center; cursor: move;
        }
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
        .route-line { stroke: #ff5722; stroke-width: 4; opacity: 0.9; }
        @media (max-width: 768px) {
            .grid { grid-template-columns: 1fr; }
            #mapContainer { height: 300px; }
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
                
                <h3 style="margin-top: 20px;">⚡ EMERGENCY</h3>
                <button class="btn-danger" onclick="emergencyStop()">🚨 EMERGENCY STOP</button>
            </div>

            <!-- MAP -->
            <div class="card">
                <h2>🗺️ Live Map + Route</h2>
                <div id="mapContainer">
                    <div style="position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); color: #666;">
                        🗺️ Loading Map...
                    </div>
                </div>
            </div>
        </div>

        <div class="grid">
            <!-- WAYPOINTS -->
            <div class="card">
                <h2>📍 Waypoint Sequence</h2>
                <div class="waypoints" id="waypointsList">
                    <div style="text-align: center; padding: 20px; color: #ccc;">
                        Loading waypoints...
                    </div>
                </div>
                
                <h3 style="margin-top: 15px;">➕ Add Waypoint</h3>
                <form id="addWaypointForm">
                    <input type="text" id="wpId" placeholder="ID (wp1)" required>
                    <input type="number" id="wpLat" placeholder="40.7128" step="0.0001" required>
                    <input type="number" id="wpLng" placeholder="-74.0060" step="0.0001" required>
                    <input type="number" id="wpSeq" placeholder="0" required>
                    <button class="btn-success" type="submit" style="width: auto; padding: 10px 20px;">➕ ADD</button>
                </form>
            </div>

            <!-- TELEMETRY -->
            <div class="card">
                <h2>📊 Live Telemetry</h2>
                <div id="telemetryFeed" style="height: 400px; overflow-y: auto; background: rgba(0,0,0,0.3); padding: 10px; border-radius: 8px; font-size: 12px;">
                    <div style="color: #ccc;">Connecting to robot...</div>
                </div>
            </div>
        </div>
    </div>

    <script>
        const GITHUB_WAYPOINTS_URL = "https://raw.githubusercontent.com/username/repo/main/waypoints.json";
        const GITHUB_TELEMETRY_URL = "https://your-endpoint.com/telemetry";
        const ROBOT_ID = "0c58a1ea-476e-4624-bf0f-a23dfd7bde58";
        
        let map, robotMarker, waypointMarkers = [], routeLine;
        let waypoints = [];
        let telemetryFeed = document.getElementById('telemetryFeed');
        
        // Initialize Map
        function initMap(lat = 40.7128, lng = -74.0060) {
            try {
                map = L.map('mapContainer').setView([lat, lng], 15);
                L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                    attribution: '© OpenStreetMap contributors'
                }).addTo(map);
                
                robotMarker = L.marker([lat, lng], {
                    icon: L.divIcon({
                        html: '<div style="font-size: 24px;">🤖</div>',
                        className: 'robot-icon',
                        iconSize: [30, 30],
                        iconAnchor: [15, 15]
                    })
                }).addTo(map).bindPopup('Robot Position');
                
                document.querySelector('#mapContainer > div').style.display = 'none';
                addTelemetry('🗺️ Map loaded!');
            } catch (error) {
                console.error('Map error:', error);
                addTelemetry('❌ Map failed to load');
            }
        }
        
        // Update Route
        function updateRoute() {
            if (routeLine) map.removeLayer(routeLine);
            if (waypoints.length < 2) return;
            
            const coords = waypoints.map(wp => [wp.lat, wp.lng]);
            routeLine = L.polyline(coords, {
                color: '#ff5722',
                weight: 4,
                opacity: 0.9
            }).addTo(map);
            
            map.fitBounds(routeLine.getBounds().pad(0.1));
        }
        
        // Fetch Data from GitHub
        async function fetchData() {
            try {
                const response = await fetch(GITHUB_WAYPOINTS_URL + `?t=${Date.now()}`);
                if (!response.ok) throw new Error('HTTP ' + response.status);
                const data = await response.json();
                
                // Update Status
                const statusBadge = document.getElementById('statusBadge');
                statusBadge.className = `status-badge ${data.status || 'idle'}`;
                statusBadge.textContent = (data.status || 'idle').toUpperCase();
                
                // Update Telemetry
                document.getElementById('gpsCoords').textContent =
                    `${(data.current_lat || 0).toFixed(4)}, ${(data.current_lng || 0).toFixed(4)}`;
                document.getElementById('battery').textContent = (data.battery_level || 100) + '%';
                document.getElementById('speed').textContent = ((data.speed || 0) * 3.6).toFixed(1) + ' km/h';
                document.getElementById('currentWP').textContent = data.current_wp || 0;
                
                // Pathfinding
                const modes = ['DIRECT', 'A*', 'RRT'];
                const mode = modes[data.path_mode || 0];
                document.getElementById('pathMode').textContent = mode;
                document.getElementById('pathMode').className = `path-mode ${mode.toLowerCase()}`;
                document.getElementById('pathLength').textContent = data.path_length || 0;
                document.getElementById('totalWP').textContent = data.waypoints ? data.waypoints.length : 0;
                
                // Update Map Position
                const lat = data.current_lat || 40.7128;
                const lng = data.current_lng || -74.0060;
                if (!map) initMap(lat, lng);
                else {
                    robotMarker.setLatLng([lat, lng]);
                    map.setView([lat, lng], 16);
                }
                
                // Update Waypoints
                waypoints = data.waypoints || [];
                waypoints.sort((a, b) => a.sequence - b.sequence);
                const list = document.getElementById('waypointsList');
                list.innerHTML = '';
                waypointMarkers.forEach(m => map?.removeLayer(m));
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
                    
                    if (map) {
                        const color = wp.status === 'pending' ? 'orange' :
                                     wp.status === 'active' ? 'green' : 'purple';
                        const marker = L.circleMarker([wp.lat, wp.lng], {
                            radius: 8, fillColor: color, color: 'white', weight: 2, fillOpacity: 0.8
                        }).addTo(map).bindPopup(`#${wp.sequence} ${wp.status}`);
                        waypointMarkers.push(marker);
                    }
                });
                
                updateRoute();
                addTelemetry(`📍 GPS: ${lat.toFixed(4)}, ${lng.toFixed(4)} | 🧠 ${mode}`);
            } catch (error) {
                console.error('Fetch error:', error);
                addTelemetry('❌ Failed to fetch data from GitHub');
            }
        }
        
        // Controls
        window.setStatus = async status => {
            try {
                const response = await fetch(GITHUB_TELEMETRY_URL, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ status, robot_id: ROBOT_ID })
                });
                if (!response.ok) throw new Error('HTTP ' + response.status);
                addTelemetry(`🎮 ${status.toUpperCase()} sent!`);
            } catch (error) {
                addTelemetry('❌ Failed to update status');
            }
        };
        
        window.emergencyStop = async () => {
            await setStatus('stopped');
            addTelemetry('🚨 EMERGENCY STOP!');
        };
        
        window.recalculatePath = () => {
            addTelemetry('🔄 Route recalc sent!');
            // Note: ESP32 handles recalculation internally
        };
        
        // Waypoint Functions
        window.moveWP = async (i, dir) => {
            const newI = i + dir;
            if (newI < 0 || newI >= waypoints.length) return;
            [waypoints[i].sequence, waypoints[newI].sequence] =
                [waypoints[newI].sequence, waypoints[i].sequence];
            await updateWaypoints();
            addTelemetry(`🔀 WP ${i+1} ↔ ${newI+1}`);
        };
        
        window.editWP = async i => {
            const wp = waypoints[i];
            const lat = prompt('Latitude:', wp.lat);
            const lng = prompt('Longitude:', wp.lng);
            if (lat && lng) {
                waypoints[i].lat = +lat;
                waypoints[i].lng = +lng;
                await updateWaypoints();
                addTelemetry(`✏️ WP #${wp.sequence} updated`);
            }
        };
        
        window.deleteWP = async i => {
            if (confirm('Delete WP?')) {
                waypoints.splice(i, 1);
                waypoints.forEach((wp, j) => wp.sequence = j + 1);
                await updateWaypoints();
                addTelemetry(`🗑️ WP #${i+1} deleted`);
            }
        };
        
        async function updateWaypoints() {
            try {
                const response = await fetch(GITHUB_TELEMETRY_URL, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ waypoints, robot_id: ROBOT_ID })
                });
                if (!response.ok) throw new Error('HTTP ' + response.status);
                fetchData(); // Refresh after update
            } catch (error) {
                addTelemetry('❌ Failed to update waypoints');
            }
        }
        
        // Add Waypoint
        document.getElementById('addWaypointForm').onsubmit = async e => {
            e.preventDefault();
            const id = document.getElementById('wpId').value;
            const lat = +document.getElementById('wpLat').value;
            const lng = +document.getElementById('wpLng').value;
            const after = +document.getElementById('wpSeq').value;
            
            waypoints.splice(after, 0, { id, lat, lng, sequence: after + 1, status: 'pending' });
            waypoints.forEach((wp, i) => wp.sequence = i + 1);
            await updateWaypoints();
            addTelemetry(`➕ WP ${id} added at #${after+1}`);
            e.target.reset();
        };
        
        // Telemetry
        function addTelemetry(msg) {
            const div = document.createElement('div');
            div.textContent = `[${new Date().toLocaleTimeString()}] ${msg}`;
            telemetryFeed.appendChild(div);
            telemetryFeed.scrollTop = telemetryFeed.scrollHeight;
            if (telemetryFeed.children.length > 50)
                telemetryFeed.removeChild(telemetryFeed.firstChild);
        }
        
        // Startup
        addTelemetry('✅ Dashboard loaded!');
        setTimeout(() => initMap(), 1000);
        fetchData();
        setInterval(fetchData, 5000); // Poll every 5 seconds
    </script>
</body>
</html>

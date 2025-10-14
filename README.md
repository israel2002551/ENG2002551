<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>ESP32 Mission Planner — Advanced</title>

<!-- Leaflet CSS -->
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>

<!-- Simple styles -->
<style>
  html,body { height:100%; margin:0; padding:0; }
  body { font-family: Arial, sans-serif; display:flex; flex-direction:column; }

  /* Login area */
  #authWrap { display:flex; align-items:center; justify-content:center; height:100vh; background: linear-gradient(#f7fbff,#eef6ff); }
  .authCard { background:white; padding:20px; width:360px; border-radius:8px; box-shadow:0 6px 20px rgba(0,0,0,0.08); }
  .authCard input { width:100%; padding:8px; margin:8px 0; border-radius:4px; border:1px solid #ddd; }

  /* App layout */
  #app { display:none; height:100vh; flex-direction:column; }
  #map { flex:1; min-height:360px; }
  #controls { padding:12px; background:#fafafa; border-top:1px solid #eee; display:flex; flex-direction:column; gap:10px; }

  .topRow { display:flex; justify-content:space-between; gap:12px; align-items:flex-start; }
  .leftCol { flex:1; display:flex; flex-direction:column; gap:8px; }
  .rightCol { width:320px; display:flex; flex-direction:column; gap:6px; }

  #robot-status-display { padding:8px; background:#e9f7ff; border-left:4px solid #2196F3; font-size:0.95em; border-radius:4px; }
  .status-text { margin-right:8px; display:inline-block; }

  .buttons { display:flex; gap:8px; flex-wrap:wrap; }
  button { padding:8px 12px; border-radius:6px; border:none; cursor:pointer; background:#1976d2; color:white; }
  button.ghost { background:#4caf50; }
  button.warn { background:#e53935; }

  textarea { width:100%; height:110px; padding:8px; border-radius:6px; border:1px solid #ddd; font-family:monospace; resize:vertical; }

  .progress-wrap { width:100%; background:#eee; height:14px; border-radius:8px; overflow:hidden; }
  .progress-bar { height:100%; width:0%; background:linear-gradient(90deg,#4caf50,#1e88e5); transition: width 400ms ease; }

  .small { font-size:0.9em; color:#444; }
  .muted { color:#666; font-size:0.85em; }

  .playback { display:flex; gap:6px; align-items:center; }
  .playback input[type="range"] { width:160px; }

  .arrowMarker { transform-origin:center; }
  .robotIcon div { transform-origin:center; }

  @media (max-width:900px){
    .topRow{flex-direction:column}
    .rightCol{width:100%}
  }
</style>
</head>
<body>

<!-- Authentication -->
<div id="authWrap">
  <div class="authCard" id="authCard">
    <h3 style="margin:0 0 8px 0">ESP32 Mission Planner — Sign in / Sign up</h3>

    <div id="authForms">
      <input id="emailField" type="email" placeholder="Email" autocomplete="username">
      <input id="passwordField" type="password" placeholder="Password" autocomplete="current-password">
      <div style="display:flex; gap:8px; margin-top:6px;">
        <button id="signInBtn">Sign In</button>
        <button id="signUpBtn" class="ghost">Sign Up</button>
      </div>
      <p class="muted" style="margin-top:8px">Uses Firebase Authentication (email/password). For production, secure your DB rules.</p>
    </div>

    <div id="authStatus" style="display:none; margin-top:8px;">
      <div class="small">Signed in as <span id="authEmail"></span></div>
      <div style="margin-top:8px;"><button id="signOutBtn" class="warn">Sign out</button></div>
    </div>
  </div>
</div>

<!-- App -->
<div id="app">
  <div id="map"></div>

  <div id="controls">
    <div class="topRow">
      <div class="leftCol">
        <div style="display:flex; justify-content:space-between; gap:8px; align-items:center;">
          <div>
            <h4 style="margin:0">Robot Status</h4>
            <div id="robot-status-display">No status yet.</div>
          </div>
          <div style="display:flex; gap:8px; align-items:center;">
            <label class="small">Follow</label>
            <input type="checkbox" id="followToggle" checked>
            <label class="small">Breadcrumb</label>
            <input type="checkbox" id="breadcrumbToggle" checked>
            <button id="logoutBtn" class="warn" style="margin-left:8px">Logout</button>
          </div>
        </div>

        <div style="margin-top:6px;">
          <div class="buttons">
            <button id="clearBtn">Clear Map Markers</button>
            <button id="uploadMissionBtn" class="ghost">Upload Mission</button>
            <button id="fetchMissionBtn" class="ghost">Fetch Mission</button>
            <button id="refreshConfigBtn">Refresh Config</button>
          </div>
        </div>

        <div style="margin-top:8px;">
          <h4 style="margin:6px 0">Mission JSON (editable)</h4>
          <textarea id="missionArea" placeholder='{"waypoints":[{"lat":6.52,"lon":3.37}]}'></textarea>
        </div>
      </div>

      <div class="rightCol">
        <div>
          <div class="small">Mission Progress</div>
          <div class="progress-wrap" style="margin-top:6px;">
            <div id="progressBar" class="progress-bar"></div>
          </div>
          <div id="progressText" class="small" style="margin-top:6px">0%</div>
        </div>

        <div style="margin-top:12px;">
          <div class="small">ETA</div>
          <div id="etaText" class="small">N/A</div>
        </div>

        <div style="margin-top:12px;">
          <div class="small">Playback</div>
          <div class="playback">
            <button id="playBtn">Play</button>
            <button id="pauseBtn">Pause</button>
            <button id="stepBackBtn">◀</button>
            <button id="stepFwdBtn">▶</button>
            <label class="small" style="margin-left:6px">Speed</label>
            <input id="playbackSpeed" type="range" min="0.25" max="4" step="0.25" value="1">
          </div>
        </div>

        <div style="margin-top:10px;">
          <div class="small">Turn-by-turn (arrows)</div>
          <div id="turnsContainer" class="muted" style="margin-top:6px">No mission</div>
        </div>
      </div>
    </div>
  </div>
</div>

<!-- Libraries -->
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@turf/turf@6/turf.min.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-auth.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>

<script>
/* ---------------- CONFIG - replace firebaseConfig with your project settings ---------------- */
const firebaseConfig = {
  apiKey: "AIzaSyADd3G_8WJFmiJmB2ewOYhs9IpuJRtTQ7A",
  authDomain: "navigator-59e90.firebaseapp.com",
  databaseURL: "https://navigator-59e90-default-rtdb.firebaseio.com",
  projectId: "navigator-59e90",
  storageBucket: "navigator-59e90.appspot.com",
  messagingSenderId: "677281499595",
  appId: "1:677281499595:web:f0bafebed198b2eece4e4e"
};
firebase.initializeApp(firebaseConfig);

/* ---------------- GLOBAL STATE ---------------- */
let map, missionRouteLayer, missionMarkersLayer, userMarkersLayer, breadcrumbLine, robotMarker = null, arrowLayer;
let robotRef = null;
let followRobot = true;
let showBreadcrumb = true;
let playbackInterval = null;
let playbackIndex = 0;
let playbackSpeed = 1.0;
let currentMission = null; // {waypoints: [{lat,lon}], mission_id}
let lastStatus = null;

/* ---------------- AUTH UI ---------------- */
const authWrap = document.getElementById('authWrap');
const authCard = document.getElementById('authCard');
const authForms = document.getElementById('authForms');
const authStatus = document.getElementById('authStatus');

const signInBtn = document.getElementById('signInBtn');
const signUpBtn = document.getElementById('signUpBtn');
const signOutBtn = document.getElementById('signOutBtn');
const emailField = document.getElementById('emailField');
const passwordField = document.getElementById('passwordField');
const authEmail = document.getElementById('authEmail');

signInBtn.addEventListener('click', async () => {
  const email = emailField.value.trim();
  const pwd = passwordField.value;
  if (!email || !pwd) return alert('Email & password required.');
  try {
    await firebase.auth().signInWithEmailAndPassword(email, pwd);
    // onAuthStateChanged will handle UI updates
  } catch(err) {
    alert('Sign in error: ' + err.message);
  }
});

signUpBtn.addEventListener('click', async () => {
  const email = emailField.value.trim();
  const pwd = passwordField.value;
  if (!email || !pwd) return alert('Email & password required.');
  try {
    await firebase.auth().createUserWithEmailAndPassword(email, pwd);
    alert('Account created. You are now signed in.');
  } catch(err) {
    alert('Sign up error: ' + err.message);
  }
});

signOutBtn.addEventListener('click', async () => {
  try {
    await firebase.auth().signOut();
  } catch(err) {
    console.error('Sign out error', err);
  }
});

/* On auth state changes, show/hide app */
firebase.auth().onAuthStateChanged(user => {
  if (user) {
    // signed in
    authForms.style.display = 'none';
    authStatus.style.display = 'block';
    authEmail.innerText = user.email || user.uid;
    // show app
    authWrap.style.display = 'none';
    document.getElementById('app').style.display = 'flex';
    startApp(); // initialize map & firebase listeners
  } else {
    // signed out
    authForms.style.display = 'block';
    authStatus.style.display = 'none';
    authWrap.style.display = 'flex';
    document.getElementById('app').style.display = 'none';
    stopApp();
  }
});

/* ---------------- START / STOP APP ---------------- */
function startApp(){
  // firebase db path for robot (customize if needed)
  robotRef = firebase.database().ref('robot1');

  if (!map) initMap();
  attachFirebaseListeners();
  setupUI();
}

function stopApp(){
  // detach db listeners
  if (robotRef) {
    try {
      robotRef.child('status').off();
      robotRef.child('mission').off();
      robotRef.child('config').off();
    } catch(e){}
  }
  // clear map layers
  if (missionMarkersLayer) missionMarkersLayer.clearLayers();
  if (userMarkersLayer) userMarkersLayer.clearLayers();
  if (missionRouteLayer) missionRouteLayer.setLatLngs([]);
  if (breadcrumbLine) breadcrumbLine.setLatLngs([]);
  if (arrowLayer) arrowLayer.clearLayers();
  if (robotMarker) { map.removeLayer(robotMarker); robotMarker = null; }
  // stop playback
  stopPlayback();
}

/* ---------------- MAP INIT ---------------- */
function initMap(){
  map = L.map('map').setView([6.5244,3.3792],15);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom:19, attribution:'© OpenStreetMap contributors'
  }).addTo(map);

  missionRouteLayer = L.polyline([], { color:'green', weight:3, dashArray:'6,6' }).addTo(map);
  missionMarkersLayer = L.layerGroup().addTo(map);
  userMarkersLayer = L.layerGroup().addTo(map);
  breadcrumbLine = L.polyline([], { color:'blue', weight:3 }).addTo(map);
  arrowLayer = L.layerGroup().addTo(map);

  // click to add user waypoint (draggable)
  map.on('click', (e) => {
    const m = L.marker(e.latlng, { draggable:true }).addTo(userMarkersLayer);
    m.on('dragend', updateMissionJsonFromUserMarkers);
    m.on('click', () => {
      if (confirm('Remove this waypoint?')) { userMarkersLayer.removeLayer(m); updateMissionJsonFromUserMarkers(); }
    });
    updateMissionJsonFromUserMarkers();
  });

  // ensure map paints fully once visible
  setTimeout(()=> map.invalidateSize(), 80);
}

/* ---------------- UI SETUP ---------------- */
function setupUI(){
  document.getElementById('logoutBtn').addEventListener('click', async () => {
    try { await firebase.auth().signOut(); } catch(e){ console.error(e); }
  });

  document.getElementById('clearBtn').addEventListener('click', () => {
    missionMarkersLayer.clearLayers();
    userMarkersLayer.clearLayers();
    missionRouteLayer.setLatLngs([]);
    breadcrumbLine.setLatLngs([]);
    arrowLayer.clearLayers();
    document.getElementById('missionArea').value = JSON.stringify({ waypoints: [] }, null, 2);
    setProgress(0);
    document.getElementById('turnsContainer').innerText = 'No mission';
    document.getElementById('etaText').innerText = 'N/A';
  });

  document.getElementById('uploadMissionBtn').addEventListener('click', () => {
    const txt = document.getElementById('missionArea').value;
    try {
      const mission = JSON.parse(txt);
      // basic validation
      if (!mission.waypoints || !Array.isArray(mission.waypoints)) return alert('Mission JSON must contain waypoints array.');
      robotRef.child('mission').set(mission).then(()=> alert('Mission uploaded.'));
    } catch(e) { alert('Invalid JSON: ' + e.message); }
  });

  document.getElementById('fetchMissionBtn').addEventListener('click', () => {
    robotRef.child('mission').once('value', snap => {
      if (!snap.exists()) alert('No mission found.');
      else {
        const mission = snap.val();
        renderMission(mission);
        alert('Mission fetched and rendered.');
      }
    });
  });

  document.getElementById('refreshConfigBtn').addEventListener('click', () => {
    robotRef.child('config').once('value').then(snap => {
      const cfg = snap.val();
      if (cfg) alert(`Config: BaseSpeed=${cfg.baseSpeed||'N/A'}, SafeDist=${cfg.safeDistance||'N/A'}`);
      else alert('No config');
    }).catch(e => console.error(e));
  });

  document.getElementById('followToggle').addEventListener('change', (e) => followRobot = e.target.checked);
  document.getElementById('breadcrumbToggle').addEventListener('change', (e) => {
    showBreadcrumb = e.target.checked;
    breadcrumbLine.setStyle({ opacity: showBreadcrumb ? 1 : 0 });
  });

  // Playback controls
  document.getElementById('playBtn').addEventListener('click', startPlayback);
  document.getElementById('pauseBtn').addEventListener('click', stopPlayback);
  document.getElementById('stepFwdBtn').addEventListener('click', () => stepPlayback(1));
  document.getElementById('stepBackBtn').addEventListener('click', () => stepPlayback(-1));
  document.getElementById('playbackSpeed').addEventListener('input', (e) => {
    playbackSpeed = parseFloat(e.target.value);
    if (playbackInterval) {
      // restart interval with new speed
      stopPlayback(); startPlayback();
    }
  });
}

/* ---------------- UTIL: mission JSON from user markers ---------------- */
function updateMissionJsonFromUserMarkers(){
  const wps = [];
  userMarkersLayer.eachLayer(l => {
    const p = l.getLatLng(); wps.push({ lat:+p.lat.toFixed(7), lon:+p.lng.toFixed(7) });
  });
  const mission = { mission_id:'UI-'+Date.now(), waypoints:wps };
  document.getElementById('missionArea').value = JSON.stringify(mission, null, 2);
}

/* ---------------- FIREBASE LISTENERS ---------------- */
function attachFirebaseListeners(){
  if (!robotRef) return;

  // mission listener
  robotRef.child('mission').on('value', snap => {
    const mission = snap.val();
    currentMission = mission || null;
    renderMission(mission);
  });

  // status listener (robot gps)
  robotRef.child('status').on('value', snap => {
    const status = snap.val();
    lastStatus = status || null;
    updateStatusUI(status);
    if (status && typeof status.lat === 'number' && typeof status.lng === 'number') {
      handleRobotMovement(status);
    }
  });
}

/* ---------------- RENDER MISSION ---------------- */
function renderMission(mission) {
  missionMarkersLayer.clearLayers();
  arrowLayer.clearLayers();
  missionRouteLayer.setLatLngs([]);
  breadcrumbLine.setLatLngs([]);
  setProgress(0);
  document.getElementById('turnsContainer').innerText = 'No mission';
  document.getElementById('etaText').innerText = 'N/A';

  if (!mission || !Array.isArray(mission.waypoints) || mission.waypoints.length === 0) {
    document.getElementById('missionArea').value = JSON.stringify({ waypoints: [] }, null, 2);
    return;
  }

  const latlngs = mission.waypoints.map(wp => [wp.lat, wp.lon]);
  missionRouteLayer.setLatLngs(latlngs);
  try { map.fitBounds(missionRouteLayer.getBounds(), { padding:[40,40] }); } catch(e){}

  // add markers and turns (simple arrows at each segment)
  const turns = [];
  for (let i=0;i<latlngs.length;i++){
    const ll = latlngs[i];
    L.marker(ll, { draggable:false }).addTo(missionMarkersLayer).bindPopup('Waypoint '+(i+1));
    if (i < latlngs.length-1) {
      // compute bearing for arrow
      const a = latlngs[i], b = latlngs[i+1];
      const bearing = turf.bearing(turf.point([a[1], a[0]]), turf.point([b[1], b[0]])); // turf uses [lon,lat]
      // arrow as rotated divIcon
      const arrow = L.marker([(a[0]+b[0])/2, (a[1]+b[1])/2], {
        icon: L.divIcon({
          className: 'arrowMarker',
          html: `<div style="transform: rotate(${bearing}deg); font-size:18px; opacity:0.9;">➤</div>`,
          iconSize: [24,24], iconAnchor:[12,12]
        }),
        interactive:false
      }).addTo(arrowLayer);
      turns.push(`Segment ${i+1} → ${i+2}: bearing ${bearing.toFixed(1)}°`);
    }
  }

  document.getElementById('turnsContainer').innerText = turns.join('\n') || 'No turns';
  document.getElementById('missionArea').value = JSON.stringify(mission, null, 2);

  // reset playback index
  playbackIndex = 0;
}

/* ---------------- UPDATE STATUS UI ---------------- */
function updateStatusUI(status) {
  const display = document.getElementById('robot-status-display');
  if (!status) { display.innerText = 'Robot status not available.'; return; }
  const lat = typeof status.lat === 'number' ? status.lat.toFixed(6) : 'N/A';
  const lng = typeof status.lng === 'number' ? status.lng.toFixed(6) : 'N/A';
  const heading = typeof status.heading === 'number' ? status.heading.toFixed(1)+'°' : 'N/A';
  const battery = typeof status.battery === 'number' ? status.battery+'%' : 'N/A';
  const speed = typeof status.speed === 'number' ? status.speed.toFixed(2)+' m/s' : 'N/A';
  const ultrasonic = typeof status.ultrasonic === 'number' ? status.ultrasonic.toFixed(1)+' cm' : 'N/A';
  const ts = status.ts ? new Date(status.ts*1000).toLocaleTimeString() : new Date().toLocaleTimeString();

  display.innerHTML = `
    <span class="status-text">Lat: ${lat}</span>
    <span class="status-text">Lng: ${lng}</span>
    <span class="status-text">Head: ${heading}</span>
    <span class="status-text">Bat: ${battery}</span>
    <span class="status-text">Speed: ${speed}</span>
    <span class="status-text">US: ${ultrasonic}</span>
    <span class="status-text">Updated: ${ts}</span>
  `;
}

/* ---------------- HANDLE ROBOT MOVEMENT ---------------- */
function handleRobotMovement(status) {
  const lat = status.lat, lng = status.lng;
  const heading = status.heading;
  const speed = typeof status.speed === 'number' ? status.speed : null;

  const dest = L.latLng(lat, lng);
  // place or move robot marker, rotate by heading
  if (!robotMarker) {
    robotMarker = L.marker(dest, {
      icon: L.divIcon({ html:`<div style="font-size:26px; transform: rotate(${heading||0}deg)">🤖</div>`, className:'robotIcon' }),
      interactive:false
    }).addTo(map);
    if (followRobot) map.panTo(dest);
  } else {
    animateRobotTo(dest, heading);
  }

  // add breadcrumb
  if (showBreadcrumb) breadcrumbLine.addLatLng(dest);

  // update progress if mission exists
  if (currentMission && Array.isArray(currentMission.waypoints) && currentMission.waypoints.length >= 2) {
    const percent = computeProgressPercent({ lat, lng }, currentMission.waypoints);
    setProgress(percent);

    // compute ETA
    const remainingMeters = computeRemainingDistanceMeters({ lat, lng }, currentMission.waypoints);
    if (speed && speed > 0) {
      const seconds = remainingMeters / speed;
      const eta = new Date(Date.now() + seconds*1000);
      document.getElementById('etaText').innerText = `${formatMeters(remainingMeters)} — ETA ${eta.toLocaleTimeString()}`;
    } else {
      document.getElementById('etaText').innerText = `${formatMeters(remainingMeters)} — ETA N/A (speed unknown)`;
    }
  }
}

/* Smooth animation to new position and rotate to heading */
function animateRobotTo(destLatLng, heading) {
  const dest = L.latLng(destLatLng.lat, destLatLng.lng);
  if (!robotMarker) { robotMarker = L.marker(dest).addTo(map); return; }
  const start = robotMarker.getLatLng();
  const frames = 18;
  const duration = Math.max(300, Math.min(1200, 800 / playbackSpeed)); // ms, adjusted by playbackSpeed
  const stepMs = duration / frames;
  let i = 0;
  const latStep = (dest.lat - start.lat) / frames;
  const lngStep = (dest.lng - start.lng) / frames;

  const anim = setInterval(() => {
    i++;
    const next = L.latLng(start.lat + latStep*i, start.lng + lngStep*i);
    robotMarker.setLatLng(next);
    // rotate icon by replacing inner HTML transform
    const el = robotMarker.getElement();
    if (el) {
      const div = el.querySelector('div');
      if (div && typeof heading === 'number') div.style.transform = `rotate(${heading}deg)`;
    }
    if (showBreadcrumb) breadcrumbLine.addLatLng(next);
    if (followRobot) map.panTo(next, { animate:false });
    if (i >= frames) clearInterval(anim);
  }, stepMs);
}

/* ---------------- PROGRESS / DISTANCE (Turf.js geodesic) ---------------- */
function computeProgressPercent(robotLatLng, waypoints) {
  if (!robotLatLng || !waypoints || waypoints.length < 2) return 0;
  try {
    // turf expects [lon,lat]
    const line = turf.lineString(waypoints.map(w => [w.lon, w.lat]));
    const pt = turf.point([robotLatLng.lng, robotLatLng.lat]);
    const snapped = turf.nearestPointOnLine(line, pt, {units:'kilometers'});
    // distance from start to snapped point
    const startPoint = turf.point(line.geometry.coordinates[0]);
    const sliceToSnapped = turf.lineSlice(startPoint, snapped, line);
    const distToSnappedKm = turf.length(sliceToSnapped, {units:'kilometers'}) || 0;
    const totalKm = turf.length(line, {units:'kilometers'}) || 0;
    if (totalKm === 0) return 0;
    return (distToSnappedKm / totalKm) * 100;
  } catch(e) { console.error('computeProgressPercent error', e); return 0; }
}

/* Compute remaining distance in meters: distance from snapped point to end of route */
function computeRemainingDistanceMeters(robotLatLng, waypoints) {
  if (!robotLatLng || !waypoints || waypoints.length < 2) return 0;
  try {
    const line = turf.lineString(waypoints.map(w => [w.lon, w.lat]));
    const pt = turf.point([robotLatLng.lng, robotLatLng.lat]);
    const snapped = turf.nearestPointOnLine(line, pt, {units:'kilometers'});
    const endPt = turf.point(line.geometry.coordinates[line.geometry.coordinates.length-1]);
    const slice = turf.lineSlice(snapped, endPt, line);
    const km = turf.length(slice, {units:'kilometers'});
    return Math.round(km * 1000);
  } catch(e) { console.error('computeRemainingDistanceMeters error', e); return 0; }
}

/* ---------------- ETA + formatting ---------------- */
function formatMeters(m) {
  if (m >= 1000) return (m/1000).toFixed(2)+' km';
  return Math.round(m) + ' m';
}

function setProgress(percent){
  const p = Math.max(0, Math.min(100, percent));
  document.getElementById('progressBar').style.width = p + '%';
  document.getElementById('progressText').innerText = p.toFixed(1) + '%';
}

/* ---------------- PLAYBACK (simulate moving along mission route) ---------------- */
function startPlayback(){
  if (!currentMission || !Array.isArray(currentMission.waypoints) || currentMission.waypoints.length < 2) {
    return alert('Load a mission first to use playback.');
  }
  stopPlayback();
  const coords = currentMission.waypoints.map(w => [w.lat, w.lon]);
  // flatten points into array of points to step through
  const route = [];
  for (let i=0;i<coords.length-1;i++){
    const a = coords[i], b = coords[i+1];
    const seg = turf.lineString([[a[1],a[0]],[b[1],b[0]]]);
    const segKm = turf.length(seg, {units:'kilometers'});
    const points = Math.max(6, Math.round(segKm * 100)); // ~100 pts per km
    for (let t=0;t<=points;t++){
      const frac = t / points;
      const interp = turf.along(seg, segKm * frac, {units:'kilometers'});
      route.push([interp.geometry.coordinates[1], interp.geometry.coordinates[0]]);
    }
  }
  // playback at interval; speed multiplier affects delay
  let idx = playbackIndex || 0;
  const baseDelay = 400; // ms between steps at speed=1
  playbackInterval = setInterval(() => {
    if (idx >= route.length) { stopPlayback(); return; }
    const p = route[idx];
    // simulate robot marker movement and status updates
    animateRobotTo({lat:p[0], lng:p[1]}, lastStatus && lastStatus.heading ? lastStatus.heading : null);
    setProgress(computeProgressPercent({lat:p[0], lng:p[1]}, currentMission.waypoints));
    idx++;
    playbackIndex = idx;
  }, Math.max(50, baseDelay / playbackSpeed));
}

function stopPlayback(){
  if (playbackInterval) { clearInterval(playbackInterval); playbackInterval = null; }
}

function stepPlayback(step){
  if (!currentMission || !Array.isArray(currentMission.waypoints) || currentMission.waypoints.length < 2) return;
  // create route snapshot as in startPlayback but without interval
  const coords = currentMission.waypoints.map(w => [w.lat, w.lon]);
  const route = [];
  for (let i=0;i<coords.length-1;i++){
    const a = coords[i], b = coords[i+1];
    const seg = turf.lineString([[a[1],a[0]],[b[1],b[0]]]);
    const segKm = turf.length(seg, {units:'kilometers'});
    const points = Math.max(6, Math.round(segKm * 100));
    for (let t=0;t<=points;t++){
      const frac = t / points;
      const interp = turf.along(seg, segKm * frac, {units:'kilometers'});
      route.push([interp.geometry.coordinates[1], interp.geometry.coordinates[0]]);
    }
  }
  if (route.length === 0) return;
  playbackIndex = (playbackIndex || 0) + step;
  if (playbackIndex < 0) playbackIndex = 0;
  if (playbackIndex >= route.length) playbackIndex = route.length - 1;
  const p = route[playbackIndex];
  animateRobotTo({lat:p[0], lng:p[1]}, lastStatus && lastStatus.heading ? lastStatus.heading : null);
  setProgress(computeProgressPercent({lat:p[0], lng:p[1]}, currentMission.waypoints));
}

/* ---------------- INITIAL UI state ---------------- */
document.getElementById('etaText').innerText = 'N/A';
document.getElementById('turnsContainer').innerText = 'No mission';

/* ---------------- Notes for production ---------------- */
/*
 - Use Firebase Auth and Realtime Database rules to restrict access.
 - Replace robotRef path if needed (e.g., use per-user robot nodes).
 - For very large missions consider server-side pre-processing or vector tiles.
*/

</script>
</body>
</html>     

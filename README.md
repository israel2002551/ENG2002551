<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>ESP32 Mission Planner — Advanced (Improved)</title>

<!-- Leaflet CSS -->
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>

<!-- Simple styles (improved accessibility + responsive) -->
<style>
  :root{ --primary:#1976d2; --accent:#4caf50; --danger:#e53935; --muted:#666; --card-bg:#fff; }
  html,body { height:100%; margin:0; padding:0; }
  body { font-family: system-ui,-apple-system,Segoe UI,Roboto,Arial; display:flex; flex-direction:column; background:#f1f6fb; color:#111; }

  /* Layout */
  #authWrap, #app { height:100vh; display:flex; align-items:center; justify-content:center; }
  .card { background:var(--card-bg); padding:18px; width:min(960px,96vw); border-radius:10px; box-shadow:0 8px 28px rgba(15,23,42,0.06); }

  /* Auth area */
  #authInner{ display:flex; gap:18px; align-items:center; }
  .authCol{ width:360px; }
  .authCard h3{ margin:0 0 8px 0 }
  .authCard input{ width:100%; padding:10px; margin:8px 0; border-radius:6px; border:1px solid #e6eefb; }
  .muted { color:var(--muted); }

  /* App layout */
  #appLayout{ display:grid; grid-template-columns: 1fr 360px; gap:14px; align-items:start; }
  #map{ height:64vh; min-height:420px; border-radius:8px; overflow:hidden; }
  #controls{ padding:12px; background:#fff; border-radius:8px; border:1px solid #eef6ff; }

  .topRow{ display:flex; justify-content:space-between; gap:12px; }
  .leftCol{ flex:1; display:flex; flex-direction:column; gap:8px; }
  .rightCol{ width:320px; display:flex; flex-direction:column; gap:10px; }

  .statusBox{ padding:10px; background:#eaf6ff; border-left:4px solid var(--primary); border-radius:6px; }
  .buttons{ display:flex; gap:8px; flex-wrap:wrap; }
  button{ padding:8px 12px; border-radius:8px; border:0; cursor:pointer; background:var(--primary); color:white; font-weight:600; }
  button.ghost{ background:var(--accent); }
  button.warn{ background:var(--danger); }
  textarea{ width:100%; height:120px; padding:10px; border-radius:8px; border:1px solid #e6eefb; font-family:monospace; resize:vertical; }

  .progress-wrap{ width:100%; background:#f0f4f8; height:14px; border-radius:8px; overflow:hidden; }
  .progress-bar{ height:100%; width:0%; background:linear-gradient(90deg,var(--accent),var(--primary)); transition: width 400ms ease; }

  .small{ font-size:0.92em; color:#333; }
  .playback{ display:flex; gap:6px; align-items:center; }

  /* responsive */
  @media (max-width:980px){
    #appLayout{ grid-template-columns: 1fr; }
    #map{ min-height:52vh; }
    .rightCol{ width:100%; }
  }

  /* accessible focus styles */
  button:focus, input:focus, textarea:focus{ outline:3px solid rgba(25,118,210,0.18); outline-offset:2px; }
</style>
</head>
<body>

<!-- Authentication -->
<div id="authWrap">
  <div class="card authCard" id="authCard">
    <div id="authInner">
      <div class="authCol">
        <h3>ESP32 Mission Planner — Sign in / Sign up</h3>
        <div id="authForms" aria-live="polite">
          <label for="emailField" class="small">Email</label>
          <input id="emailField" type="email" placeholder="Email" autocomplete="username" required>
          <label for="passwordField" class="small">Password</label>
          <input id="passwordField" type="password" placeholder="Password" autocomplete="current-password" required>
          <div style="display:flex; gap:8px; margin-top:8px;">
            <button id="signInBtn">Sign In</button>
            <button id="signUpBtn" class="ghost">Sign Up</button>
          </div>
          <p class="muted" style="margin-top:8px">Uses Firebase Authentication (email/password). For production, secure your DB rules.</p>
        </div>
        <div id="authStatus" style="display:none; margin-top:8px;" aria-live="polite">
          <div class="small">Signed in as <strong id="authEmail"></strong></div>
          <div style="margin-top:8px;"><button id="signOutBtn" class="warn">Sign out</button></div>
        </div>
      </div>
      <div style="flex:1; display:flex; align-items:center; justify-content:center;">
        <img src="https://upload.wikimedia.org/wikipedia/commons/3/3d/Robot_icon.svg" alt="robot illustration" style="width:140px; opacity:0.9;">
      </div>
    </div>
  </div>
</div>

<!-- App -->
<div id="app" style="display:none;">
  <div class="card" style="width:98%;">
    <div id="appLayout">
      <div>
        <div id="map"></div>
      </div>
      <div id="controls">
        <div class="topRow">
          <div class="leftCol">
            <div style="display:flex; justify-content:space-between; align-items:flex-start; gap:8px;">
              <div>
                <h4 style="margin:0">Robot Status</h4>
                <div id="robot-status-display" class="statusBox">No status yet.</div>
              </div>
              <div style="display:flex; gap:8px; align-items:center;">
                <label class="small" for="followToggle">Follow</label>
                <input type="checkbox" id="followToggle" checked aria-label="Follow robot on map">
                <label class="small" for="breadcrumbToggle">Breadcrumb</label>
                <input type="checkbox" id="breadcrumbToggle" checked aria-label="Show breadcrumb trail">
                <button id="logoutBtn" class="warn" style="margin-left:8px">Logout</button>
              </div>
            </div>

            <div style="margin-top:10px;">
              <div class="buttons" role="group" aria-label="Mission actions">
                <button id="clearBtn">Clear Map Markers</button>
                <button id="uploadMissionBtn" class="ghost">Upload Mission</button>
                <button id="fetchMissionBtn" class="ghost">Fetch Mission</button>
                <button id="refreshConfigBtn">Refresh Config</button>
              </div>
            </div>

            <div style="margin-top:12px;">
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

            <div>
              <div class="small">ETA</div>
              <div id="etaText" class="small">N/A</div>
            </div>

            <div>
              <div class="small">Playback</div>
              <div class="playback">
                <button id="playBtn">Play</button>
                <button id="pauseBtn">Pause</button>
                <button id="stepBackBtn" aria-label="step back">◀</button>
                <button id="stepFwdBtn" aria-label="step forward">▶</button>
                <label class="small" style="margin-left:6px">Speed</label>
                <input id="playbackSpeed" type="range" min="0.25" max="4" step="0.25" value="1" aria-label="playback speed">
              </div>
            </div>

            <div>
              <div class="small">Turn-by-turn (arrows)</div>
              <pre id="turnsContainer" class="muted" style="margin-top:6px; white-space:pre-wrap;">No mission</pre>
            </div>
          </div>
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

/* ---------------- GLOBAL STATE (wrapped) ---------------- */
const App = (function(){
  // DOM refs
  const els = {
    authWrap: document.getElementById('authWrap'),
    authForms: document.getElementById('authForms'),
    authStatus: document.getElementById('authStatus'),
    authEmail: document.getElementById('authEmail'),
    signInBtn: document.getElementById('signInBtn'),
    signUpBtn: document.getElementById('signUpBtn'),
    signOutBtn: document.getElementById('signOutBtn'),
    emailField: document.getElementById('emailField'),
    passwordField: document.getElementById('passwordField'),
    appRoot: document.getElementById('app'),
    mapEl: document.getElementById('map'),
    missionArea: document.getElementById('missionArea'),
    progressBar: document.getElementById('progressBar'),
    progressText: document.getElementById('progressText'),
    robotStatusDisplay: document.getElementById('robot-status-display'),
    etaText: document.getElementById('etaText'),
    turnsContainer: document.getElementById('turnsContainer'),
    followToggle: document.getElementById('followToggle'),
    breadcrumbToggle: document.getElementById('breadcrumbToggle'),
    playbackSpeed: document.getElementById('playbackSpeed'),
    playBtn: document.getElementById('playBtn'),
    pauseBtn: document.getElementById('pauseBtn'),
    stepBackBtn: document.getElementById('stepBackBtn'),
    stepFwdBtn: document.getElementById('stepFwdBtn'),
    uploadMissionBtn: document.getElementById('uploadMissionBtn'),
    fetchMissionBtn: document.getElementById('fetchMissionBtn'),
    clearBtn: document.getElementById('clearBtn'),
    refreshConfigBtn: document.getElementById('refreshConfigBtn'),
    logoutBtn: document.getElementById('logoutBtn')
  };

  // map & layers
  let map, missionRouteLayer, missionMarkersLayer, userMarkersLayer, breadcrumbLine, robotMarker = null, arrowLayer;
  let robotRef = null;

  // state
  let followRobot = true;
  let showBreadcrumb = true;
  let playbackInterval = null;
  let playbackIndex = 0;
  let playbackSpeed = 1.0;
  let currentMission = null;
  let lastStatus = null;

  // throttles
  const STATUS_THROTTLE_MS = 200; // limit UI updates
  let lastStatusUpdate = 0;

  /* ---------------- AUTH ---------------- */
  function setupAuth(){
    // Sign in
    els.signInBtn.addEventListener('click', async () => {
      const email = els.emailField.value.trim();
      const pwd = els.passwordField.value;
      if (!email || !pwd) return alert('Email & password required.');
      try { await firebase.auth().signInWithEmailAndPassword(email, pwd); }
      catch(err){ alert('Sign in error: ' + err.message); }
    });

    // Sign up
    els.signUpBtn.addEventListener('click', async () => {
      const email = els.emailField.value.trim();
      const pwd = els.passwordField.value;
      if (!email || !pwd) return alert('Email & password required.');
      try { await firebase.auth().createUserWithEmailAndPassword(email, pwd); alert('Account created. You are now signed in.'); }
      catch(err){ alert('Sign up error: ' + err.message); }
    });

    els.signOutBtn.addEventListener('click', async () => {
      try{ await firebase.auth().signOut(); } catch(e){ console.error('Sign out error', e); }
    });

    // on auth change
    firebase.auth().onAuthStateChanged(user => {
      if (user) onSignedIn(user); else onSignedOut();
    });
  }

  function onSignedIn(user){
    els.authForms.style.display = 'none';
    els.authStatus.style.display = 'block';
    els.authEmail.textContent = user.email || user.uid;
    els.authWrap.style.display = 'none';
    els.appRoot.style.display = 'flex';
    startApp();
  }
  function onSignedOut(){
    els.authForms.style.display = 'block';
    els.authStatus.style.display = 'none';
    els.authWrap.style.display = 'flex';
    els.appRoot.style.display = 'none';
    stopApp();
  }

  /* ---------------- MAP INIT ---------------- */
  function initMap(){
    map = L.map(els.mapEl).setView([6.5244,3.3792],15);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { maxZoom:19, attribution:'© OpenStreetMap contributors' }).addTo(map);

    missionRouteLayer = L.polyline([], { color:'green', weight:3, dashArray:'6,6' }).addTo(map);
    missionMarkersLayer = L.layerGroup().addTo(map);
    userMarkersLayer = L.layerGroup().addTo(map);
    breadcrumbLine = L.polyline([], { color:'blue', weight:3 }).addTo(map);
    arrowLayer = L.layerGroup().addTo(map);

    // add click handler with debounce to prevent accidental rapid markers
    let clickTimeout = null;
    map.on('click', (e)=>{
      if (clickTimeout) return; // simple rate limit
      clickTimeout = setTimeout(()=> clickTimeout = null, 200);
      const m = L.marker(e.latlng, { draggable:true }).addTo(userMarkersLayer);
      m.on('dragend', updateMissionJsonFromUserMarkers);
      m.on('click', () => { if (confirm('Remove this waypoint?')) { userMarkersLayer.removeLayer(m); updateMissionJsonFromUserMarkers(); } });
      updateMissionJsonFromUserMarkers();
    });

    // ensure map paints fully once visible
    setTimeout(()=> map.invalidateSize(), 80);
  }

  /* ---------------- START/STOP APP ---------------- */
  function startApp(){
    robotRef = firebase.database().ref('robot1');
    if (!map) initMap();
    attachFirebaseListeners();
    setupUI();
  }

  function stopApp(){
    if (robotRef) {
      try { robotRef.child('status').off(); robotRef.child('mission').off(); robotRef.child('config').off(); } catch(e){}
    }
    if (missionMarkersLayer) missionMarkersLayer.clearLayers();
    if (userMarkersLayer) userMarkersLayer.clearLayers();
    if (missionRouteLayer) missionRouteLayer.setLatLngs([]);
    if (breadcrumbLine) breadcrumbLine.setLatLngs([]);
    if (arrowLayer) arrowLayer.clearLayers();
    if (robotMarker) { map.removeLayer(robotMarker); robotMarker = null; }
    stopPlayback();
  }

  /* ---------------- UI SETUP ---------------- */
  function setupUI(){
    // logout
    els.logoutBtn.addEventListener('click', async ()=>{ try{ await firebase.auth().signOut(); } catch(e){ console.error(e); } });

    // core actions
    els.clearBtn.addEventListener('click', clearMapAndMission);
    els.uploadMissionBtn.addEventListener('click', uploadMission);
    els.fetchMissionBtn.addEventListener('click', fetchMission);
    els.refreshConfigBtn.addEventListener('click', refreshConfig);

    els.followToggle.addEventListener('change', e => followRobot = e.target.checked);
    els.breadcrumbToggle.addEventListener('change', e => { showBreadcrumb = e.target.checked; breadcrumbLine.setStyle({ opacity: showBreadcrumb ? 1 : 0 }); });

    // playback
    els.playBtn.addEventListener('click', startPlayback);
    els.pauseBtn.addEventListener('click', stopPlayback);
    els.stepFwdBtn.addEventListener('click', ()=> stepPlayback(1));
    els.stepBackBtn.addEventListener('click', ()=> stepPlayback(-1));
    els.playbackSpeed.addEventListener('input', (e)=>{ playbackSpeed = parseFloat(e.target.value); if (playbackInterval){ stopPlayback(); startPlayback(); } });
  }

  /* ---------------- MISSION JSON helpers ---------------- */
  function updateMissionJsonFromUserMarkers(){
    const wps = [];
    userMarkersLayer.eachLayer(l => { const p = l.getLatLng(); wps.push({ lat:+p.lat.toFixed(7), lon:+p.lng.toFixed(7) }); });
    const mission = { mission_id:'UI-'+Date.now(), waypoints:wps };
    els.missionArea.value = JSON.stringify(mission, null, 2);
  }

  function clearMapAndMission(){
    missionMarkersLayer.clearLayers(); userMarkersLayer.clearLayers(); missionRouteLayer.setLatLngs([]); breadcrumbLine.setLatLngs([]); arrowLayer.clearLayers();
    els.missionArea.value = JSON.stringify({ waypoints: [] }, null, 2);
    setProgress(0); els.turnsContainer.textContent = 'No mission'; els.etaText.textContent = 'N/A';
  }

  /* ---------------- FIREBASE LISTENERS ---------------- */
  function attachFirebaseListeners(){
    if (!robotRef) return;
    // ensure previous listeners removed
    try{ robotRef.child('mission').off(); robotRef.child('status').off(); } catch(e){}

    robotRef.child('mission').on('value', snap => { const mission = snap.val(); currentMission = mission || null; renderMission(mission); });

    robotRef.child('status').on('value', snap => { const status = snap.val(); lastStatus = status || null; const now = Date.now(); if (now - lastStatusUpdate > STATUS_THROTTLE_MS){ lastStatusUpdate = now; updateStatusUI(status); if (status && typeof status.lat === 'number' && typeof status.lng === 'number') handleRobotMovement(status); } });
  }

  /* ---------------- RENDER MISSION ---------------- */
  function renderMission(mission){
    missionMarkersLayer.clearLayers(); arrowLayer.clearLayers(); missionRouteLayer.setLatLngs([]); breadcrumbLine.setLatLngs([]); setProgress(0); els.turnsContainer.textContent = 'No mission'; els.etaText.textContent = 'N/A';
    if (!mission || !Array.isArray(mission.waypoints) || mission.waypoints.length === 0){ els.missionArea.value = JSON.stringify({ waypoints: [] }, null, 2); return; }

    // validate & sanitize waypoints
    const sanitized = mission.waypoints.filter(w=> isFiniteNumber(w.lat) && isFiniteNumber(w.lon));
    const latlngs = sanitized.map(w=> [w.lat, w.lon]);
    missionRouteLayer.setLatLngs(latlngs);
    try{ map.fitBounds(missionRouteLayer.getBounds(), { padding:[40,40] }); } catch(e){}

    const turns = [];
    for (let i=0;i<latlngs.length;i++){
      const ll = latlngs[i];
      L.marker(ll, { draggable:false }).addTo(missionMarkersLayer).bindPopup('Waypoint '+(i+1));
      if (i < latlngs.length-1){
        const a = latlngs[i], b = latlngs[i+1];
        const bearing = turf.bearing(turf.point([a[1], a[0]]), turf.point([b[1], b[0]]));
        const mid = [(a[0]+b[0])/2, (a[1]+b[1])/2];
        const arrow = L.marker(mid, { icon: L.divIcon({ className:'arrowMarker', html:`<div style="transform: rotate(${bearing}deg); font-size:18px; opacity:0.9;">➤</div>`, iconSize:[24,24], iconAnchor:[12,12] }), interactive:false }).addTo(arrowLayer);
        turns.push(`Segment ${i+1} → ${i+2}: bearing ${bearing.toFixed(1)}°`);
      }
    }

    els.turnsContainer.textContent = turns.join('\n') || 'No turns';
    els.missionArea.value = JSON.stringify(mission, null, 2);
    playbackIndex = 0; // reset
  }

  /* ---------------- STATUS UI ---------------- */
  function updateStatusUI(status){
    if (!status){ els.robotStatusDisplay.textContent = 'Robot status not available.'; return; }
    const lat = isFiniteNumber(status.lat) ? status.lat.toFixed(6) : 'N/A';
    const lng = isFiniteNumber(status.lng) ? status.lng.toFixed(6) : 'N/A';
    const heading = isFiniteNumber(status.heading) ? status.heading.toFixed(1)+'°' : 'N/A';
    const battery = isFiniteNumber(status.battery) ? status.battery+'%' : 'N/A';
    const speed = isFiniteNumber(status.speed) ? status.speed.toFixed(2)+' m/s' : 'N/A';
    const ultrasonic = isFiniteNumber(status.ultrasonic) ? status.ultrasonic.toFixed(1)+' cm' : 'N/A';
    const ts = status.ts ? new Date(status.ts*1000).toLocaleString() : new Date().toLocaleString();

    els.robotStatusDisplay.innerHTML = `\n      <span class="status-text">Lat: ${lat}</span> <span class="status-text">Lng: ${lng}</span> <span class="status-text">Head: ${heading}</span> <span class="status-text">Bat: ${battery}</span> <span class="status-text">Speed: ${speed}</span> <span class="status-text">US: ${ultrasonic}</span> <span class="status-text">Updated: ${ts}</span>\n    `;
  }

  /* ---------------- ROBOT MOVEMENT ---------------- */
  function handleRobotMovement(status){
    const lat = status.lat, lng = status.lng; const heading = status.heading; const speed = isFiniteNumber(status.speed) ? status.speed : null;
    const dest = L.latLng(lat, lng);
    if (!robotMarker){
      robotMarker = L.marker(dest, { icon: L.divIcon({ html: robotSvg(heading||0), className:'robotIcon' }), interactive:false }).addTo(map);
      if (followRobot) map.panTo(dest);
    } else {
      animateRobotTo(dest, heading);
    }

    if (showBreadcrumb) breadcrumbLine.addLatLng(dest);

    if (currentMission && Array.isArray(currentMission.waypoints) && currentMission.waypoints.length >= 2){
      const percent = computeProgressPercent({ lat, lng }, currentMission.waypoints);
      setProgress(percent);
      const remainingMeters = computeRemainingDistanceMeters({ lat, lng }, currentMission.waypoints);
      if (speed && speed > 0){ const seconds = remainingMeters / speed; const eta = new Date(Date.now() + seconds*1000); els.etaText.innerText = `${formatMeters(remainingMeters)} — ETA ${eta.toLocaleTimeString()}`; }
      else { els.etaText.innerText = `${formatMeters(remainingMeters)} — ETA N/A (speed unknown)`; }
    }
  }

  // requestAnimationFrame-based smooth movement
  function animateRobotTo(destLatLng, heading){
    if (!robotMarker) { robotMarker = L.marker(destLatLng).addTo(map); return; }
    const start = robotMarker.getLatLng();
    const duration = Math.max(200, Math.min(1200, 800 / playbackSpeed));
    const startTime = performance.now();

    function step(now){
      const t = Math.min(1, (now - startTime) / duration);
      const nextLat = start.lat + (destLatLng.lat - start.lat) * t;
      const nextLng = start.lng + (destLatLng.lng - start.lng) * t;
      const next = L.latLng(nextLat, nextLng);
      robotMarker.setLatLng(next);
      const el = robotMarker.getElement();
      if (el){ const div = el.querySelector('div'); if (div && isFiniteNumber(heading)) div.style.transform = `rotate(${heading}deg)`; }
      if (showBreadcrumb) breadcrumbLine.addLatLng(next);
      if (followRobot) map.panTo(next, { animate:false });
      if (t < 1) requestAnimationFrame(step);
    }
    requestAnimationFrame(step);
  }

  /* ---------------- PROGRESS / DISTANCE ---------------- */
  function computeProgressPercent(robotLatLng, waypoints){
    if (!robotLatLng || !waypoints || waypoints.length < 2) return 0;
    try{
      const line = turf.lineString(waypoints.map(w=>[w.lon,w.lat]));
      const pt = turf.point([robotLatLng.lng, robotLatLng.lat]);
      const snapped = turf.nearestPointOnLine(line, pt, {units:'kilometers'});
      const startPoint = turf.point(line.geometry.coordinates[0]);
      const sliceToSnapped = turf.lineSlice(startPoint, snapped, line);
      const distToSnappedKm = turf.length(sliceToSnapped, {units:'kilometers'}) || 0;
      const totalKm = turf.length(line, {units:'kilometers'}) || 0;
      if (totalKm === 0) return 0;
      return (distToSnappedKm / totalKm) * 100;
    } catch(e){ console.error('computeProgressPercent error', e); return 0; }
  }

  function computeRemainingDistanceMeters(robotLatLng, waypoints){
    if (!robotLatLng || !waypoints || waypoints.length < 2) return 0;
    try{
      const line = turf.lineString(waypoints.map(w=>[w.lon,w.lat]));
      const pt = turf.point([robotLatLng.lng, robotLatLng.lat]);
      const snapped = turf.nearestPointOnLine(line, pt, {units:'kilometers'});
      const endPt = turf.point(line.geometry.coordinates[line.geometry.coordinates.length-1]);
      const slice = turf.lineSlice(snapped, endPt, line);
      const km = turf.length(slice, {units:'kilometers'});
      return Math.round(km * 1000);
    } catch(e){ console.error('computeRemainingDistanceMeters error', e); return 0; }
  }

  /* ---------------- ETA + formatting ---------------- */
  function formatMeters(m){ if (m >= 1000) return (m/1000).toFixed(2)+' km'; return Math.round(m) + ' m'; }
  function setProgress(percent){ const p = Math.max(0, Math.min(100, percent)); els.progressBar.style.width = p + '%'; els.progressText.innerText = p.toFixed(1) + '%'; }

  /* ---------------- PLAYBACK ---------------- */
  function startPlayback(){
    if (!currentMission || !Array.isArray(currentMission.waypoints) || currentMission.waypoints.length < 2) return alert('Load a mission first to use playback.');
    stopPlayback();
    const coords = currentMission.waypoints.map(w => [w.lat, w.lon]);
    const route = buildPlaybackRoute(coords);
    if (!route.length) return;
    let idx = playbackIndex || 0;
    const baseDelay = 400; // ms between steps at speed=1
    playbackInterval = setInterval(()=>{
      if (idx >= route.length){ stopPlayback(); return; }
      const p = route[idx]; animateRobotTo({lat:p[0], lng:p[1]}, lastStatus && lastStatus.heading ? lastStatus.heading : null); setProgress(computeProgressPercent({lat:p[0], lng:p[1]}, currentMission.waypoints)); idx++; playbackIndex = idx;
    }, Math.max(50, baseDelay / playbackSpeed));
  }

  function stopPlayback(){ if (playbackInterval){ clearInterval(playbackInterval); playbackInterval = null; } }

  function stepPlayback(step){ if (!currentMission || !Array.isArray(currentMission.waypoints) || currentMission.waypoints.length < 2) return; const coords = currentMission.waypoints.map(w=>[w.lat,w.lon]); const route = buildPlaybackRoute(coords); if (!route.length) return; playbackIndex = (playbackIndex||0) + step; if (playbackIndex < 0) playbackIndex = 0; if (playbackIndex >= route.length) playbackIndex = route.length-1; const p = route[playbackIndex]; animateRobotTo({lat:p[0], lng:p[1]}, lastStatus && lastStatus.heading ? lastStatus.heading : null); setProgress(computeProgressPercent({lat:p[0], lng:p[1]}, currentMission.waypoints)); }

  function buildPlaybackRoute(coords){
    const route = [];
    for (let i=0;i<coords.length-1;i++){
      const a = coords[i], b = coords[i+1];
      const seg = turf.lineString([[a[1],a[0]],[b[1],b[0]]]);
      const segKm = turf.length(seg, {units:'kilometers'});
      const points = Math.max(6, Math.round(segKm * 100));
      for (let t=0;t<=points;t++){ const frac = t/points; const interp = turf.along(seg, segKm*frac, {units:'kilometers'}); route.push([interp.geometry.coordinates[1], interp.geometry.coordinates[0]]); }
    }
    return route;
  }

  /* ---------------- MISSION UPLOAD / FETCH ---------------- */
  function uploadMission(){
    const txt = els.missionArea.value;
    let mission = null;
    try{ mission = JSON.parse(txt); }
    catch(e){ return alert('Invalid JSON: ' + e.message); }
    if (!mission.waypoints || !Array.isArray(mission.waypoints)) return alert('Mission JSON must contain waypoints array.');
    // basic validation: coordinates within plausible ranges
    for (const wp of mission.waypoints){ if (!isFiniteNumber(wp.lat) || !isFiniteNumber(wp.lon) || wp.lat < -90 || wp.lat > 90 || wp.lon < -180 || wp.lon > 180) return alert('Invalid waypoint coordinates detected.'); }
    robotRef.child('mission').set(mission).then(()=> alert('Mission uploaded.')).catch(err=> alert('Upload failed: '+err.message));
  }

  function fetchMission(){ robotRef.child('mission').once('value', snap=>{ if (!snap.exists()) alert('No mission found.'); else { const mission = snap.val(); renderMission(mission); alert('Mission fetched and rendered.'); } }); }

  function refreshConfig(){ robotRef.child('config').once('value').then(snap=>{ const cfg = snap.val(); if (cfg) alert(`Config: BaseSpeed=${cfg.baseSpeed||'N/A'}, SafeDist=${cfg.safeDistance||'N/A'}`); else alert('No config'); }).catch(e=>console.error(e)); }

  /* ---------------- UTIL ---------------- */
  function isFiniteNumber(v){ return typeof v === 'number' && isFinite(v); }
  function formatMeters(m){ if (m >= 1000) return (m/1000).toFixed(2)+' km'; return Math.round(m) + ' m'; }

  function robotSvg(angle){ // inline small SVG for robot icon
    const sanitized = isFiniteNumber(angle) ? angle : 0;
    return `<div style="width:34px;height:34px;display:flex;align-items:center;justify-content:center;transform:rotate(${sanitized}deg);"><svg viewBox="0 0 24 24" width="28" height="28" aria-hidden="true"><path d="M12 2c1.1 0 2 .9 2 2v1h2v2h-1v1h-1v1h-2V9H9v1H7V9H6V8H8V6h2V5c0-1.1.9-2 2-2z" fill="currentColor"/></svg></div>`; }

  /* ---------------- PUBLIC ENTRYPNT ---------------- */
  function init(){ setupAuth(); }

  return { init };
})();

// start
document.addEventListener('DOMContentLoaded', ()=> App.init());
</script>
</body>
</html>

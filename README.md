<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">
<title>Brick Climber: 150 Wall Ascents</title>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; user-select: none; -webkit-user-select: none; }
  body, html { width: 100%; height: 100%; overflow: hidden; background: #87CEEB; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
  #canvas-container { width: 100%; height: 100%; position: absolute; left: 0; top: 0; z-index: 1; }
  
  #err-log {
    position: absolute; top: 70px; left: 10px; right: 10px; background: rgba(220,38,38,0.9);
    color: #fff; padding: 8px; font-size: 11px; z-index: 999; border-radius: 6px; display: none;
    font-family: monospace; word-break: break-all;
  }

  #top-hud {
    position: absolute; top: 12px; left: 10px; right: 10px;
    display: flex; justify-content: space-between; align-items: center;
    z-index: 10; pointer-events: none;
  }
  .hud-card {
    background: rgba(15, 23, 42, 0.88); border: 2px solid rgba(255, 255, 255, 0.2);
    border-radius: 10px; padding: 6px 12px; color: #fff; pointer-events: auto;
    display: flex; align-items: center; gap: 8px; font-size: 12px; font-weight: 700;
  }
  .title-text { color: #facc15; }
  .badge { background: #2563eb; padding: 3px 8px; border-radius: 6px; }
  select, button {
    background: #3b82f6; color: #fff; border: none; font-size: 11px;
    font-weight: 700; padding: 6px 10px; border-radius: 6px; cursor: pointer;
  }
  select { background: #334155; }

  #announcement {
    position: absolute; top: 80px; left: 50%; transform: translateX(-50%);
    background: #10b981; color: #fff; padding: 10px 24px; border-radius: 20px;
    font-size: 18px; font-weight: 800; z-index: 20; display: none; text-align: center;
    border: 2px solid #fff; box-shadow: 0 8px 20px rgba(0,0,0,0.3); pointer-events: none;
  }

  #touch-controls {
    position: absolute; bottom: 16px; left: 16px; right: 16px;
    display: flex; justify-content: space-between; align-items: flex-end;
    z-index: 10; pointer-events: none;
  }
  #joystick-base {
    width: 120px; height: 120px; background: rgba(255, 255, 255, 0.25);
    border: 3px solid rgba(255, 255, 255, 0.5); border-radius: 50%;
    position: relative; pointer-events: auto; touch-action: none;
  }
  #joystick-stick {
    width: 48px; height: 48px; background: #facc15;
    border: 3px solid #b45309; border-radius: 50%;
    position: absolute; top: 36px; left: 36px; pointer-events: none;
  }
  .act-col { display: flex; flex-direction: column; gap: 10px; pointer-events: auto; }
  .act-btn {
    width: 70px; height: 70px; border-radius: 50%; border: 3px solid #fff;
    color: #fff; font-size: 11px; font-weight: 800; cursor: pointer;
    box-shadow: 0 4px 12px rgba(0,0,0,0.3); display: flex; align-items: center; justify-content: center;
  }
  .btn-jump { background: #10b981; }
  .btn-grab { background: #f59e0b; }
  .btn-reset { background: #ef4444; width: 48px; height: 48px; align-self: flex-end; font-size: 9px; }
</style>

<!-- Primary + Secondary Fallback Three.js CDN -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
  if (typeof THREE === 'undefined') {
    document.write('<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/build/three.min.js"><\/script>');
  }
</script>
</head>
<body>

<div id="err-log"></div>
<div id="canvas-container"></div>

<div id="top-hud">
  <div class="hud-card">
    <span class="title-text">BRICK CLIMBER</span>
    <span class="badge" id="lvl-txt">LVL 1/150</span>
    <span class="badge" style="background:#8b5cf6;" id="hgt-txt">WALL: 2.5m</span>
  </div>
  <div class="hud-card">
    <button id="mode-btn">1 Player</button>
    <select id="lvl-pick"></select>
  </div>
</div>

<div id="announcement">LEVEL CLEAR!</div>

<div id="touch-controls">
  <div id="joystick-base">
    <div id="joystick-stick"></div>
  </div>
  <div class="act-col">
    <button class="act-btn btn-reset" id="btn-reset">RESET</button>
    <button class="act-btn btn-grab" id="btn-grab">GRAB / DROP</button>
    <button class="act-btn btn-jump" id="btn-jump">JUMP</button>
  </div>
</div>

<script>
// On-screen Error Logger (Shows if mobile GPU fails)
window.onerror = function(msg, url, line) {
  var d = document.getElementById('err-log');
  if (d) { d.style.display = 'block'; d.innerText = "Error: " + msg + " (Line " + line + ")"; }
};

// Simple Web Audio
var actx = null;
function snd(f, type, d) {
  try {
    if (!actx) actx = new (window.AudioContext || window.webkitAudioContext)();
    var osc = actx.createOscillator();
    var gain = actx.createGain();
    osc.type = type || 'sine';
    osc.frequency.setValueAtTime(f, actx.currentTime);
    gain.gain.setValueAtTime(0.12, actx.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.01, actx.currentTime + (d || 0.15));
    osc.connect(gain);
    gain.connect(actx.destination);
    osc.start();
    osc.stop(actx.currentTime + (d || 0.15));
  } catch(e){}
}

// Check Three.js Loaded
if (typeof THREE === 'undefined') {
  throw new Error("Three.js failed to load from CDN. Please check internet connection.");
}

// Three.js Scene Setup
var container = document.getElementById('canvas-container');
var scene = new THREE.Scene();
scene.background = new THREE.Color(0x87CEEB);

var camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 300);
var renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
container.appendChild(renderer.domElement);

// Lighting
var ambLight = new THREE.AmbientLight(0xffffff, 0.8);
scene.add(ambLight);

var dirLight = new THREE.DirectionalLight(0xffffff, 0.7);
dirLight.position.set(15, 30, 15);
scene.add(dirLight);

// Ground
var ground = new THREE.Mesh(
  new THREE.PlaneGeometry(36, 60),
  new THREE.MeshLambertMaterial({ color: 0x64748b })
);
ground.rotation.x = -Math.PI / 2;
scene.add(ground);

// Boundary Walls
var wallMat = new THREE.MeshLambertMaterial({ color: 0x334155 });
var wL = new THREE.Mesh(new THREE.BoxGeometry(1.2, 50, 60), wallMat);
wL.position.set(-18, 25, 0); scene.add(wL);
var wR = new THREE.Mesh(new THREE.BoxGeometry(1.2, 50, 60), wallMat);
wR.position.set(18, 25, 0); scene.add(wR);
var wB = new THREE.Mesh(new THREE.BoxGeometry(36, 50, 1.2), wallMat);
wB.position.set(0, 25, -30); scene.add(wB);

// Center Dividing Wall
var wallH = 2.5;
var wallMesh = new THREE.Mesh(
  new THREE.BoxGeometry(36, 2.5, 1.6),
  new THREE.MeshLambertMaterial({ color: 0x1e293b })
);
scene.add(wallMesh);

// Goal Pad (Z = 16)
var goalPad = new THREE.Mesh(
  new THREE.CylinderGeometry(2.2, 2.3, 0.3, 24),
  new THREE.MeshLambertMaterial({ color: 0x10b981 })
);
goalPad.position.set(0, 0.15, 16);
scene.add(goalPad);

// Avatar Generator
function makeBuddy(col) {
  var g = new THREE.Group();
  var h = new THREE.Mesh(new THREE.BoxGeometry(0.7, 0.7, 0.7), new THREE.MeshLambertMaterial({ color: 0xfacc15 }));
  h.position.y = 1.65; g.add(h);

  var v = new THREE.Mesh(new THREE.BoxGeometry(0.55, 0.18, 0.15), new THREE.MeshLambertMaterial({ color: 0x0f172a }));
  v.position.set(0, 1.7, 0.35); g.add(v);

  var t = new THREE.Mesh(new THREE.BoxGeometry(1.0, 1.0, 0.55), new THREE.MeshLambertMaterial({ color: col }));
  t.position.y = 0.95; g.add(t);

  var aGeo = new THREE.BoxGeometry(0.3, 0.8, 0.3);
  var aMat = new THREE.MeshLambertMaterial({ color: 0xfacc15 });
  var lArmP = new THREE.Group(); lArmP.position.set(-0.68, 1.3, 0);
  var lA = new THREE.Mesh(aGeo, aMat); lA.position.y = -0.38; lArmP.add(lA); g.add(lArmP);
  var rArmP = new THREE.Group(); rArmP.position.set(0.68, 1.3, 0);
  var rA = new THREE.Mesh(aGeo, aMat); rA.position.y = -0.38; rArmP.add(rA); g.add(rArmP);

  var legGeo = new THREE.BoxGeometry(0.36, 0.9, 0.36);
  var legMat = new THREE.MeshLambertMaterial({ color: 0x1e293b });
  var lLegP = new THREE.Group(); lLegP.position.set(-0.25, 0.5, 0);
  var lL = new THREE.Mesh(legGeo, legMat); lL.position.y = -0.42; lLegP.add(lL); g.add(lLegP);
  var rLegP = new THREE.Group(); rLegP.position.set(0.25, 0.5, 0);
  var rL = new THREE.Mesh(legGeo, legMat); rL.position.y = -0.42; rLegP.add(rL); g.add(rLegP);

  return { root: g, lArmP: lArmP, rArmP: rArmP, lLegP: lLegP, rLegP: rLegP };
}

var p1 = { id: 1, avatar: makeBuddy(0x2563eb), pos: new THREE.Vector3(0, 0, -18), vel: new THREE.Vector3(), yaw: 0, isGrounded: true, heldItem: null, dir: new THREE.Vector2(), step: 0 };
scene.add(p1.avatar.root);

var p2 = { id: 2, avatar: makeBuddy(0x16a34a), pos: new THREE.Vector3(3, 0, -18), vel: new THREE.Vector3(), yaw: 0, isGrounded: true, heldItem: null, dir: new THREE.Vector2(), step: 0 };
scene.add(p2.avatar.root);
p2.avatar.root.visible = false;

var is2P = false;
var currentLvl = 1;
var items = [];

// Populate Level Selector
var lvlPick = document.getElementById('lvl-pick');
for (var i = 1; i <= 150; i++) {
  var o = document.createElement('option');
  o.value = i; o.textContent = 'Lvl ' + i;
  lvlPick.appendChild(o);
}
lvlPick.addEventListener('change', function(e) { loadLevel(parseInt(e.target.value)); });

function loadLevel(lvl) {
  currentLvl = lvl;
  lvlPick.value = lvl;
  document.getElementById('lvl-txt').textContent = 'LVL ' + lvl + '/150';

  wallH = 2.5 + (lvl - 1) * 0.32;
  document.getElementById('hgt-txt').textContent = 'WALL: ' + wallH.toFixed(1) + 'm';

  wallMesh.geometry.dispose();
  wallMesh.geometry = new THREE.BoxGeometry(36, wallH, 1.6);
  wallMesh.position.set(0, wallH / 2, 0);

  items.forEach(function(it) { scene.remove(it.mesh); });
  items = [];
  p1.heldItem = null;
  p2.heldItem = null;

  p1.pos.set(is2P ? -3 : 0, 0, -18);
  p2.pos.set(3, 0, -18);
  p1.vel.set(0,0,0);
  p2.vel.set(0,0,0);

  var count = Math.min(65, 4 + Math.floor((lvl - 1) * 0.42));
  for (var j = 0; j < count; j++) {
    var sz, col;
    var r = Math.random();
    if (j < 3 || r < 0.45) { sz = [1.2, 0.6, 1.2]; col = 0xf59e0b; }
    else if (r < 0.75) { sz = [1.3, 1.3, 1.3]; col = 0xd97706; }
    else if (r < 0.90) { sz = [1.1, 2.8, 1.1]; col = 0x64748b; }
    else { sz = [3.6, 0.45, 1.2]; col = 0x854d0e; }

    var m = new THREE.Mesh(new THREE.BoxGeometry(sz[0], sz[1], sz[2]), new THREE.MeshLambertMaterial({ color: col }));
    var rx = (Math.random() - 0.5) * 26;
    var rz = -15 + Math.random() * 11;
    m.position.set(rx, sz[1] / 2, rz);
    scene.add(m);
    items.push({ mesh: m, size: sz, heldBy: null });
  }
}

// Keyboard controls
var keys = {};
window.addEventListener('keydown', function(e) {
  keys[e.code] = true;
  if (e.code === 'KeyE') doAction(p1, 'grab');
  if (e.code === 'Space') doAction(p1, 'jump');
  if (e.code === 'KeyR') loadLevel(currentLvl);
  if (is2P) {
    if (e.code === 'KeyM' || e.code === 'ShiftRight') doAction(p2, 'grab');
    if (e.code === 'Enter') doAction(p2, 'jump');
  }
});
window.addEventListener('keyup', function(e) { keys[e.code] = false; });

// Virtual Joystick for Mobile
var joyBase = document.getElementById('joystick-base');
var joyStick = document.getElementById('joystick-stick');
var jTouchId = null;
var jCenter = { x: 0, y: 0 };

joyBase.addEventListener('touchstart', function(e) {
  var t = e.changedTouches[0];
  jTouchId = t.identifier;
  var r = joyBase.getBoundingClientRect();
  jCenter = { x: r.left + r.width / 2, y: r.top + r.height / 2 };
  updateJoy(t.clientX, t.clientY);
}, { passive: false });

window.addEventListener('touchmove', function(e) {
  for (var i = 0; i < e.changedTouches.length; i++) {
    if (e.changedTouches[i].identifier === jTouchId) updateJoy(e.changedTouches[i].clientX, e.changedTouches[i].clientY);
  }
}, { passive: false });

function resetJoy() {
  jTouchId = null;
  joyStick.style.transform = 'translate(0, 0)';
  p1.dir.set(0, 0);
}
window.addEventListener('touchend', function(e) {
  for (var i = 0; i < e.changedTouches.length; i++) {
    if (e.changedTouches[i].identifier === jTouchId) resetJoy();
  }
});
window.addEventListener('touchcancel', resetJoy);

function updateJoy(cx, cy) {
  var dx = cx - jCenter.x;
  var dy = cy - jCenter.y;
  var dist = Math.hypot(dx, dy);
  var maxR = 40;
  var ang = Math.atan2(dy, dx);
  var cDist = Math.min(dist, maxR);
  var nx = Math.cos(ang) * cDist;
  var ny = Math.sin(ang) * cDist;
  joyStick.style.transform = 'translate(' + nx + 'px, ' + ny + 'px)';
  p1.dir.set(nx / maxR, -ny / maxR);
}

document.getElementById('btn-jump').addEventListener('pointerdown', function() { doAction(p1, 'jump'); });
document.getElementById('btn-grab').addEventListener('pointerdown', function() { doAction(p1, 'grab'); });
document.getElementById('btn-reset').addEventListener('pointerdown', function() { loadLevel(currentLvl); });

var modeBtn = document.getElementById('mode-btn');
modeBtn.addEventListener('click', function() {
  is2P = !is2P;
  modeBtn.textContent = is2P ? '2 Players' : '1 Player';
  p2.avatar.root.visible = is2P;
  loadLevel(currentLvl);
});

function doAction(p, act) {
  if (act === 'jump' && p.isGrounded) {
    p.vel.y = 8.5; p.isGrounded = false; snd(360, 'triangle', 0.2);
  } else if (act === 'grab') {
    if (p.heldItem) {
      snd(200, 'square', 0.15);
      p.heldItem.heldBy = null;
      p.heldItem.mesh.position.y = Math.max(p.heldItem.size[1]/2, p.pos.y);
      p.heldItem = null;
    } else {
      var nearest = null;
      var minD = 3.5;
      for (var k = 0; k < items.length; k++) {
        var it = items[k];
        if (it.heldBy) continue;
        var d = p.pos.distanceTo(it.mesh.position);
        if (d < minD) { minD = d; nearest = it; }
      }
      if (nearest) {
        snd(480, 'sine', 0.12);
        p.heldItem = nearest;
        nearest.heldBy = p;
      }
    }
  }
}

// Physics & Collision
function updatePlayer(p, dt) {
  var mx = 0, mz = 0;
  if (p.id === 1) {
    if (keys['KeyW']) mz += 1;
    if (keys['KeyS']) mz -= 1;
    if (keys['KeyA']) mx -= 1;
    if (keys['KeyD']) mx += 1;
    if (p.dir.lengthSq() > 0.05) { mx = p.dir.x; mz = p.dir.y; }
  } else if (p.id === 2 && is2P) {
    if (keys['ArrowUp']) mz += 1;
    if (keys['ArrowDown']) mz -= 1;
    if (keys['ArrowLeft']) mx -= 1;
    if (keys['ArrowRight']) mx += 1;
  }

  var len = Math.hypot(mx, mz);
  if (len > 0.05) {
    p.pos.x += (mx / (len > 1 ? len : 1)) * 8.0 * dt;
    p.pos.z += (mz / (len > 1 ? len : 1)) * 8.0 * dt;
    p.yaw = Math.atan2(mx, mz);
    p.step += dt * 14;
  } else {
    p.step = 0;
  }

  p.vel.y -= 18.0 * dt;
  p.pos.y += p.vel.y * dt;

  var floorY = 0;
  var pr = 0.5;

  // Wall Collision
  if (Math.abs(p.pos.x) < 18 + pr) {
    if (p.pos.z + pr > -0.8 && p.pos.z - pr < 0.8) {
      if (p.pos.y >= wallH) { floorY = Math.max(floorY, wallH); }
      else { p.pos.z = p.pos.z < 0 ? -0.8 - pr : 0.8 + pr; }
    }
  }

  // Items Collision & Step-Up
  for (var i = 0; i < items.length; i++) {
    var it = items[i];
    if (it.heldBy === p) continue;
    var hx = it.size[0] / 2;
    var hy = it.size[1] / 2;
    var hz = it.size[2] / 2;
    var ip = it.mesh.position;

    if (
      p.pos.x + pr > ip.x - hx && p.pos.x - pr < ip.x + hx &&
      p.pos.z + pr > ip.z - hz && p.pos.z - pr < ip.z + hz
    ) {
      var topY = ip.y + hy;
      if (p.pos.y >= topY - 0.85 && p.vel.y <= 0.5) {
        if (topY > floorY) floorY = topY;
      } else {
        var ox = (hx + pr) - Math.abs(p.pos.x - ip.x);
        var oz = (hz + pr) - Math.abs(p.pos.z - ip.z);
        if (oz < ox) p.pos.z = p.pos.z > ip.z ? ip.z + hz + pr : ip.z - hz - pr;
        else p.pos.x = p.pos.x > ip.x ? ip.x + hx + pr : ip.x - hx - pr;
      }
    }
  }

  if (p.pos.y <= floorY) {
    p.pos.y = floorY;
    p.vel.y = 0;
    p.isGrounded = true;
  } else {
    p.isGrounded = false;
  }

  p.pos.x = Math.max(-17, Math.min(17, p.pos.x));
  p.pos.z = Math.max(-28, Math.min(28, p.pos.z));

  p.avatar.root.position.copy(p.pos);
  p.avatar.root.rotation.y = p.yaw;

  var swing = Math.sin(p.step) * 0.6;
  p.avatar.lLegP.rotation.x = swing;
  p.avatar.rLegP.rotation.x = -swing;

  if (p.heldItem) {
    p.avatar.lArmP.rotation.x = 1.35;
    p.avatar.rArmP.rotation.x = 1.35;
    var fwd = new THREE.Vector3(0, 0, 1.4).applyAxisAngle(new THREE.Vector3(0, 1, 0), p.yaw);
    p.heldItem.mesh.position.set(p.pos.x + fwd.x, p.pos.y + 1.2, p.pos.z + fwd.z);
    p.heldItem.mesh.rotation.y = p.yaw;
  } else {
    p.avatar.lArmP.rotation.x = -swing;
    p.avatar.rArmP.rotation.x = swing;
  }

  if (p.pos.distanceTo(goalPad.position) < 2.5 && p.pos.y <= 0.6) {
    winLevel();
  }
}

var winning = false;
function winLevel() {
  if (winning) return;
  winning = true;
  snd(523, 'triangle', 0.2);
  setTimeout(function() { snd(659, 'triangle', 0.2); }, 120);
  setTimeout(function() { snd(784, 'triangle', 0.3); }, 240);

  var banner = document.getElementById('announcement');
  banner.style.display = 'block';
  banner.textContent = 'LEVEL ' + currentLvl + ' CLEARED!';

  setTimeout(function() {
    banner.style.display = 'none';
    winning = false;
    loadLevel(currentLvl < 150 ? currentLvl + 1 : 1);
  }, 1400);
}

// Render Loop
var lastT = performance.now();
function loop() {
  requestAnimationFrame(loop);
  var now = performance.now();
  var dt = Math.min((now - lastT) / 1000, 0.08);
  lastT = now;

  updatePlayer(p1, dt);
  if (is2P) updatePlayer(p2, dt);

  if (!is2P) {
    camera.position.lerp(new THREE.Vector3(p1.pos.x * 0.4, Math.max(p1.pos.y + 5.5, 6.0), p1.pos.z - 9.5), 0.08);
    camera.lookAt(p1.pos.x, p1.pos.y + 1.5, p1.pos.z + 3.0);
  } else {
    var mx = (p1.pos.x + p2.pos.x) / 2;
    var my = (p1.pos.y + p2.pos.y) / 2;
    var mz = (p1.pos.z + p2.pos.z) / 2;
    camera.position.lerp(new THREE.Vector3(mx * 0.4, Math.max(my + 7.0, 7.5), mz - 12.0), 0.08);
    camera.lookAt(mx, my + 1.5, mz + 2.5);
  }

  renderer.render(scene, camera);
}

window.addEventListener('resize', function() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});

// Start
loadLevel(1);
loop();
</script>
</body>
</html>

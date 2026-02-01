<html>
<head>
  <title>Doom 3D Style - Fixed & Better</title>
  <style>
    * { margin:0; padding:0; }
    body { background:#000; overflow:hidden; font-family:monospace; color:#0f0; }
    canvas { display:block; cursor:none; image-rendering:pixelated; }
    #info { position:absolute; top:10px; left:10px; z-index:10; }
    #minimap { position:absolute; top:10px; right:10px; border:2px solid #0f0; image-rendering:pixelated; }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  <canvas id="minimap" width="200" height="200"></canvas>
  <div id="info">
    Click to lock mouse & play<br>
    WASD: Move/Strafe | Mouse: Look/Shoot
  </div>

<script>
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const minimap = document.getElementById('minimap');
const mctx = minimap.getContext('2d');

function resize() {
  canvas.width = innerWidth;
  canvas.height = innerHeight;
  ctx.imageSmoothingEnabled = false;
}
resize();
window.addEventListener('resize', resize);

// Game state
const player = {
  x: 1.5, y: 1.5,
  angle: 0,
  speed: 0.1,
  rotSpeed: 0.04
};
const FOV = Math.PI / 3;
const HALF_FOV = FOV / 2;
const NUM_RAYS = 400; // Balance perf/quality
const MAX_DEPTH = 16;
const DELTA_ANGLE = FOV / NUM_RAYS;
const DIST = NUM_RAYS / (2 * Math.tan(HALF_FOV));
const PROJ_COEFF = 3 * DIST * canvas.height; // Wait, dynamic later

// Bigger map
const map = [
  [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,0,1,1,0,0,1,1,1,1,0,0,1,1,0,1],
  [1,0,0,1,0,0,0,0,0,0,0,0,1,0,0,1],
  [1,0,0,0,0,1,1,0,0,1,1,0,0,0,0,1],
  [1,0,0,0,0,1,0,0,0,0,1,0,0,0,0,1],
  [1,0,1,1,0,1,0,1,1,0,1,0,1,1,0,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,0,1,0,0,0,1,0,0,1,0,0,0,1,0,1],
  [1,0,0,0,1,0,0,0,0,0,0,1,0,0,0,1],
  [1,0,0,1,1,1,1,1,1,1,1,1,1,0,0,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,0,1,0,0,0,0,1,1,0,0,0,0,1,0,1],
  [1,0,0,0,0,1,0,1,1,0,1,0,0,0,0,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1]
];
const MAP_SIZE = map.length;
const TILE_SIZE = 64; // For minimap

// Input
const keys = {};
let mouseLocked = false;
let shootFlash = 0;

// Events
document.addEventListener('click', () => {
  canvas.requestPointerLock();
});
document.addEventListener('pointerlockchange', () => {
  mouseLocked = document.pointerLockElement === canvas;
});
document.addEventListener('mousemove', (e) => {
  if (mouseLocked) {
    player.angle += e.movementX * 0.002;
  }
});
document.addEventListener('mousedown', () => {
  if (mouseLocked) shootFlash = 20;
});
window.addEventListener('keydown', (e) => keys[e.key.toLowerCase()] = true);
window.addEventListener('keyup', (e) => keys[e.key.toLowerCase()] = false);

// Movement
function move() {
  const forward = Math.cos(player.angle) * player.speed;
  const strafe = Math.sin(player.angle) * player.speed;
  const right = Math.cos(player.angle + Math.PI / 2) * player.speed;
  const left = Math.cos(player.angle - Math.PI / 2) * player.speed;

  if (keys['w']) {
    const nx = player.x + forward, ny = player.y + Math.sin(player.angle) * player.speed;
    if (map[Math.floor(ny)]?.[Math.floor(nx)] === 0) {
      player.x = nx; player.y = ny;
    }
  }
  if (keys['s']) {
    const nx = player.x - forward, ny = player.y - Math.sin(player.angle) * player.speed;
    if (map[Math.floor(ny)]?.[Math.floor(nx)] === 0) {
      player.x = nx; player.y = ny;
    }
  }
  if (keys['a']) {
    const nx = player.x + left, ny = player.y + Math.sin(player.angle - Math.PI / 2) * player.speed;
    if (map[Math.floor(ny)]?.[Math.floor(nx)] === 0) {
      player.x = nx; player.y = ny;
    }
  }
  if (keys['d']) {
    const nx = player.x + right, ny = player.y - Math.sin(player.angle + Math.PI / 2) * player.speed;
    if (map[Math.floor(ny)]?.[Math.floor(nx)] === 0) {
      player.x = nx; player.y = ny;
    }
  }
}

// DDA Raycaster
function castRay(rayAngle) {
  const sin = Math.sin(rayAngle);
  const cos = Math.cos(rayAngle);

  // DDA setup
  let mapX = Math.floor(player.x);
  let mapY = Math.floor(player.y);

  const deltaDistX = Math.abs(1 / cos);
  const deltaDistY = Math.abs(1 / sin);

  let stepX, stepY;
  let sideDistX, sideDistY;

  if (cos > 0) { stepX = 1; sideDistX = (mapX + 1 - player.x) * deltaDistX; }
  else { stepX = -1; sideDistX = (player.x - mapX) * deltaDistX; }
  if (sin > 0) { stepY = 1; sideDistY = (mapY + 1 - player.y) * deltaDistY; }
  else { stepY = -1; sideDistY = (player.y - mapY) * deltaDistY; }

  let side = 0;
  while (true) {
    if (sideDistX < sideDistY) {
      sideDistX += deltaDistX;
      mapX += stepX;
      side = 0;
    } else {
      sideDistY += deltaDistY;
      mapY += stepY;
      side = 1;
    }
    if (map[mapY]?.[mapX] === 1) break;
    if (mapX < 0 || mapX >= MAP_SIZE || mapY < 0 || mapY >= MAP_SIZE) return {dist: MAX_DEPTH};
  }

  const dist = side === 0 ? sideDistX - deltaDistX : sideDistY - deltaDistY;
  const perpDist = dist * Math.cos(player.angle - rayAngle);
  return {dist: perpDist, side};
}

// Render
function render() {
  // Resize proj
  const projCoeff = 277 * canvas.height / 277; // Approx for 277 fov-ish, adjust

  // Clear
  const skyH = canvas.height * 0.15;
  const gradSky = ctx.createLinearGradient(0, 0, 0, skyH);
  gradSky.addColorStop(0, '#001122');
  gradSky.addColorStop(1, '#334455');
  ctx.fillStyle = gradSky;
  ctx.fillRect(0, 0, canvas.width, skyH);

  const gradFloor = ctx.createLinearGradient(0, skyH, 0, canvas.height);
  gradFloor.addColorStop(0, '#444');
  gradFloor.addColorStop(1, '#111');
  ctx.fillStyle = gradFloor;
  ctx.fillRect(0, skyH, canvas.width, canvas.height - skyH);

  // Shoot flash
  if (shootFlash > 0) {
    ctx.fillStyle = `rgba(255,200,100,${shootFlash / 40})`;
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    shootFlash--;
  }

  // Rays
  for (let i = 0; i < NUM_RAYS; i++) {
    const rayAngle = player.angle - HALF_FOV + i * DELTA_ANGLE;
    const {dist, side} = castRay(rayAngle);

    const lineH = Math.min(projCoeff / (dist + 0.0001), canvas.height);

    if (side === 1) {
      ctx.fillStyle = `rgb(80,80,80)`; // Darker for side
    } else {
      const shade = 100 + (100 * (1 - dist / MAX_DEPTH));
      ctx.fillStyle = `rgb(${shade},${shade * 0.7},${shade * 0.4})`;
    }

    const wallTop = (canvas.height - lineH) / 2;
    ctx.fillRect(i * (canvas.width / NUM_RAYS), wallTop, canvas.width / NUM_RAYS + 1, lineH);
  }

  // Crosshair / Gun
  ctx.strokeStyle = '#ff0';
  ctx.lineWidth = 3;
  ctx.lineCap = 'round';
  const cx = canvas.width / 2, cy = canvas.height * 0.85;
  ctx.beginPath();
  ctx.moveTo(cx - 20, cy); ctx.lineTo(cx + 20, cy);
  ctx.moveTo(cx, cy - 20); ctx.lineTo(cx, cy + 20);
  ctx.stroke();
}

// Minimap
function minimapRender() {
  mctx.fillStyle = '#000';
  mctx.fillRect(0, 0, 200, 200);

  const scale = 200 / MAP_SIZE;

  // Map
  for (let y = 0; y < MAP_SIZE; y++) {
    for (let x = 0; x < MAP_SIZE; x++) {
      if (map[y][x] === 1) {
        mctx.fillStyle = '#0f0';
        mctx.fillRect(x * scale, y * scale, scale, scale);
      }
    }
  }

  // Player
  mctx.fillStyle = '#f00';
  mctx.fillRect(player.x * scale - 2, player.y * scale - 2, 4, 4);

  // Dir
  mctx.strokeStyle = '#ff0';
  mctx.lineWidth = 2;
  mctx.beginPath();
  mctx.moveTo(player.x * scale, player.y * scale);
  mctx.lineTo(player.x * scale + Math.cos(player.angle) * 20, player.y * scale + Math.sin(player.angle) * 20);
  mctx.stroke();
}

// Loop
function loop() {
  move();
  render();
  minimapRender();
  requestAnimationFrame(loop);
}
loop();
</script>
</body>
</html>

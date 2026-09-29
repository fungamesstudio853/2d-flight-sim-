<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>2D Airbus A320 Flight Simulator</title>
  <style>
    body {
      margin: 0;
      padding: 0;
      background-color: #111;
      color: #fff;
      font-family: sans-serif;
      overflow: hidden;
      display: flex;
      flex-direction: column;
      align-items: center;
    }

    #gameCanvas {
      background: #87CEEB;
      box-shadow: 0 0 20px rgba(0,0,0,0.5);
    }

    .controls-hint {
      margin-top: 8px;
      font-size: 14px;
      color: #ccc;
    }
  </style>
</head>
<body>

<canvas id="gameCanvas" width="900" height="400"></canvas>
<div class="controls-hint">
  <b>Controls:</b> Up/Down Arrows (or W/S) = Pitch | Right/Left Arrows (or D/A) = Throttle | <b>G</b> = Gear | <b>R</b> = Restart
</div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// Game State
let plane = {
  x: 200,
  y: 300,
  vx: 0,
  vy: 0,
  speed: 0,
  maxSpeed: 12,
  angle: 0,
  gearPosition: 1, // 1 = down, 0 = up
  gearTarget: 1,   // target for animation
  gearSpeed: 0.02,
  altitude: 0,
  crashed: false
};

let cameraX = 0;
const GROUND_Y = 320;
const keys = {};

// Particle Arrays
let particles = [];
let debrisList = [];

function resetGame() {
  plane = {
    x: 200,
    y: 300,
    vx: 0,
    vy: 0,
    speed: 0,
    maxSpeed: 12,
    angle: 0,
    gearPosition: 1,
    gearTarget: 1,
    gearSpeed: 0.02,
    altitude: 0,
    crashed: false
  };
  particles = [];
  debrisList = [];
}

// Keyboard input tracking
window.addEventListener('keydown', (e) => {
  if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', ' '].includes(e.key)) {
    e.preventDefault();
  }

  keys[e.key.toLowerCase()] = true;
  keys[e.key] = true;

  if (e.key.toLowerCase() === 'g' && !plane.crashed) {
    plane.gearTarget = plane.gearTarget === 1 ? 0 : 1;
  }

  if (e.key.toLowerCase() === 'r') {
    resetGame();
  }
});

window.addEventListener('keyup', (e) => {
  keys[e.key.toLowerCase()] = false;
  keys[e.key] = false;
});

// Particle Spawners
function spawnSparks(x, y, amount) {
  for (let i = 0; i < amount; i++) {
    particles.push({
      x: x + (Math.random() - 0.5) * 20,
      y: y,
      vx: (Math.random() - 0.5) * 6 - plane.speed * 0.5,
      vy: -Math.random() * 4 - 1,
      size: Math.random() * 3 + 1,
      color: Math.random() > 0.3 ? '#FFA500' : '#FFFF00',
      life: 1,
      decay: Math.random() * 0.05 + 0.02
    });
  }
}

function triggerExplosion(x, y) {
  // Fire and Smoke Particles
  for (let i = 0; i < 80; i++) {
    const isFire = Math.random() > 0.4;
    particles.push({
      x: x,
      y: y,
      vx: (Math.random() - 0.5) * 12,
      vy: (Math.random() - 0.5) * 12 - 4,
      size: Math.random() * 12 + 4,
      color: isFire ? (Math.random() > 0.5 ? '#FF4500' : '#FF8C00') : '#555555',
      life: 1,
      decay: Math.random() * 0.02 + 0.01
    });
  }

  // Debris Flying Out
  const colors = ['#E8E8E8', '#003399', '#555', '#333', '#B0C4DE'];
  for (let i = 0; i < 25; i++) {
    debrisList.push({
      x: x,
      y: y,
      vx: (Math.random() - 0.5) * 16,
      vy: -Math.random() * 12 - 4,
      angle: Math.random() * Math.PI * 2,
      vRot: (Math.random() - 0.5) * 0.3,
      size: Math.random() * 15 + 5,
      color: colors[Math.floor(Math.random() * colors.length)]
    });
  }
}

// Update Game Logic
function update() {
  if (!plane.crashed) {
    // Throttle Controls
    if (keys['ArrowRight'] || keys['d']) plane.speed = Math.min(plane.speed + 0.05, plane.maxSpeed);
    if (keys['ArrowLeft'] || keys['a']) plane.speed = Math.max(plane.speed - 0.05, 0);

    // Pitch Controls
    if (keys['ArrowUp'] || keys['w']) plane.angle -= 0.015;
    if (keys['ArrowDown'] || keys['s']) plane.angle += 0.015;

    // Angle limit
    plane.angle = Math.max(-0.6, Math.min(0.5, plane.angle));

    // Gear Animation
    if (plane.gearPosition < plane.gearTarget) {
      plane.gearPosition = Math.min(plane.gearPosition + plane.gearSpeed, 1);
    } else if (plane.gearPosition > plane.gearTarget) {
      plane.gearPosition = Math.max(plane.gearPosition - plane.gearSpeed, 0);
    }

    // Physics
    const lift = (plane.speed / plane.maxSpeed) * (-plane.angle) * 4;
    const gravity = 0.5;
    plane.vy = gravity - lift;
    plane.vx = plane.speed;

    cameraX += plane.vx;
    plane.y += plane.vy;

    // Ground Interaction
    if (plane.y >= GROUND_Y - 15) {
      // Check for High Speed / Steep Nose-Down Impact (Explosion)
      if (plane.vy > 3.5 || plane.angle > 0.25) {
        plane.crashed = true;
        triggerExplosion(plane.x, GROUND_Y - 10);
      } else {
        // Soft Touchdown
        plane.y = GROUND_Y - 15;
        if (plane.vy > 0) plane.vy = 0;

        // Belly Landing without Gear Down -> Create Sparks!
        if (plane.gearPosition < 0.5 && plane.speed > 0.5) {
          spawnSparks(plane.x, GROUND_Y - 5, 4);
          plane.speed = Math.max(0, plane.speed - 0.03); // Friction slows down plane
        }

        if (plane.angle > 0) plane.angle = 0;
      }
    }
  }

  // Update Particles (Sparks, Smoke, Fire)
  for (let i = particles.length - 1; i >= 0; i--) {
    let p = particles[i];
    p.x += p.vx;
    p.y += p.vy;
    p.life -= p.decay;
    if (p.life <= 0) particles.splice(i, 1);
  }

  // Update Debris
  for (let i = debrisList.length - 1; i >= 0; i--) {
    let d = debrisList[i];
    d.x += d.vx;
    d.y += d.vy;
    d.vy += 0.4; // Gravity on debris
    d.angle += d.vRot;

    // Bounce Debris on Ground
    if (d.y >= GROUND_Y - 5) {
      d.y = GROUND_Y - 5;
      d.vy = -d.vy * 0.4;
      d.vx *= 0.7;
    }
  }

  plane.altitude = Math.max(0, Math.round(GROUND_Y - 15 - plane.y));
}

// Draw Environment
function drawEnvironment() {
  const skyGradient = ctx.createLinearGradient(0, 0, 0, GROUND_Y);
  skyGradient.addColorStop(0, '#1E90FF');
  skyGradient.addColorStop(1, '#87CEEB');
  ctx.fillStyle = skyGradient;
  ctx.fillRect(0, 0, canvas.width, GROUND_Y);

  ctx.fillStyle = '#2d8a4e';
  ctx.fillRect(0, GROUND_Y, canvas.width, canvas.height - GROUND_Y);

  ctx.fillStyle = '#444';
  ctx.fillRect(0, GROUND_Y - 5, canvas.width, 10);

  ctx.fillStyle = '#FFF';
  const stripeWidth = 40;
  const stripeGap = 40;
  const cycle = stripeWidth + stripeGap;
  const offset = cameraX % cycle;

  for (let x = -offset; x < canvas.width; x += cycle) {
    ctx.fillRect(x, GROUND_Y - 1, stripeWidth, 2);
  }
}

// Draw Airbus A320
function drawA320() {
  if (plane.crashed) return; // Don't draw plane if destroyed

  ctx.save();
  ctx.translate(plane.x, plane.y);
  ctx.rotate(plane.angle);

  // Landing Gear Animation
  if (plane.gearPosition > 0) {
    const gearLength = 12 * plane.gearPosition;
    ctx.fillStyle = '#333';
    ctx.strokeStyle = '#888';
    ctx.lineWidth = 3;

    ctx.beginPath();
    ctx.moveTo(35, 8);
    ctx.lineTo(35, 8 + gearLength);
    ctx.stroke();
    ctx.fillRect(32, 8 + gearLength, 6, 4);

    ctx.beginPath();
    ctx.moveTo(-15, 8);
    ctx.lineTo(-15, 8 + gearLength);
    ctx.stroke();
    ctx.fillRect(-18, 8 + gearLength, 6, 4);
  }

  // Fuselage
  ctx.fillStyle = '#E8E8E8';
  ctx.beginPath();
  ctx.ellipse(0, 0, 55, 10, 0, 0, Math.PI * 2);
  ctx.fill();

  // Nose
  ctx.fillStyle = '#CCC';
  ctx.beginPath();
  ctx.arc(50, 0, 5, 0, Math.PI * 2);
  ctx.fill();

  // Cockpit Windows
  ctx.fillStyle = '#111';
  ctx.beginPath();
  ctx.moveTo(35, -4);
  ctx.lineTo(48, -2);
  ctx.lineTo(45, 1);
  ctx.lineTo(35, 1);
  ctx.closePath();
  ctx.fill();

  // Tail
  ctx.fillStyle = '#003399';
  ctx.beginPath();
  ctx.moveTo(-45, -8);
  ctx.lineTo(-58, -30);
  ctx.lineTo(-42, -30);
  ctx.lineTo(-32, -8);
  ctx.closePath();
  ctx.fill();

  // Wing
  ctx.fillStyle = '#B0C4DE';
  ctx.beginPath();
  ctx.moveTo(10, 2);
  ctx.lineTo(-20, 18);
  ctx.lineTo(-10, 18);
  ctx.lineTo(22, 2);
  ctx.closePath();
  ctx.fill();

  // Engine
  ctx.fillStyle = '#555';
  ctx.fillRect(0, 10, 18, 7);
  ctx.fillStyle = '#333';
  ctx.fillRect(16, 11, 3, 5);

  ctx.restore();
}

// Draw Particles & Debris
function drawParticles() {
  particles.forEach(p => {
    ctx.fillStyle = p.color;
    ctx.globalAlpha = Math.max(0, p.life);
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
    ctx.fill();
  });
  ctx.globalAlpha = 1;

  debrisList.forEach(d => {
    ctx.save();
    ctx.translate(d.x, d.y);
    ctx.rotate(d.angle);
    ctx.fillStyle = d.color;
    ctx.fillRect(-d.size / 2, -d.size / 4, d.size, d.size / 2);
    ctx.restore();
  });
}

// Draw HUD
function drawHUD() {
  ctx.fillStyle = 'rgba(0, 40, 0, 0.6)';
  ctx.fillRect(10, 10, 200, 90);
  ctx.strokeStyle = '#00FF00';
  ctx.strokeRect(10, 10, 200, 90);

  ctx.fillStyle = '#00FF00';
  ctx.font = '12px monospace';
  ctx.fillText(`SPEED: ${(plane.speed * 20).toFixed(0)} knots`, 20, 30);
  ctx.fillText(`ALT  : ${plane.altitude} ft`, 20, 50);
  ctx.fillText(`PITCH: ${(-plane.angle * (180 / Math.PI)).toFixed(1)}°`, 20, 70);
  
  const gearStatus = plane.gearPosition === 1 ? 'DOWN' : plane.gearPosition === 0 ? 'UP' : 'MOVING...';
  ctx.fillText(`GEAR : ${gearStatus}`, 20, 90);

  if (plane.crashed) {
    ctx.fillStyle = 'rgba(200, 0, 0, 0.8)';
    ctx.fillRect(canvas.width / 2 - 150, canvas.height / 2 - 40, 300, 80);
    ctx.strokeStyle = '#FFF';
    ctx.strokeRect(canvas.width / 2 - 150, canvas.height / 2 - 40, 300, 80);

    ctx.fillStyle = '#FFF';
    ctx.font = '20px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('CRASH DESTROYED', canvas.width / 2, canvas.height / 2 - 5);
    ctx.font = '14px sans-serif';
    ctx.fillText('Press "R" to Restart', canvas.width / 2, canvas.height / 2 + 20);
    ctx.textAlign = 'left';
  }
}

// Loop
function loop() {
  update();
  drawEnvironment();
  drawA320();
  drawParticles();
  drawHUD();
  requestAnimationFrame(loop);
}

loop();
</script>

</body>
</html>

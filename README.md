const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const healthEl = document.getElementById('health');
const ammoEl = document.getElementById('ammo');
const scoreEl = document.getElementById('score');
const waveEl = document.getElementById('wave');
const overlay = document.getElementById('gameOverlay');
const startButton = document.getElementById('startButton');

const WORLD = { width: 2200, height: 1800 };

const keys = {};
const mouse = { x: canvas.width / 2, y: canvas.height / 2, down: false };

let gameStarted = false;
let gameOver = false;
let score = 0;
let wave = 1;
let nextWaveScore = 220;

const clamp = (value, min, max) => Math.min(Math.max(value, min), max);

function getPointerPosition(event) {
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  return {
    x: (event.clientX - rect.left) * scaleX,
    y: (event.clientY - rect.top) * scaleY,
  };
}

canvas.addEventListener('mousemove', (event) => {
  const pos = getPointerPosition(event);
  mouse.x = pos.x;
  mouse.y = pos.y;
});

canvas.addEventListener('mousedown', (event) => {
  mouse.down = true;
  event.preventDefault();
  if (gameStarted && !gameOver) {
    shoot();
  }
});

canvas.addEventListener('mouseup', () => {
  mouse.down = false;
});

canvas.addEventListener('mouseleave', () => {
  mouse.down = false;
});

window.addEventListener('keydown', (event) => {
  const key = event.key.toLowerCase();
  keys[key] = true;
  if (key === ' ') {
    event.preventDefault();
    if (gameStarted && !gameOver) {
      shoot();
    }
  }
  if (key === 'p') {
    if (gameStarted && !gameOver) {
      gameStarted = false;
      overlay.classList.add('active');
      overlay.querySelector('.panel').innerHTML = `
        <p class='eyebrow'>Pause</p>
        <h1>Pause</h1>
        <p>Reprends la partie quand tu veux.</p>
        <button id='resumeButton'>Reprendre</button>
      `;
      document.getElementById('resumeButton').addEventListener('click', () => {
        overlay.classList.remove('active');
        gameStarted = true;
      });
    }
  }
});

window.addEventListener('keyup', (event) => {
  keys[event.key.toLowerCase()] = false;
});

function resetGame() {
  gameStarted = true;
  gameOver = false;
  score = 0;
  wave = 1;
  nextWaveScore = 220;

  player.x = WORLD.width / 2;
  player.y = WORLD.height / 2;
  player.health = 100;
  player.maxHealth = 100;
  player.speed = 260;
  player.angle = 0;
  player.ammo = 12;
  player.maxAmmo = 30;
  player.fireCooldown = 0;
  player.reloadTime = 0;
  player.flash = 0;

  bullets.length = 0;
  enemies.length = 0;
  particles.length = 0;
  pickups.length = 0;
  enemySpawnTimer = 0.8;
  spawnEnemies(8);
  updateHUD();
  overlay.classList.remove('active');
}

function setOverlay(title, text, buttonText) {
  overlay.classList.add('active');
  overlay.querySelector('.panel').innerHTML = `
    <p class='eyebrow'>Mission</p>
    <h1>${title}</h1>
    <p>${text}</p>
    <button id='overlayAction'>${buttonText}</button>
  `;
  document.getElementById('overlayAction').addEventListener('click', () => {
    resetGame();
  });
}

startButton.addEventListener('click', () => {
  resetGame();
});

const player = {
  x: WORLD.width / 2,
  y: WORLD.height / 2,
  radius: 22,
  health: 100,
  maxHealth: 100,
  speed: 260,
  angle: 0,
  ammo: 12,
  maxAmmo: 30,
  fireCooldown: 0,
  reloadTime: 0,
  flash: 0,
};

const bullets = [];
const enemies = [];
const particles = [];
const pickups = [];

const obstacles = [
  { x: 340, y: 280, w: 200, h: 120 },
  { x: 740, y: 540, w: 240, h: 170 },
  { x: 1180, y: 300, w: 180, h: 260 },
  { x: 1420, y: 820, w: 220, h: 160 },
  { x: 900, y: 1160, w: 260, h: 180 },
  { x: 160, y: 1000, w: 220, h: 180 },
  { x: 1700, y: 980, w: 260, h: 200 },
  { x: 600, y: 1400, w: 220, h: 120 },
  { x: 1480, y: 1350, w: 250, h: 150 },
];

let enemySpawnTimer = 0.8;

function spawnEnemies(count) {
  for (let i = 0; i < count; i += 1) {
    let spawnX, spawnY, safe = false;
    for (let attempt = 0; attempt < 30 && !safe; attempt += 1) {
      spawnX = Math.random() * WORLD.width;
      spawnY = Math.random() * WORLD.height;
      safe = Math.hypot(spawnX - player.x, spawnY - player.y) > 220;
    }

    enemies.push({
      x: spawnX,
      y: spawnY,
      radius: 18,
      speed: 80 + Math.random() * 40 + wave * 8,
      maxHealth: 45 + wave * 8,
      health: 45 + wave * 8,
      hitCooldown: 0,
      color: Math.random() > 0.5 ? '#ff4d6d' : '#ff7d66',
    });
  }
}

function updateHUD() {
  healthEl.textContent = Math.max(0, Math.ceil(player.health));
  ammoEl.textContent = `${player.ammo} / ${player.maxAmmo}`;
  scoreEl.textContent = score;
  waveEl.textContent = wave;
}

function createParticles(x, y, color, count = 10) {
  for (let i = 0; i < count; i += 1) {
    const angle = Math.random() * Math.PI * 2;
    const speed = 40 + Math.random() * 90;
    particles.push({
      x,
      y,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed,
      life: 0.5 + Math.random() * 0.6,
      size: 2 + Math.random() * 4,
      color,
    });
  }
}

function spawnPickup(x, y) {
  const type = Math.random() > 0.4 ? 'ammo' : 'med';
  pickups.push({ x, y, radius: 12, type });
}

function shoot() {
  if (!gameStarted || gameOver) return;

  if (player.reloadTime > 0) return;
  if (player.fireCooldown > 0) return;

  if (player.ammo <= 0) {
    player.reloadTime = 1.3;
    return;
  }

  const angle = player.angle;
  const speed = 620;
  bullets.push({
    x: player.x + Math.cos(angle) * 24,
    y: player.y + Math.sin(angle) * 24,
    vx: Math.cos(angle) * speed,
    vy: Math.sin(angle) * speed,
    radius: 5,
    life: 1.1,
    damage: 26,
    color: '#7ae7ff',
  });

  player.ammo -= 1;
  player.fireCooldown = 0.12;
  player.flash = 0.08;

  createParticles(player.x + Math.cos(angle) * 30, player.y + Math.sin(angle) * 30, '#7ae7ff', 6);

  if (player.ammo <= 0) {
    player.reloadTime = 1.2;
  }

  updateHUD();
}

function reloadWeapon() {
  if (player.ammo >= player.maxAmmo || player.reloadTime > 0 || !gameStarted || gameOver) return;
  const needed = player.maxAmmo - player.ammo;
  if (needed <= 0) return;
  player.reloadTime = 1.0;
}

window.addEventListener('keydown', (event) => {
  if (event.key.toLowerCase() === 'r') {
    reloadWeapon();
  }
});

function rectCircleCollision(rect, circle) {
  const closestX = clamp(circle.x, rect.x, rect.x + rect.w);
  const closestY = clamp(circle.y, rect.y, rect.y + rect.h);
  const dx = circle.x - closestX;
  const dy = circle.y - closestY;
  return dx * dx + dy * dy < circle.radius * circle.radius;
}

function circleVsCircle(a, b, radiusSum) {
  return Math.hypot(a.x - b.x, a.y - b.y) < radiusSum;
}

function moveEntity(entity, dx, dy, radius) {
  const nextX = entity.x + dx;
  const nextY = entity.y + dy;

  const entityBox = { x: nextX - radius, y: nextY - radius, w: radius * 2, h: radius * 2 };
  let blocked = false;

  for (const obstacle of obstacles) {
    if (rectCircleCollision(obstacle, { x: nextX, y: nextY, radius })) {
      blocked = true;
      break;
    }
  }

  if (!blocked) {
    entity.x = clamp(nextX, radius, WORLD.width - radius);
    entity.y = clamp(nextY, radius, WORLD.height - radius);
  }
}

function update(delta) {
  if (!gameStarted || gameOver) return;

  const moveX = (keys['d'] || keys['arrowright'] ? 1 : 0) - (keys['a'] || keys['arrowleft'] ? 1 : 0);
  const moveY = (keys['s'] || keys['arrowdown'] ? 1 : 0) - (keys['w'] || keys['arrowup'] ? 1 : 0);

  let dx = 0;
  let dy = 0;

  if (moveX || moveY) {
    const len = Math.hypot(moveX, moveY) || 1;
    dx = (moveX / len) * player.speed * delta;
    dy = (moveY / len) * player.speed * delta;
  }

  moveEntity(player, dx, dy, player.radius);

  const angleToMouse = Math.atan2(mouse.y - canvas.height / 2, mouse.x - canvas.width / 2);
  player.angle = angleToMouse;

  if (player.fireCooldown > 0) player.fireCooldown -= delta;
  if (player.reloadTime > 0) {
    player.reloadTime -= delta;
    if (player.reloadTime <= 0) {
      const ammoNeeded = player.maxAmmo - player.ammo;
      player.ammo = player.maxAmmo;
      createParticles(player.x, player.y, '#6ef3a5', 18);
      updateHUD();
    }
  }

  if (mouse.down) {
    shoot();
  }

  for (let i = bullets.length - 1; i >= 0; i -= 1) {
    const bullet = bullets[i];
    bullet.x += bullet.vx * delta;
    bullet.y += bullet.vy * delta;
    bullet.life -= delta;

    if (bullet.life <= 0 || bullet.x < 0 || bullet.y < 0 || bullet.x > WORLD.width || bullet.y > WORLD.height) {
      bullets.splice(i, 1);
      continue;
    }

    let hit = false;
    for (let j = enemies.length - 1; j >= 0; j -= 1) {
      const enemy = enemies[j];
      if (Math.hypot(bullet.x - enemy.x, bullet.y - enemy.y) < bullet.radius + enemy.radius) {
        enemy.health -= bullet.damage;
        createParticles(bullet.x, bullet.y, '#7ae7ff', 10);
        bullets.splice(i, 1);
        hit = true;

        if (enemy.health <= 0) {
          score += 10;
          createParticles(enemy.x, enemy.y, '#ff4d6d', 18);
          if (Math.random() < 0.25) spawnPickup(enemy.x, enemy.y);
          enemies.splice(j, 1);
          if (score >= nextWaveScore) {
            wave += 1;
            nextWaveScore += 220;
            spawnEnemies(4 + wave);
            createParticles(player.x, player.y, '#6ef3a5', 30);
          }
        }
        break;
      }
    }

    if (hit) continue;

    for (const obstacle of obstacles) {
      if (rectCircleCollision(obstacle, { x: bullet.x, y: bullet.y, radius: bullet.radius })) {
        bullets.splice(i, 1);
        createParticles(bullet.x, bullet.y, '#fff2bf', 8);
        hit = true;
        break;
      }
    }
  }

  enemySpawnTimer -= delta;
  if (enemySpawnTimer <= 0) {
    spawnEnemies(1 + Math.min(3, wave));
    enemySpawnTimer = Math.max(0.45, 1.5 - wave * 0.08);
  }

  for (let i = enemies.length - 1; i >= 0; i -= 1) {
    const enemy = enemies[i];
    const dxToPlayer = player.x - enemy.x;
    const dyToPlayer = player.y - enemy.y;
    const distance = Math.hypot(dxToPlayer, dyToPlayer) || 1;

    const dirX = dxToPlayer / distance;
    const dirY = dyToPlayer / distance;

    let moveX = dirX * enemy.speed * delta;
    let moveY = dirY * enemy.speed * delta;

    const nextEnemy = { x: enemy.x + moveX, y: enemy.y + moveY, radius: enemy.radius };
    const collides = obstacles.some((obstacle) => rectCircleCollision(obstacle, nextEnemy));

    if (!collides) {
      enemy.x = clamp(nextEnemy.x, enemy.radius, WORLD.width - enemy.radius);
      enemy.y = clamp(nextEnemy.y, enemy.radius, WORLD.height - enemy.radius);
    }

    if (distance < player.radius + enemy.radius + 4) {
      player.health -= 20 * delta;
      createParticles(player.x, player.y, '#ff4d6d', 5);
      if (player.health <= 0) {
        player.health = 0;
        gameOver = true;
        gameStarted = false;
        setOverlay('Mission échouée', `Tu as marqué ${score} points. Recommence pour gagner davantage.`, 'Rejouer');
      }
    }
  }

  for (let i = pickups.length - 1; i >= 0; i -= 1) {
    const item = pickups[i];
    if (Math.hypot(item.x - player.x, item.y - player.y) < player.radius + item.radius) {
      if (item.type === 'ammo') {
        player.ammo = Math.min(player.maxAmmo, player.ammo + 8);
      } else {
        player.health = Math.min(player.maxHealth, player.health + 20);
      }
      createParticles(item.x, item.y, '#6ef3a5', 16);
      pickups.splice(i, 1);
      updateHUD();
    }
  }

  for (let i = particles.length - 1; i >= 0; i -= 1) {
    const particle = particles[i];
    particle.x += particle.vx * delta;
    particle.y += particle.vy * delta;
    particle.life -= delta;
    if (particle.life <= 0) particles.splice(i, 1);
  }

  updateHUD();
}

function drawBackground(cameraX, cameraY) {
  const sky = ctx.createLinearGradient(0, 0, 0, canvas.height);
  sky.addColorStop(0, '#07141d');
  sky.addColorStop(0.55, '#101f2c');
  sky.addColorStop(1, '#0d1c26');
  ctx.fillStyle = sky;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = 'rgba(122, 231, 255, 0.06)';
  for (let i = 0; i < 120; i += 1) {
    const x = ((i * 137) % (WORLD.width + 600)) - cameraX * 0.18;
    const y = ((i * 79) % (WORLD.height + 300)) - cameraY * 0.18;
    ctx.fillRect(x % canvas.width, y % canvas.height, 2, 2);
  }

  ctx.strokeStyle = 'rgba(122, 231, 255, 0.08)';
  ctx.lineWidth = 1;
  for (let x = -100; x < canvas.width + 100; x += 80) {
    ctx.beginPath();
    ctx.moveTo(x, 0);
    ctx.lineTo(x + 140, canvas.height);
    ctx.stroke();
  }

  for (let y = -100; y < canvas.height + 100; y += 80) {
    ctx.beginPath();
    ctx.moveTo(0, y);
    ctx.lineTo(canvas.width, y + 80);
    ctx.stroke();
  }
}

function drawObstacle(obstacle) {
  const x = obstacle.x - camera.x;
  const y = obstacle.y - camera.y;
  ctx.fillStyle = '#1d2937';
  ctx.fillRect(x, y, obstacle.w, obstacle.h);

  ctx.fillStyle = 'rgba(122, 231, 255, 0.08)';
  ctx.fillRect(x + 8, y + 8, obstacle.w - 16, obstacle.h - 16);

  ctx.strokeStyle = 'rgba(255, 255, 255, 0.07)';
  ctx.strokeRect(x, y, obstacle.w, obstacle.h);
}

function drawPickup(item) {
  const x = item.x - camera.x;
  const y = item.y - camera.y;

  ctx.save();
  ctx.translate(x, y);
  ctx.fillStyle = item.type === 'ammo' ? '#7ae7ff' : '#6ef3a5';
  ctx.beginPath();
  ctx.arc(0, 0, item.radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#03181c';
  ctx.font = 'bold 12px sans-serif';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(item.type === 'ammo' ? 'A' : '+', 0, 1);
  ctx.restore();
}

function drawPlayer() {
  const x = player.x - camera.x;
  const y = player.y - camera.y;

  ctx.save();
  ctx.translate(x, y);
  ctx.rotate(player.angle);

  ctx.fillStyle = 'rgba(0, 0, 0, 0.35)';
  ctx.beginPath();
  ctx.ellipse(0, 10, 18, 12, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#dfeaf5';
  ctx.beginPath();
  ctx.arc(0, 0, player.radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#1b2733';
  ctx.fillRect(12, -3, 28, 6);

  ctx.fillStyle = '#ffca6b';
  ctx.fillRect(32, -5, 14, 10);

  ctx.fillStyle = '#ff4d6d';
  ctx.fillRect(-14, -2, 10, 4);

  ctx.restore();

  if (player.flash > 0) {
    ctx.fillStyle = `rgba(122, 231, 255, ${player.flash * 10})`;
    ctx.beginPath();
    ctx.arc(x, y, player.radius + 12, 0, Math.PI * 2);
    ctx.fill();
  }
}

function drawEnemy(enemy) {
  const x = enemy.x - camera.x;
  const y = enemy.y - camera.y;

  ctx.fillStyle = 'rgba(0, 0, 0, 0.25)';
  ctx.beginPath();
  ctx.ellipse(x, y + 14, enemy.radius + 4, enemy.radius + 2, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = enemy.color;
  ctx.beginPath();
  ctx.arc(x, y, enemy.radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#fff';
  ctx.fillRect(x - 18, y - enemy.radius - 12, 36, 5);
  ctx.fillStyle = '#6ef3a5';
  ctx.fillRect(x - 18, y - enemy.radius - 12, (enemy.health / enemy.maxHealth) * 36, 5);
}

function drawBullet(bullet) {
  ctx.fillStyle = bullet.color;
  ctx.beginPath();
  ctx.arc(bullet.x - camera.x, bullet.y - camera.y, bullet.radius, 0, Math.PI * 2);
  ctx.fill();
}

function drawParticles() {
  for (const particle of particles) {
    ctx.fillStyle = particle.color;
    ctx.globalAlpha = clamp(particle.life * 1.2, 0, 1);
    ctx.fillRect(particle.x - camera.x, particle.y - camera.y, particle.size, particle.size);
    ctx.globalAlpha = 1;
  }
}

function drawCrosshair() {
  ctx.strokeStyle = 'rgba(122, 231, 255, 0.9)';
  ctx.lineWidth = 1.5;
  const cx = mouse.x;
  const cy = mouse.y;
  ctx.beginPath();
  ctx.moveTo(cx - 10, cy);
  ctx.lineTo(cx + 10, cy);
  ctx.moveTo(cx, cy - 10);
  ctx.lineTo(cx, cy + 10);
  ctx.stroke();
}

let camera = { x: 0, y: 0 };
let lastTime = 0;

function render() {
  const currentTime = performance.now();
  const delta = Math.min((currentTime - lastTime) / 1000 || 0.016, 0.033);
  lastTime = currentTime;

  update(delta);

  camera.x = clamp(player.x - canvas.width / 2, 0, WORLD.width - canvas.width);
  camera.y = clamp(player.y - canvas.height / 2, 0, WORLD.height - canvas.height);

  drawBackground(camera.x, camera.y);

  for (const obstacle of obstacles) {
    drawObstacle(obstacle);
  }

  for (const pickup of pickups) {
    drawPickup(pickup);
  }

  for (const bullet of bullets) {
    drawBullet(bullet);
  }

  for (const enemy of enemies) {
    drawEnemy(enemy);
  }

  drawPlayer();
  drawParticles();
  drawCrosshair();

  requestAnimationFrame(render);
}

resetGame();
render();

window.addEventListener('blur', () => {
  keys.w = false;
  keys.a = false;
  keys.s = false;
  keys.d = false;
  mouse.down = false;
});

setOverlay('Night City Survival', 'Déplace-toi avec ZQSD, vise avec la souris et élimine les ennemis.', 'Commencer');

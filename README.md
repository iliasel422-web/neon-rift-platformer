const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const playerHealthFill = document.getElementById('playerHealthFill');
const enemyHealthFill = document.getElementById('enemyHealthFill');
const timerLabel = document.getElementById('timer');
const roundLabel = document.getElementById('roundLabel');

const arena = {
  width: canvas.width,
  height: canvas.height,
  floorY: 420,
};

const keys = {};

const state = {
  timer: 60,
  round: 1,
  gameOver: false,
  lastTime: 0,
};

const hitFlash = {
  player: 0,
  enemy: 0,
};

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function fighterFactory(config) {
  return {
    ...config,
    x: config.x,
    y: config.y,
    width: 48,
    height: 138,
    vx: 0,
    vy: 0,
    facing: config.facing || 1,
    attack: null,
    attackTimer: 0,
    attackCooldown: 0,
    block: false,
    health: 100,
    grounded: true,
    hitStun: 0,
    isAI: config.isAI || false,
    aiTimer: 0,
    combo: 0,
  };
}

const player = fighterFactory({
  name: 'YOU',
  x: 220,
  y: arena.floorY - 138,
  color: '#5ee9ff',
  facing: 1,
  isAI: false,
});

const enemy = fighterFactory({
  name: 'RIVAL',
  x: 740,
  y: arena.floorY - 138,
  color: '#ff5db1',
  facing: -1,
  isAI: true,
});

function resetRound() {
  player.x = 220;
  player.y = arena.floorY - player.height;
  player.vx = 0;
  player.vy = 0;
  player.health = 100;
  player.attack = null;
  player.attackTimer = 0;
  player.attackCooldown = 0;
  player.block = false;
  player.hitStun = 0;
  player.grounded = true;
  player.combo = 0;

  enemy.x = 740;
  enemy.y = arena.floorY - enemy.height;
  enemy.vx = 0;
  enemy.vy = 0;
  enemy.health = 100;
  enemy.attack = null;
  enemy.attackTimer = 0;
  enemy.attackCooldown = 0;
  enemy.block = false;
  enemy.hitStun = 0;
  enemy.grounded = true;
  enemy.combo = 0;

  state.gameOver = false;
  state.timer = 60;
  roundLabel.textContent = `ROUND ${state.round}`;
  hitFlash.player = 0;
  hitFlash.enemy = 0;
}

function startAttack(fighter, type) {
  if (!fighter || fighter.attack || fighter.attackCooldown > 0 || fighter.hitStun > 0) {
    return;
  }

  fighter.attack = type;
  fighter.attackTimer = type === 'punch' ? 0.16 : 0.22;
  fighter.attackCooldown = type === 'punch' ? 0.28 : 0.4;
}

function doHit(fighter, opponent, damage, knockback) {
  if (fighter.attack && fighter.attackTimer > 0.04 && !opponent.hitStun) {
    const blocked = opponent.block && opponent.facing === -fighter.facing;
    const finalDamage = blocked ? Math.round(damage * 0.35) : damage;

    opponent.health = clamp(opponent.health - finalDamage, 0, 100);
    opponent.hitStun = 0.22;
    opponent.vx = fighter.facing * knockback;
    opponent.vy = -4;

    if (blocked) {
      opponent.vx *= 0.5;
    }

    if (opponent === player) {
      hitFlash.player = 0.18;
    } else {
      hitFlash.enemy = 0.18;
    }

    fighter.combo += 1;
    fighter.attack = null;
    fighter.attackTimer = 0;
  }
}

function handleAttackCollision(attacker, defender) {
  if (!attacker.attack || attacker.attackTimer <= 0.04) {
    return;
  }

  const attackDistance = attacker.attack === 'punch' ? 64 : 92;
  const attackHeight = 30;
  const dx = Math.abs(attacker.x - defender.x);
  const dy = Math.abs(attacker.y - defender.y);
  const inFront = Math.sign(defender.x - attacker.x) === attacker.facing;

  if (inFront && dx < attackDistance && dy < attackHeight && defender.hitStun <= 0) {
    if (attacker.attack === 'punch') {
      doHit(attacker, defender, 10, 8.5);
    } else {
      doHit(attacker, defender, 14, 11.5);
    }
  }
}

function updateFighter(fighter, dt, isPlayer) {
  fighter.attackCooldown = Math.max(0, fighter.attackCooldown - dt);
  fighter.hitStun = Math.max(0, fighter.hitStun - dt);

  if (fighter.attack) {
    fighter.attackTimer = Math.max(0, fighter.attackTimer - dt);
    if (fighter.attackTimer <= 0) {
      fighter.attack = null;
    }
  }

  fighter.block = isPlayer ? keys.s : fighter.block;

  if (isPlayer) {
    const moveLeft = keys.a;
    const moveRight = keys.d;
    const jump = keys.w;

    let direction = 0;
    if (moveLeft) direction -= 1;
    if (moveRight) direction += 1;

    fighter.vx += direction * 0.8;
    fighter.vx *= fighter.grounded ? 0.8 : 0.94;
    fighter.vx = clamp(fighter.vx, -4.6, 4.6);

    if (jump && fighter.grounded) {
      fighter.vy = -14;
      fighter.grounded = false;
    }

    if (fighter.attack === null && direction !== 0) {
      fighter.facing = direction > 0 ? 1 : -1;
    }

    if (keys.j && !fighter.attack) {
      startAttack(fighter, 'punch');
    }

    if (keys.k && !fighter.attack) {
      startAttack(fighter, 'kick');
    }
  } else {
    const dx = player.x - fighter.x;
    const distance = Math.abs(dx);

    fighter.aiTimer -= dt;

    if (fighter.aiTimer <= 0) {
      fighter.aiTimer = 0.15 + Math.random() * 0.35;

      if (distance < 90) {
        if (Math.random() < 0.35) {
          startAttack(fighter, 'punch');
        } else if (Math.random() < 0.18) {
          startAttack(fighter, 'kick');
        }
      } else if (distance > 140) {
        fighter.vx += Math.sign(dx) * 1.4;
      }

      if (distance < 100 && Math.random() < 0.2) {
        fighter.block = true;
      } else {
        fighter.block = false;
      }
    }

    fighter.vx *= fighter.grounded ? 0.88 : 0.96;
    fighter.vx = clamp(fighter.vx, -4.5, 4.5);

    if (distance !== 0) {
      fighter.facing = dx > 0 ? 1 : -1;
    }
  }

  fighter.vy += 0.7;
  fighter.x += fighter.vx * 60 * dt;
  fighter.y += fighter.vy * 60 * dt;

  const minX = 30;
  const maxX = arena.width - 30;
  fighter.x = clamp(fighter.x, minX, maxX);

  if (fighter.y >= arena.floorY - fighter.height) {
    fighter.y = arena.floorY - fighter.height;
    fighter.vy = 0;
    fighter.grounded = true;
  } else {
    fighter.grounded = false;
  }
}

function checkWinner() {
  if (player.health <= 0) {
    state.gameOver = true;
    roundLabel.textContent = 'RIVAL WINS';
    return true;
  }

  if (enemy.health <= 0) {
    state.gameOver = true;
    roundLabel.textContent = 'YOU WIN';
    return true;
  }

  return false;
}

function updateGame(dt) {
  if (!state.gameOver) {
    state.timer = Math.max(0, state.timer - dt);
    if (state.timer <= 0) {
      state.gameOver = true;
      if (player.health > enemy.health) {
        roundLabel.textContent = 'YOU WIN';
      } else if (enemy.health > player.health) {
        roundLabel.textContent = 'RIVAL WINS';
      } else {
        roundLabel.textContent = 'DRAW';
      }
    }
  }

  updateFighter(player, dt, true);
  updateFighter(enemy, dt, false);

  handleAttackCollision(player, enemy);
  handleAttackCollision(enemy, player);

  if (!state.gameOver) {
    checkWinner();
  }

  if (state.gameOver && keys.r) {
    state.round += 1;
    resetRound();
  }

  if (checkWinner()) {
    if (keys.r) {
      state.round += 1;
      resetRound();
    }
  }

  playerHealthFill.style.width = `${player.health}%`;
  enemyHealthFill.style.width = `${enemy.health}%`;
  timerLabel.textContent = state.timer.toFixed(1);
}

function drawArena() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = '#0b1420';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
  gradient.addColorStop(0, '#0d1d2c');
  gradient.addColorStop(1, '#101420');
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.strokeStyle = 'rgba(94, 233, 255, 0.3)';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(30, arena.floorY);
  ctx.lineTo(canvas.width - 30, arena.floorY);
  ctx.stroke();

  for (let x = 40; x < canvas.width - 40; x += 60) {
    ctx.fillStyle = 'rgba(94, 233, 255, 0.12)';
    ctx.fillRect(x, arena.floorY + 18, 18, 5);
  }

  ctx.fillStyle = 'rgba(255, 255, 255, 0.06)';
  ctx.fillRect(0, 0, canvas.width, 120);
}

function drawFighter(fighter) {
  const bodyX = fighter.x - fighter.width / 2;
  const bodyY = fighter.y;

  ctx.save();
  ctx.translate(bodyX + fighter.width / 2, bodyY + fighter.height / 2);
  ctx.scale(fighter.facing, 1);
  ctx.translate(-(bodyX + fighter.width / 2), -(bodyY + fighter.height / 2));

  if (fighter.hitStun > 0) {
    ctx.shadowColor = fighter.color;
    ctx.shadowBlur = 18;
  }

  ctx.fillStyle = fighter.color;
  ctx.fillRect(bodyX, bodyY + 18, fighter.width, fighter.height - 18);

  ctx.fillStyle = '#d8f4ff';
  ctx.fillRect(bodyX + 10, bodyY + 12, 28, 18);

  if (fighter.block) {
    ctx.fillStyle = 'rgba(255,255,255,0.5)';
    ctx.fillRect(bodyX - 14, bodyY + 24, 8, fighter.height - 34);
    ctx.fillRect(bodyX + fighter.width + 6, bodyY + 24, 8, fighter.height - 34);
  }

  if (fighter.attack) {
    ctx.fillStyle = fighter.attack === 'punch' ? 'rgba(254, 210, 102, 0.88)' : 'rgba(255, 93, 177, 0.88)';
    const range = fighter.attack === 'punch' ? 62 : 94;
    const attackX = fighter.facing > 0 ? bodyX + fighter.width : bodyX - range;
    ctx.fillRect(attackX, bodyY + 42, range, 44);
  }

  ctx.restore();

  if (fighter === player && hitFlash.player > 0) {
    ctx.fillStyle = `rgba(255,255,255,${hitFlash.player * 4})`;
    ctx.fillRect(bodyX - 4, bodyY - 8, fighter.width + 8, fighter.height + 12);
  }

  if (fighter === enemy && hitFlash.enemy > 0) {
    ctx.fillStyle = `rgba(255,255,255,${hitFlash.enemy * 4})`;
    ctx.fillRect(bodyX - 4, bodyY - 8, fighter.width + 8, fighter.height + 12);
  }
}

function drawUIText() {
  ctx.fillStyle = 'rgba(238,247,255,0.9)';
  ctx.font = 'bold 18px Arial';
  ctx.fillText('YOU', 34, 64);
  ctx.fillText('RIVAL', canvas.width - 126, 64);

  if (state.gameOver) {
    ctx.fillStyle = 'rgba(7, 13, 21, 0.55)';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    ctx.fillStyle = '#eef7ff';
    ctx.font = 'bold 52px Arial';
    const text = roundLabel.textContent;
    const width = ctx.measureText(text).width;
    ctx.fillText(text, (canvas.width - width) / 2, canvas.height / 2 - 10);

    ctx.font = 'bold 20px Arial';
    const sub = 'Press R to reset';
    const subWidth = ctx.measureText(sub).width;
    ctx.fillText(sub, (canvas.width - subWidth) / 2, canvas.height / 2 + 30);
  }
}

function updateVisuals() {
  hitFlash.player = Math.max(0, hitFlash.player - 0.016);
  hitFlash.enemy = Math.max(0, hitFlash.enemy - 0.016);
}

function gameLoop(timestamp) {
  const dt = Math.min(0.033, (timestamp - state.lastTime) / 1000 || 0.016);
  state.lastTime = timestamp;

  updateGame(dt);
  updateVisuals();

  drawArena();
  drawFighter(player);
  drawFighter(enemy);
  drawUIText();

  requestAnimationFrame(gameLoop);
}

window.addEventListener('keydown', (event) => {
  const key = event.key.toLowerCase();
  keys[key] = true;

  if (key === 'j') {
    startAttack(player, 'punch');
  }
  if (key === 'k') {
    startAttack(player, 'kick');
  }

  if (key === 'r') {
    state.round += 1;
    resetRound();
  }
});

window.addEventListener('keyup', (event) => {
  const key = event.key.toLowerCase();
  keys[key] = false;
});

resetRound();
requestAnimationFrame(gameLoop);

<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>School 21 Space Game Jam</title>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap');

        :root {
            --neon-blue: #00C8FF;
            --neon-green: #00FF85;
            --neon-purple: #D400FF;
            --neon-orange: #FFA600;
            --neon-gold: #FFD300;
            --neon-red: #FF0033;
            --neon-magenta: #FF00FF;
            --neon-cyan: #00FFFF;
            --bg-color: #050505;
        }

        body {
            margin: 0;
            padding: 0;
            background-color: #000;
            overflow: hidden;
            font-family: 'Press Start 2P', cursive;
            color: #fff;
            user-select: none;
        }

        #game-container {
            position: relative;
            width: 100vw;
            height: 100vh;
        }

        canvas {
            display: block;
            background-color: var(--bg-color);
        }

        /* Scanline Effect */
        .scanlines {
            position: absolute;
            top: 0; left: 0; right: 0; bottom: 0;
            background: linear-gradient(rgba(18, 16, 16, 0) 50%, rgba(0, 0, 0, 0.25) 50%), linear-gradient(90deg, rgba(255, 0, 0, 0.06), rgba(0, 255, 0, 0.02), rgba(0, 0, 255, 0.06));
            background-size: 100% 2px, 3px 100%;
            pointer-events: none;
            z-index: 2;
        }

        /* UI */
        #ui-layer {
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            pointer-events: none;
            z-index: 10;
        }

        .top-controls {
            position: absolute;
            top: 20px; right: 20px;
            pointer-events: auto;
            display: flex; gap: 10px;
        }

        .btn-ui {
            background: rgba(0,0,0,0.5);
            border: 1px solid var(--neon-blue);
            color: var(--neon-blue);
            padding: 8px 12px;
            font-family: 'Press Start 2P', cursive;
            font-size: 10px;
            cursor: pointer;
        }
        .btn-ui:hover { background: var(--neon-blue); color: #000; }

        /* Menus */
        .menu-overlay {
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.9);
            display: flex; flex-direction: column;
            justify-content: center; align-items: center;
            z-index: 20;
            pointer-events: auto;
        }

        h1 {
            font-size: 40px;
            color: #fff;
            text-shadow: 4px 4px 0px var(--neon-purple);
            margin-bottom: 40px;
            text-align: center;
            line-height: 1.5;
        }

        .btn-menu {
            background: transparent;
            border: 2px solid var(--neon-green);
            color: var(--neon-green);
            padding: 15px 30px;
            margin: 10px;
            font-family: 'Press Start 2P';
            font-size: 16px;
            cursor: pointer;
            transition: 0.2s;
        }
        .btn-menu:hover { background: var(--neon-green); color: #000; box-shadow: 0 0 15px var(--neon-green); }

        .hidden { display: none !important; }
        
        .keys-hint { margin-top: 30px; font-size: 12px; color: #888; text-align: center; line-height: 1.8; }
    </style>
</head>
<body>

<div id="game-container">
    <div class="scanlines"></div>
    <canvas id="gameCanvas"></canvas>

    <div id="ui-layer">
        <div class="top-controls">
            <button class="btn-ui" onclick="toggleSound()">MUTE</button>
            <button class="btn-ui" onclick="goToMenu()">MENU</button>
        </div>
    </div>

    <!-- MAIN MENU -->
    <div id="menu-main" class="menu-overlay">
        <h1>SPACE JAM<br>SCHOOL 21</h1>
        <button class="btn-menu" onclick="initGame('single')">SINGLE PLAYER</button>
        <button class="btn-menu" onclick="initGame('multi')">2 PLAYERS (SPLIT)</button>
        <div class="keys-hint">
            P1: WASD + K (Shoot)<br>
            P2: ARROWS + NUMPAD 5 (Shoot)<br>
            'Z' for Nuke
        </div>
    </div>

    <!-- GAME OVER -->
    <div id="menu-gameover" class="menu-overlay hidden">
        <h1 style="color: var(--neon-red)">GAME OVER</h1>
        <div id="score-final" style="margin-bottom:20px; font-size:18px;"></div>
        <button class="btn-menu" onclick="initGame(lastMode)">RETRY</button>
        <button class="btn-menu" onclick="goToMenu()">MAIN MENU</button>
    </div>
</div>

<script>
/**
 * 1. CONSTANTS & UTILS
 */
const COLORS = {
    blue: '#00C8FF', green: '#00FF85', purple: '#D400FF', orange: '#FFA600',
    gold: '#FFD300', red: '#FF0033', white: '#FFFFFF', magenta: '#FF00FF', cyan: '#00FFFF'
};

const ASTEROID_TYPES = {
    SMALL: { hp: 1, speed: 2.0, size: 10, color: COLORS.white, score: 10 },
    MEDIUM: { hp: 3, speed: 1.2, size: 20, color: COLORS.blue, score: 30 },
    LARGE: { hp: 5, speed: 0.8, size: 30, color: COLORS.purple, score: 50 },
    GREEN: { hp: 1, speed: 2.5, size: 15, color: COLORS.green, score: 100, drop: 'green' },
    DIAMOND: { hp: 7, speed: 0.7, size: 25, color: COLORS.blue, score: 200, drop: 'diamond' }
};

// Global Inputs
const keys = {
    p1: { u:false, d:false, l:false, r:false, fire:false },
    p2: { u:false, d:false, l:false, r:false, fire:false }
};

document.addEventListener('keydown', e => {
    const c = e.code;
    // P1
    if(c==='KeyW') keys.p1.u=true; if(c==='KeyS') keys.p1.d=true;
    if(c==='KeyA') keys.p1.l=true; if(c==='KeyD') keys.p1.r=true;
    if(c==='KeyK') keys.p1.fire=true;
    // P2
    if(c==='ArrowUp') keys.p2.u=true; if(c==='ArrowDown') keys.p2.d=true;
    if(c==='ArrowLeft') keys.p2.l=true; if(c==='ArrowRight') keys.p2.r=true;
    if(c==='Numpad5' || c==='Digit5') keys.p2.fire=true;
    // Nuke
    if(c==='KeyZ') { if(window.game) window.game.triggerNuke(); }
});

document.addEventListener('keyup', e => {
    const c = e.code;
    if(c==='KeyW') keys.p1.u=false; if(c==='KeyS') keys.p1.d=false;
    if(c==='KeyA') keys.p1.l=false; if(c==='KeyD') keys.p1.r=false;
    if(c==='KeyK') keys.p1.fire=false;
    if(c==='ArrowUp') keys.p2.u=false; if(c==='ArrowDown') keys.p2.d=false;
    if(c==='ArrowLeft') keys.p2.l=false; if(c==='ArrowRight') keys.p2.r=false;
    if(c==='Numpad5' || c==='Digit5') keys.p2.fire=false;
});

/**
 * 2. AUDIO SYSTEM
 */
const AudioSys = {
    ctx: null, muted: false,
    init() {
        if(!this.ctx) {
            try { this.ctx = new (window.AudioContext || window.webkitAudioContext)(); } 
            catch(e) { console.log('Audio init failed'); }
        }
        if(this.ctx && this.ctx.state === 'suspended') this.ctx.resume();
    },
    playTone(freq, type, dur, vol=0.1) {
        if(!this.ctx || this.muted) return;
        const o = this.ctx.createOscillator();
        const g = this.ctx.createGain();
        o.type = type; o.frequency.value = freq;
        g.gain.setValueAtTime(vol, this.ctx.currentTime);
        g.gain.exponentialRampToValueAtTime(0.01, this.ctx.currentTime + dur);
        o.connect(g); g.connect(this.ctx.destination);
        o.start(); o.stop(this.ctx.currentTime + dur);
    },
    shoot() { this.playTone(400, 'square', 0.1); },
    hit() { this.playTone(100, 'sawtooth', 0.1); },
    boom() { this.playTone(50, 'sawtooth', 0.3, 0.2); },
    powerup() { this.playTone(600, 'sine', 0.2); },
    boss() { this.playTone(80, 'square', 0.2, 0.2); }
};

/**
 * 3. GAME CLASSES
 */

class Particle {
    constructor(x, y, color) {
        this.x = x; this.y = y; this.color = color;
        this.vx = (Math.random()-0.5)*100; this.vy = (Math.random()-0.5)*100;
        this.life = 1.0; this.active = true;
    }
    update(dt) {
        this.x += this.vx*dt; this.y += this.vy*dt;
        this.life -= dt*2;
        if(this.life<=0) this.active = false;
    }
    draw(ctx) {
        ctx.globalAlpha = this.life;
        ctx.fillStyle = this.color;
        ctx.fillRect(this.x, this.y, 3, 3);
        ctx.globalAlpha = 1.0;
    }
}

class Projectile {
    constructor(x, y, vy, color, type='normal', isEnemy=false) {
        this.x = x; this.y = y; this.vx = 0; this.vy = vy;
        this.color = color; this.type = type;
        this.active = true; this.radius = 4;
        this.damage = 1;
        this.isEnemy = isEnemy;
        
        // Special props for Laser and Bomb (Nuke/Ricochet/Laser)
        this.isLaser = false;
        this.life = 1.0; // Life is primarily used by Laser
    }
    update(dt, w, h) {
        if(this.isLaser) {
            // Лазер не движется, просто существует короткое время
            this.life -= dt;
            if(this.life <= 0) this.active = false;
        } else {
            this.x += this.vx * dt;
            this.y += this.vy * dt;
            if(this.y < -50 || this.y > h+50) this.active = false;
        }
    }
    draw(ctx) {
        ctx.shadowBlur = 10;
        ctx.shadowColor = this.color;
        
        if(this.isLaser) {
            // Лазерный луч рисуется от точки выстрела (this.y) до верха мира (0).
            const beamHeight = this.y; 
            
            ctx.globalAlpha = this.life;
            
            // Outer beam (Wider)
            ctx.fillStyle = this.color;
            ctx.fillRect(this.x - 4, 0, 8, beamHeight); 
            
            // Core beam (Narrower, brighter)
            ctx.fillStyle = COLORS.white;
            ctx.fillRect(this.x - 2, 0, 4, beamHeight); 
            
            ctx.globalAlpha = 1.0;
        } else {
            ctx.fillStyle = this.color;
            ctx.beginPath(); ctx.arc(this.x, this.y, this.radius, 0, Math.PI*2); ctx.fill();
        }
        ctx.shadowBlur = 0;
    }
}

class PowerUp {
    constructor(x, y, type) {
        this.x = x; this.y = y; this.type = type; 
        this.active = true; this.radius = 15; this.vy = 80;
        this.color = COLORS.white;
        
        // Color coding
        if(this.type === 'heal') this.color = COLORS.magenta;
        else if(this.type === 'nuke') this.color = COLORS.red; 
        else if(this.type === 'multi') this.color = COLORS.green;
        else if(this.type === 'drone') this.color = COLORS.cyan;
        else if(this.type === 'laser') this.color = COLORS.purple;
        else if(this.type === 'shield') this.color = COLORS.gold;
        else if(this.type === 'speed') this.color = COLORS.orange; 
        else this.color = COLORS.orange;
    }
    update(dt, h) {
        this.y += this.vy * dt;
        if(this.y > h + 50) this.active = false;
    }
    draw(ctx) {
        ctx.save();
        ctx.translate(this.x, this.y);
        ctx.shadowBlur = 10;
        ctx.shadowColor = this.color;
        ctx.strokeStyle = this.color;
        ctx.lineWidth = 2;
        
        // Box
        ctx.strokeRect(-10, -10, 20, 20);
        
        // Text
        ctx.fillStyle = '#fff';
        ctx.font = '10px "Press Start 2P"';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        
        let txt = '?';
        if(this.type==='heal') txt='21';
        else if(this.type==='nuke') txt='Z';
        else if(this.type==='multi') txt='M';
        else if(this.type==='drone') txt='D';
        else if(this.type==='laser') txt='L';
        else if(this.type==='shield') txt='S';
        else if(this.type==='speed') txt='SP';
        
        ctx.fillText(txt, 0, 0);
        ctx.restore();
    }
}

class Entity {
    constructor(x, y, typeKey) {
        this.x = x; this.y = y; 
        const conf = ASTEROID_TYPES[typeKey] || ASTEROID_TYPES.SMALL;
        this.hp = conf.hp; this.maxHp = conf.hp;
        this.color = conf.color; this.score = conf.score;
        this.drop = conf.drop;
        this.radius = conf.size;
        this.active = true;
        this.vy = conf.speed * 60; 
        this.angle = 0;
        this.rot = (Math.random()-0.5);
        this.typeKey = typeKey;

        this.points = [];
        const sides = typeKey === 'DIAMOND' ? 4 : 6 + Math.floor(Math.random() * 4);
        for(let i=0; i<sides; i++) {
            const r = this.radius * (0.8 + Math.random() * 0.4); 
            const a = (i / sides) * Math.PI * 2;
            this.points.push({x: Math.cos(a) * r, y: Math.sin(a) * r});
        }
    }
    update(dt, h) {
        this.y += this.vy * dt;
        this.angle += this.rot * dt;
        if(this.y > h + 50) this.active = false;
    }
    draw(ctx) {
        ctx.save();
        ctx.translate(this.x, this.y);
        ctx.rotate(this.angle);
        ctx.shadowBlur = 10;
        ctx.shadowColor = this.color;
        ctx.strokeStyle = this.color;
        ctx.lineWidth = 2;
        
        if (this.typeKey === 'DIAMOND') {
            ctx.beginPath(); ctx.moveTo(0, -this.radius); ctx.lineTo(this.radius*0.7, 0); ctx.lineTo(0, this.radius); ctx.lineTo(-this.radius*0.7, 0); ctx.closePath(); ctx.stroke();
            ctx.beginPath(); ctx.moveTo(0, -this.radius*0.5); ctx.lineTo(this.radius*0.3, 0); ctx.lineTo(0, this.radius*0.5); ctx.lineTo(-this.radius*0.3, 0); ctx.closePath(); ctx.stroke();
        } else {
            ctx.beginPath(); ctx.moveTo(this.points[0].x, this.points[0].y);
            for(let i=1; i<this.points.length; i++) { ctx.lineTo(this.points[i].x, this.points[i].y); }
            ctx.closePath(); ctx.stroke();
            if(this.radius > 20) {
                 ctx.beginPath(); ctx.moveTo(this.points[0].x * 0.4, this.points[0].y * 0.4); ctx.lineTo(this.points[2].x * 0.4, this.points[2].y * 0.4); ctx.stroke();
            }
        }
        ctx.restore();
    }
}

class Boss {
    constructor(w) {
        this.x = w/2; this.y = -150;
        this.w = w;
        this.hp = 1500; this.maxHp = 1500;
        this.active = true;
        this.state = 'enter'; 
        this.timer = 0;
        this.cooldown = 0;
    }
    update(dt) {
        this.timer += dt;
        if(this.state === 'enter') {
            this.y += 80 * dt;
            if(this.y > 100) this.state = 'fight';
        } else {
            this.x = (this.w/2) + Math.sin(this.timer) * (this.w/3);
            this.cooldown -= dt;
        }
    }
    draw(ctx) {
        ctx.save();
        ctx.translate(this.x, this.y);
        const mainColor = COLORS.purple;
        ctx.shadowBlur = 20; ctx.shadowColor = mainColor; ctx.strokeStyle = mainColor; ctx.lineWidth = 3; ctx.fillStyle = '#050505';
        ctx.beginPath(); ctx.moveTo(0, 100); ctx.lineTo(30, 0); ctx.lineTo(80, -40); ctx.lineTo(120, 20); ctx.lineTo(80, 40); ctx.lineTo(40, 20); ctx.lineTo(20, -60); ctx.lineTo(-20, -60); ctx.lineTo(-40, 20); ctx.lineTo(-80, 40); ctx.lineTo(-120, 20); ctx.lineTo(-80, -40); ctx.lineTo(-30, 0); ctx.lineTo(0, 100); ctx.closePath(); ctx.fill(); ctx.stroke();
        ctx.fillStyle = COLORS.orange; ctx.shadowBlur = 30; ctx.shadowColor = COLORS.orange; ctx.beginPath(); ctx.arc(-25, -65, 8, 0, Math.PI*2); ctx.arc(25, -65, 8, 0, Math.PI*2); ctx.fill();
        ctx.fillStyle = COLORS.red; ctx.shadowColor = COLORS.red; ctx.beginPath(); ctx.arc(0, 0, 15 + Math.sin(this.timer*10)*3, 0, Math.PI*2); ctx.fill();
        ctx.restore();
    }
}

class Player {
    constructor(x, y, inputKey) {
        this.x = x; this.y = y;
        this.inputKey = inputKey; 
        this.radius = 15;
        this.hp = 3; this.maxHp = 3;
        this.cooldown = 0;
        this.droneTimer = 0;
        this.shield = 0; 

        this.buffs = { multi:0, laser:0, drone:0, speed:0 }; 
    }
    update(dt, w, h) {
        const speedMultiplier = this.buffs.speed > 0 ? 1.5 : 1;
        const speed = 300 * speedMultiplier;

        const inp = keys[this.inputKey];
        if(inp.l) this.x -= speed * dt; if(inp.r) this.x += speed * dt; if(inp.u) this.y -= speed * dt; if(inp.d) this.y += speed * dt;
        this.x = Math.max(20, Math.min(w-20, this.x)); this.y = Math.max(20, Math.min(h-20, this.y));
        
        if(this.cooldown > 0) this.cooldown -= dt;

        // Decrement buffs
        for (const key in this.buffs) {
            if (this.buffs[key] > 0) this.buffs[key] -= dt;
        }

        // Drone timer update
        if(this.buffs.drone > 0) {
            this.droneTimer -= dt;
        }
    }
    // shoot logic moved to GameWorld.update for better projectile handling
    draw(ctx) {
        ctx.save();
        ctx.translate(this.x, this.y);
        const color = (this.inputKey === 'p1') ? COLORS.green : COLORS.blue;
        
        // Shield Visual
        if(this.shield > 0) {
            ctx.beginPath();
            ctx.arc(0, 0, this.radius + 8, 0, Math.PI*2);
            ctx.strokeStyle = COLORS.gold;
            ctx.lineWidth = 2;
            ctx.shadowBlur = 15;
            ctx.shadowColor = COLORS.gold;
            ctx.stroke();
            ctx.shadowBlur = 0;
        }

        ctx.shadowBlur = 20; ctx.shadowColor = color; ctx.strokeStyle = COLORS.white; ctx.lineWidth = 2;
        ctx.beginPath(); ctx.moveTo(0, -20); ctx.lineTo(15, 15); ctx.lineTo(0, 5); ctx.lineTo(-15, 15); ctx.closePath(); ctx.stroke();
        ctx.shadowColor = COLORS.orange; ctx.strokeStyle = COLORS.orange; ctx.beginPath(); ctx.moveTo(-5, 10); ctx.lineTo(0, 25 + Math.random() * 10); ctx.lineTo(5, 10); ctx.stroke();

        // Drone Visual
        if(this.buffs.drone > 0) {
            ctx.restore(); ctx.save();
            ctx.translate(this.x - 30, this.y + Math.sin(Date.now()/100)*5);
            ctx.shadowBlur = 10; ctx.shadowColor = COLORS.cyan; ctx.strokeStyle = COLORS.cyan;
            ctx.beginPath(); ctx.moveTo(0, -10); ctx.lineTo(8, 8); ctx.lineTo(-8, 8); ctx.closePath(); ctx.stroke();
            ctx.restore(); ctx.save(); ctx.translate(this.x, this.y);
        }
        ctx.restore();
    }
}

/**
 * 4. WORLD CONTAINER
 */
class GameWorld {
    constructor(x, y, w, h, playerKey) {
        this.vx = x; this.vy = y; this.vw = w; this.vh = h;
        this.player = new Player(w/2, h-80, playerKey);
        this.bullets = [];      // Player bullets
        this.enemyBullets = []; // Boss bullets
        this.enemies = [];
        this.particles = [];
        this.powerups = [];
        this.boss = null;
        this.score = 0;
        this.time = 0;
        this.spawnTimer = 0;
        this.bossTimer = 90; // Boss appears after 90 seconds
        this.active = true;
    }

    update(dt) {
        if(!this.active) return;
        this.time += dt;

        // FIX: Asteroids spawn regardless of boss presence
        this.spawnTimer -= dt;
        if(this.spawnTimer <= 0) {
            this.spawnEnemy();
            this.spawnTimer = 0.8;
        }

        // Boss appearance logic
        if(!this.boss) {
            this.bossTimer -= dt;
            if(this.bossTimer <= 0) {
                this.boss = new Boss(this.vw);
            }
        } else {
            this.boss.update(dt);
            // Boss Shooting (Aggressive Multi-shot, inherited from previous fix)
            if (this.boss.state === 'fight' && this.boss.cooldown <= 0) {
                AudioSys.shoot();
                this.boss.cooldown = 1.2;
                // Center Shot
                this.enemyBullets.push(new Projectile(this.boss.x, this.boss.y + 60, 400, COLORS.red, 'normal', true));
                // Angled Shots
                let p1 = new Projectile(this.boss.x, this.boss.y + 60, 400, COLORS.red, 'normal', true); p1.vx = -150;
                let p2 = new Projectile(this.boss.x, this.boss.y + 60, 400, COLORS.red, 'normal', true); p2.vx = 150;
                this.enemyBullets.push(p1, p2);
            }
        }

        const player = this.player;
        const inp = keys[player.inputKey];
        
        // Calculate attack speed multiplier from buff
        const attackSpeedMultiplier = player.buffs.speed > 0 ? 0.6 : 1.0;
        const shotRequested = (inp.fire && player.cooldown <= 0);

        player.update(dt, this.vw, this.vh);
        
        // --- Player Shooting Logic ---
        if(shotRequested) {
            const baseCooldown = 0.2;
            player.cooldown = baseCooldown * attackSpeedMultiplier;
            AudioSys.shoot();

            if (player.buffs.laser > 0) {
                // Laser logic (vy=0 now, it does not move)
                let laser = new Projectile(player.x, player.y, 0, COLORS.purple); 
                laser.isLaser = true;
                laser.life = 0.3; // Short duration for laser
                laser.damage = 15; // High damage
                this.bullets.push(laser);
            } else if (player.buffs.multi > 0) {
                // Multishot
                this.bullets.push(new Projectile(player.x, player.y, -600, COLORS.green));
                let p1 = new Projectile(player.x, player.y, -500, COLORS.green); p1.vx = -150;
                let p2 = new Projectile(player.x, player.y, -500, COLORS.green); p2.vx = 150;
                this.bullets.push(p1, p2);
            } else {
                // Normal shot
                this.bullets.push(new Projectile(player.x, player.y, -600, COLORS.blue));
            }
        }
        
        // --- Drone Logic (Now respects Multishot) ---
        if(player.buffs.drone > 0 && player.droneTimer <= 0) {
            player.droneTimer = 0.3 * attackSpeedMultiplier; // Drone attack speed scales with player speed buff
            AudioSys.shoot();
            const dx = player.x - 30;
            const dy = player.y;
            
            if (player.buffs.multi > 0) {
                // Drone Multishot
                this.bullets.push(new Projectile(dx, dy, -900, COLORS.cyan));
                let p1 = new Projectile(dx, dy, -800, COLORS.cyan); p1.vx = -100;
                let p2 = new Projectile(dx, dy, -800, COLORS.cyan); p2.vx = 100;
                this.bullets.push(p1, p2);
            } else {
                this.bullets.push(new Projectile(dx, dy, -900, COLORS.cyan));
            }
        }

        this.bullets.forEach(b => b.update(dt, this.vw, this.vh));
        this.enemyBullets.forEach(b => b.update(dt, this.vw, this.vh));
        this.enemies.forEach(e => e.update(dt, this.vh));
        this.particles.forEach(p => p.update(dt));
        this.powerups.forEach(p => p.update(dt, this.vh));

        this.checkCollisions();

        this.bullets = this.bullets.filter(b => b.active);
        this.enemyBullets = this.enemyBullets.filter(b => b.active);
        this.enemies = this.enemies.filter(e => e.active);
        this.particles = this.particles.filter(p => p.active);
        this.powerups = this.powerups.filter(p => p.active);

        if(this.player.hp <= 0) this.active = false;
    }

    spawnEnemy() {
        const r = Math.random();
        let type = 'SMALL';
        
        // 18% chance for Diamond/Green
        if(r > 0.92) type = 'DIAMOND'; 
        else if(r > 0.82) type = 'GREEN'; 
        else if(r > 0.65) type = 'LARGE'; 
        else if(r > 0.40) type = 'MEDIUM'; 
        // else SMALL 

        const e = new Entity(Math.random()*(this.vw-40)+20, -50, type);
        this.enemies.push(e);
    }

    checkCollisions() {
        // Player Bullets vs Enemies/Boss
        for(let b of this.bullets) {
            
            // --- Проверка попадания по врагам (астероидам) ---
            for(let e of this.enemies) {
                
                let isHit = false;

                if (b.isLaser) {
                    // FIX: Специальная логика столкновения для лазерного луча (AABB/Box Check)
                    const laserWidth = 8;
                    const laserX1 = b.x - laserWidth / 2;
                    const laserX2 = b.x + laserWidth / 2;
                    
                    const enemyX1 = e.x - e.radius;
                    const enemyX2 = e.x + e.radius;
                    
                    // Проверка горизонтального перекрытия
                    const horizontalOverlap = laserX1 < enemyX2 && laserX2 > enemyX1;
                    
                    // Проверка вертикального перекрытия: враг должен быть выше игрока (b.y)
                    const verticalOverlap = e.y > 0 && e.y < b.y; 

                    if (horizontalOverlap && verticalOverlap) {
                        isHit = true;
                    }
                } else if(this.dist(b,e) < e.radius + b.radius) {
                    // Стандартный снаряд (точка-круг)
                    isHit = true;
                }

                if (isHit) {
                    if(!b.isLaser) b.active = false; // Лазер не деактивируется при попадании
                    e.hp -= b.damage;
                    this.createParticles(e.x, e.y, e.color);
                    
                    if(e.hp <= 0) {
                        e.active = false;
                        this.score += e.score;
                        AudioSys.boom();
                        if(e.drop) this.dropPowerup(e.x, e.y, e.drop);
                        if(Math.random() < 0.05) this.dropPowerup(e.x, e.y, 'heal');
                    } else {
                        AudioSys.hit();
                    }
                }
            }
            
            // --- Проверка попадания по боссу ---
            if(this.boss) {
                let bossHit = false;
                
                if (b.isLaser) {
                    // FIX: Специальная логика столкновения для лазера с боссом
                    const laserX1 = b.x - 4;
                    const laserX2 = b.x + 4;
                    
                    // Упрощенная зона босса (примерно)
                    const bossRadius = 80;
                    const bossX1 = this.boss.x - bossRadius;
                    const bossX2 = this.boss.x + bossRadius;
                    const bossY1 = this.boss.y - bossRadius;
                    const bossY2 = this.boss.y + bossRadius;

                    const horizontalOverlap = laserX1 < bossX2 && laserX2 > bossX1;
                    // Босс должен быть в зоне действия луча
                    const verticalOverlap = bossY2 > 0 && bossY1 < b.y;
                    
                    if (horizontalOverlap && verticalOverlap) {
                        bossHit = true;
                    }
                } else if(this.dist(b, this.boss) < 80) { 
                    // Стандартный снаряд
                    bossHit = true;
                }

                if (bossHit) {
                    if(!b.isLaser) b.active = false;
                    this.boss.hp -= b.damage * (b.isLaser ? 1 : 0.5); // Лазер наносит полный урон (15), обычные 0.5
                    AudioSys.hit();
                    if(this.boss.hp<=0) {
                        this.boss = null;
                        this.score += 5000;
                        this.createParticles(this.boss.x, this.boss.y, COLORS.red);
                    }
                }
            }
        }

        // Enemies vs Player
        for(let e of this.enemies) {
            if(this.dist(this.player, e) < this.player.radius + e.radius) {
                e.active = false;
                this.playerHit();
                this.createParticles(this.player.x, this.player.y, COLORS.red);
                AudioSys.boom();
            }
        }

        // Boss Bullets vs Player
        for(let b of this.enemyBullets) {
            if(this.dist(b, this.player) < this.player.radius + b.radius) {
                b.active = false;
                this.playerHit();
                this.createParticles(this.player.x, this.player.y, COLORS.red);
            }
        }

        // Powerups
        for(let p of this.powerups) {
            if(this.dist(this.player, p) < this.player.radius + p.radius) {
                p.active = false;
                this.applyPowerup(p.type);
                AudioSys.powerup();
            }
        }
    }

    playerHit() {
        if (this.player.shield > 0) {
            this.player.shield--;
            AudioSys.hit(); 
        } else {
            this.player.hp--;
        }
    }

    dist(a, b) { return Math.sqrt((a.x-b.x)**2 + (a.y-b.y)**2); }

    createParticles(x, y, color) {
        for(let i=0; i<10; i++) this.particles.push(new Particle(x, y, color));
    }

    dropPowerup(x, y, type) {
        let finalType = 'heal';
        if (type === 'heal') finalType = 'heal';
        else if (type === 'green') {
            const r = Math.random();
            if (r < 0.3) finalType = 'shield'; 
            else if (r < 0.6) finalType = 'multi'; 
            else if (r < 0.8) finalType = 'speed'; 
            else finalType = 'heal'; 
        } 
        else if (type === 'diamond') {
            const r = Math.random();
            if(r<0.25) finalType = 'multi';
            else if(r<0.50) finalType = 'drone';
            else if(r<0.70) finalType = 'shield';
            else if(r<0.85) finalType = 'laser'; 
            else finalType = 'nuke';
        }
        this.powerups.push(new PowerUp(x, y, finalType));
    }

    applyPowerup(t) {
        if(t==='heal') this.player.hp = Math.min(this.player.maxHp, this.player.hp+1);
        if(t==='multi') this.player.buffs.multi = 15;
        if(t==='drone') this.player.buffs.drone = 15;
        if(t==='shield') this.player.shield = 3; 
        if(t==='speed') this.player.buffs.speed = 15; 
        if(t==='laser') this.player.buffs.laser = 5; 

        if(t==='nuke') {
            AudioSys.boom();
            this.enemies.forEach(e => {
                e.active = false;
                this.score += e.score;
                this.createParticles(e.x, e.y, e.color);
            });
            if(this.boss) this.boss.hp -= 200;
        }
    }

    draw(ctx) {
        ctx.save();
        ctx.translate(this.vx, this.vy);
        ctx.beginPath(); ctx.rect(0, 0, this.vw, this.vh); ctx.clip();
        ctx.strokeStyle = '#333'; ctx.strokeRect(0, 0, this.vw, this.vh);

        this.player.draw(ctx);
        if(this.boss) this.boss.draw(ctx);
        
        this.enemies.forEach(e => e.draw(ctx));
        this.bullets.forEach(b => b.draw(ctx));
        this.enemyBullets.forEach(b => b.draw(ctx)); 
        this.powerups.forEach(p => p.draw(ctx));
        this.particles.forEach(p => p.draw(ctx));

        // HUD
        ctx.fillStyle = "#fff";
        ctx.font = "12px 'Press Start 2P'";
        ctx.textAlign = "left";
        ctx.fillText(`SCORE: ${this.score}`, 10, 20);

        const hpPct = this.player.hp / this.player.maxHp;
        ctx.strokeStyle = COLORS.green;
        ctx.strokeRect(10, 30, 100, 10);
        ctx.fillStyle = COLORS.green;
        ctx.fillRect(10, 30, 100 * hpPct, 10);

        let buffY = 55;
        
        // Buff indicators
        const buffs = this.player.buffs;
        const buffMap = [
            { key: 'multi', label: 'MULTI', color: COLORS.green },
            { key: 'drone', label: 'DRONE', color: COLORS.cyan },
            { key: 'speed', label: 'SPEED', color: COLORS.orange },
            { key: 'laser', label: 'LASER', color: COLORS.purple }
        ];

        buffMap.forEach(b => {
            if (buffs[b.key] > 0) {
                ctx.fillStyle = b.color;
                const w = (buffs[b.key] / 15) * 80;
                ctx.fillText(b.label, 10, buffY);
                ctx.fillRect(70, buffY - 8, w, 6);
                buffY += 15;
            }
        });
        
        // Shield Count
        if(this.player.shield > 0) {
            ctx.fillStyle = COLORS.gold;
            ctx.fillText(`SHIELD x${this.player.shield}`, 10, buffY);
            buffY += 15;
        }

        if(this.boss) {
             const bPct = Math.max(0, this.boss.hp / this.boss.maxHp);
             ctx.strokeStyle = COLORS.red;
             ctx.strokeRect(this.vw/2 - 60, 50, 120, 8);
             ctx.fillStyle = COLORS.red;
             ctx.fillRect(this.vw/2 - 60, 50, 120 * bPct, 8);
        }

        if(!this.active) {
            ctx.fillStyle = "rgba(0,0,0,0.7)";
            ctx.fillRect(0,0,this.vw,this.vh);
            ctx.fillStyle = COLORS.red;
            ctx.font = "20px 'Press Start 2P'";
            ctx.textAlign = "center";
            ctx.fillText("GAME OVER", this.vw/2, this.vh/2);
        }
        ctx.restore();
    }
}

/**
 * 5. MAIN GAME LOOP
 */
let canvas, ctx;
let worlds = [];
let gameLoopId;
let lastTime = 0;
let lastMode = 'single';

window.onload = function() {
    canvas = document.getElementById('gameCanvas');
    ctx = canvas.getContext('2d');
    resize();
    window.addEventListener('resize', resize);
};

function resize() {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
}

function initGame(mode) {
    lastMode = mode;
    AudioSys.init(); 
    
    document.getElementById('menu-main').classList.add('hidden');
    document.getElementById('menu-gameover').classList.add('hidden');
    
    worlds = [];
    if(mode === 'single') {
        worlds.push(new GameWorld(0, 0, canvas.width, canvas.height, 'p1'));
    } else {
        const w = canvas.width / 2;
        worlds.push(new GameWorld(0, 0, w, canvas.height, 'p1'));
        worlds.push(new GameWorld(w, 0, w, canvas.height, 'p2'));
    }

    lastTime = performance.now();
    if(gameLoopId) cancelAnimationFrame(gameLoopId);
    gameLoop(lastTime);
}

function gameLoop(timestamp) {
    const dt = Math.min((timestamp - lastTime) / 1000, 0.1);
    lastTime = timestamp;

    ctx.fillStyle = '#050505';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    let allDead = true;
    worlds.forEach(w => {
        w.update(dt);
        w.draw(ctx);
        if(w.active) allDead = false;
    });

    if(worlds.length === 2) {
        ctx.beginPath();
        ctx.moveTo(canvas.width/2, 0);
        ctx.lineTo(canvas.width/2, canvas.height);
        ctx.strokeStyle = '#fff';
        ctx.stroke();
    }

    if(allDead) {
        let txt = "SCORES:\n";
        worlds.forEach((w,i) => txt += `P${i+1}: ${w.score}\n`);
        document.getElementById('score-final').innerText = txt;
        document.getElementById('menu-gameover').classList.remove('hidden');
    } else {
        gameLoopId = requestAnimationFrame(gameLoop);
    }
}

function goToMenu() {
    if(gameLoopId) cancelAnimationFrame(gameLoopId);
    document.getElementById('menu-main').classList.remove('hidden');
    document.getElementById('menu-gameover').classList.add('hidden');
}

function toggleSound() {
    AudioSys.muted = !AudioSys.muted;
}

window.worlds = worlds; // Expose worlds for Projectile drawing reference
window.game = {
    triggerNuke: () => {
        worlds.forEach(w => {
            w.applyPowerup('nuke');
        });
    }
};
</script>
</body>
</html>

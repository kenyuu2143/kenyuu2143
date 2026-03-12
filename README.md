<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>AXI β : Bug Hunt Protocol</title>
  <style>
    :root{
      --bg:#040714;
      --bg2:#09102a;
      --panel:rgba(7,12,30,.82);
      --panel2:rgba(8,16,34,.92);
      --cyan:#00f5ff;
      --cyan2:#7df9ff;
      --blue:#4da6ff;
      --magenta:#ff00aa;
      --pink:#ff5ccf;
      --red:#ff4d6d;
      --orange:#ff9f43;
      --green:#00ff99;
      --yellow:#ffe066;
      --white:#dffcff;
      --grid:rgba(0,245,255,.08);
      --line:rgba(0,245,255,.18);
    }

    *{box-sizing:border-box}

    html,body{
      margin:0;
      min-height:100%;
      font-family:Consolas,"Courier New",monospace;
      color:var(--white);
      background:
        radial-gradient(circle at 50% 0%, rgba(255,0,170,.12), transparent 28%),
        radial-gradient(circle at 50% 100%, rgba(0,245,255,.10), transparent 34%),
        linear-gradient(180deg, #0b1030 0%, #050816 40%, #040714 100%);
      overflow:hidden;
    }

    body::before{
      content:"";
      position:fixed;
      inset:0;
      pointer-events:none;
      background:
        linear-gradient(rgba(255,255,255,.02) 50%, transparent 50%);
      background-size:100% 4px;
      opacity:.22;
      mix-blend-mode:screen;
    }

    .wrap{
      width:min(1280px, 96vw);
      margin:0 auto;
      padding:18px 16px 20px;
      position:relative;
      z-index:2;
    }

    .title{
      text-align:center;
      margin:0;
      color:var(--cyan);
      font-size:clamp(28px,4vw,52px);
      letter-spacing:3px;
      text-shadow:
        0 0 8px rgba(0,245,255,.6),
        0 0 22px rgba(0,245,255,.35);
    }

    .subtitle{
      text-align:center;
      margin:8px 0 16px;
      color:var(--cyan2);
      font-size:clamp(12px,1.4vw,18px);
      opacity:.95;
    }

    .layout{
      display:grid;
      grid-template-columns:300px 1fr;
      gap:16px;
      align-items:start;
    }

    .panel{
      background:var(--panel);
      border:1px solid var(--line);
      border-radius:20px;
      box-shadow:
        0 0 30px rgba(0,245,255,.09),
        inset 0 0 22px rgba(255,0,170,.05);
      backdrop-filter:blur(10px);
    }

    .sidebar{
      padding:16px;
    }

    .sectionTitle{
      color:var(--cyan);
      margin:0 0 12px;
      font-size:18px;
      text-shadow:0 0 8px rgba(0,245,255,.35);
    }

    .stat{
      margin-bottom:14px;
    }

    .label{
      display:flex;
      justify-content:space-between;
      font-size:13px;
      margin-bottom:6px;
      color:#d4f3ff;
    }

    .bar{
      height:13px;
      border-radius:999px;
      background:rgba(255,255,255,.06);
      overflow:hidden;
      border:1px solid rgba(255,255,255,.08);
    }

    .fill{
      height:100%;
      width:50%;
      border-radius:999px;
      box-shadow:0 0 14px rgba(0,245,255,.3);
      transition:width .08s linear;
    }

    .hpFill{ background:linear-gradient(90deg,var(--green),var(--cyan)); }
    .energyFill{ background:linear-gradient(90deg,var(--cyan),var(--magenta)); }
    .scoreFill{ background:linear-gradient(90deg,var(--magenta),var(--red)); }
    .waveFill{ background:linear-gradient(90deg,var(--blue),var(--cyan)); }
    .bossFill{ background:linear-gradient(90deg,var(--orange),var(--red)); }

    .terminal{
      margin-top:12px;
      background:rgba(0,0,0,.26);
      border:1px solid rgba(0,245,255,.15);
      border-radius:14px;
      padding:12px;
      min-height:220px;
      white-space:pre-line;
      line-height:1.7;
      font-size:13px;
      color:#daf8ff;
      overflow:auto;
    }

    .tips{
      margin-top:12px;
      font-size:12px;
      line-height:1.75;
      color:#c8e8ff;
      opacity:.95;
    }

    .gamePanel{
      padding:14px;
      position:relative;
    }

    canvas{
      width:100%;
      height:auto;
      aspect-ratio:16/10;
      display:block;
      border-radius:16px;
      border:1px solid rgba(0,245,255,.18);
      background:
        radial-gradient(circle at 50% 50%, rgba(0,245,255,.07), transparent 40%),
        radial-gradient(circle at 20% 20%, rgba(255,0,170,.05), transparent 26%),
        rgba(3,8,24,.96);
      box-shadow:
        inset 0 0 36px rgba(0,245,255,.08),
        0 0 30px rgba(0,245,255,.06);
    }

    .footer{
      margin-top:12px;
      text-align:center;
      font-size:12px;
      color:#ccecff;
      opacity:.9;
    }

    .bossBox{
      margin-top:14px;
      display:none;
    }

    .bossBox.active{ display:block; }

    .overlayUi{
      position:absolute;
      inset:14px;
      pointer-events:none;
      border-radius:16px;
      overflow:hidden;
    }

    .flash{
      position:absolute;
      inset:0;
      background:radial-gradient(circle, rgba(255,255,255,.14), transparent 50%);
      opacity:0;
      transition:opacity .12s linear;
      mix-blend-mode:screen;
    }

    @media (max-width: 960px){
      .layout{ grid-template-columns:1fr; }
    }
  </style>
</head>
<body>
  <div class="wrap">
    <h1 class="title">AXI β : BUG HUNT PROTOCOL</h1>
    <div class="subtitle">AXIβをちょろ出しした軽量サイバーパンク・ミニシューティング</div>

    <div class="layout">
      <aside class="panel sidebar">
        <h2 class="sectionTitle">SYSTEM STATUS</h2>

        <div class="stat">
          <div class="label"><span>CORE HP</span><span id="hpText">100</span></div>
          <div class="bar"><div id="hpBar" class="fill hpFill"></div></div>
        </div>

        <div class="stat">
          <div class="label"><span>ENERGY</span><span id="energyText">100</span></div>
          <div class="bar"><div id="energyBar" class="fill energyFill"></div></div>
        </div>

        <div class="stat">
          <div class="label"><span>SCORE</span><span id="scoreText">0</span></div>
          <div class="bar"><div id="scoreBar" class="fill scoreFill"></div></div>
        </div>

        <div class="stat">
          <div class="label"><span>WAVE</span><span id="waveText">1</span></div>
          <div class="bar"><div id="waveBar" class="fill waveFill" style="width:0%"></div></div>
        </div>

        <div id="bossBox" class="bossBox">
          <div class="label"><span>BOSS CORE</span><span id="bossHpText">0</span></div>
          <div class="bar"><div id="bossBar" class="fill bossFill" style="width:0%"></div></div>
        </div>

        <div class="terminal" id="log">
> booting AXI beta core...
> synchronizing neon shell...
> enemy signature: BUG swarm
> mission: survive and purge hostile code
        </div>

        <div class="tips">
          MOVE : Arrow Keys / WASD<br>
          SHOOT : Space<br>
          RESTART : R<br>
          <br>
          AXIβ はまだ未完成。<br>
          でもBUG掃除と緊急戦闘くらいならできる。
        </div>
      </aside>

      <section class="panel gamePanel">
        <canvas id="game" width="960" height="600"></canvas>
        <div class="overlayUi">
          <div id="flash" class="flash"></div>
        </div>
      </section>
    </div>

    <div class="footer">
      GitHub Pages mini game for <strong>kenyuu2143</strong> • AXI Cyber Lab prototype
    </div>
  </div>

  <script>
    const canvas = document.getElementById("game");
    const ctx = canvas.getContext("2d");

    const hpText = document.getElementById("hpText");
    const energyText = document.getElementById("energyText");
    const scoreText = document.getElementById("scoreText");
    const waveText = document.getElementById("waveText");
    const hpBar = document.getElementById("hpBar");
    const energyBar = document.getElementById("energyBar");
    const scoreBar = document.getElementById("scoreBar");
    const waveBar = document.getElementById("waveBar");
    const bossBox = document.getElementById("bossBox");
    const bossHpText = document.getElementById("bossHpText");
    const bossBar = document.getElementById("bossBar");
    const flashEl = document.getElementById("flash");
    const logEl = document.getElementById("log");

    const state = {
      mode: "title",
      keys: {},
      bullets: [],
      enemies: [],
      particles: [],
      items: [],
      score: 0,
      hp: 100,
      energy: 100,
      wave: 1,
      killsInWave: 0,
      targetKillsForWave: 8,
      shootCooldown: 0,
      enemySpawnTimer: 0,
      gameOver: false,
      time: 0,
      dialogueCooldown: 0,
      screenShake: 0,
      flash: 0,
      bossSpawned: false,
      boss: null,
      player: {
        x: canvas.width / 2,
        y: canvas.height / 2,
        r: 16,
        speed: 4.2,
        angle: 0
      }
    };

    function rand(min, max){ return Math.random() * (max - min) + min; }
    function clamp(v, min, max){ return Math.max(min, Math.min(max, v)); }

    function addLog(text){
      logEl.textContent += `\n> ${text}`;
      logEl.scrollTop = logEl.scrollHeight;
    }

    function axiSpeak(text){
      addLog(`AXIβ : ${text}`);
    }

    function triggerFlash(power=1){
      state.flash = Math.max(state.flash, power);
    }

    function maybeAxiLine(type){
      if(state.dialogueCooldown > 0) return;

      const lines = {
        start: [
          "system check complete.",
          "bug掃除、開始する。",
          "未完成だけど…やってみる。"
        ],
        hit: [
          "ぐっ…侵入を確認。",
          "core damage detected.",
          "少し痛い。"
        ],
        heal: [
          "resource acquired.",
          "補給完了。まだ動ける。",
          "energy stabilized."
        ],
        wave: [
          "敵性反応、増加中。",
          "次のwaveが来る。",
          "まだ終わらないみたい。"
        ],
        boss: [
          "大きいのが来る…注意。",
          "boss signature detected.",
          "これは少し本気で行く。"
        ],
        victory: [
          "purge complete.",
          "sector secured.",
          "片付いたね。"
        ]
      };

      const group = lines[type];
      if(!group || group.length === 0) return;
      axiSpeak(group[Math.floor(Math.random() * group.length)]);
      state.dialogueCooldown = 150;
    }

    function spawnEnemy(){
      const side = Math.floor(Math.random() * 4);
      let x, y;

      if(side === 0){ x = rand(0, canvas.width); y = -30; }
      else if(side === 1){ x = canvas.width + 30; y = rand(0, canvas.height); }
      else if(side === 2){ x = rand(0, canvas.width); y = canvas.height + 30; }
      else { x = -30; y = rand(0, canvas.height); }

      const size = rand(12, 22);
      const speed = rand(0.9, 1.8) + state.wave * 0.12;

      state.enemies.push({
        kind: "bug",
        x, y,
        r: size,
        speed,
        hp: Math.max(1, Math.floor(state.wave / 2)),
        pulse: rand(0, Math.PI * 2)
      });
    }

    function spawnBoss(){
      state.boss = {
        kind: "boss",
        x: canvas.width / 2,
        y: -80,
        r: 42,
        speed: 1.25,
        hp: 70 + state.wave * 10,
        maxHp: 70 + state.wave * 10,
        pulse: 0,
        phase: 0
      };
      state.bossSpawned = true;
      bossBox.classList.add("active");
      maybeAxiLine("boss");
      explosion(canvas.width / 2, 80, "#ff4d6d", 80, 6);
      state.screenShake = 12;
      triggerFlash(0.9);
    }

    function fireBullet(){
      if(state.mode !== "play") return;
      if(state.shootCooldown > 0 || state.energy < 4 || state.gameOver) return;

      const p = state.player;
      const dx = Math.cos(p.angle);
      const dy = Math.sin(p.angle);

      state.bullets.push({
        x: p.x + dx * 20,
        y: p.y + dy * 20,
        vx: dx * 9,
        vy: dy * 9,
        r: 4,
        life: 72,
        color: "#00f5ff"
      });

      state.energy = clamp(state.energy - 4, 0, 100);
      state.shootCooldown = 9;
      muzzleBurst(p.x + dx * 18, p.y + dy * 18, p.angle);
      triggerFlash(0.15);
    }

    function muzzleBurst(x, y, angle){
      for(let i = 0; i < 8; i++){
        const a = angle + rand(-0.35, 0.35);
        const sp = rand(2, 5);
        state.particles.push({
          x, y,
          vx: Math.cos(a) * sp,
          vy: Math.sin(a) * sp,
          life: rand(12, 20),
          size: rand(2, 4),
          color: Math.random() < 0.5 ? "#00f5ff" : "#7df9ff",
          glow: 1
        });
      }
    }

    function explosion(x, y, color, count=14, power=4){
      for(let i = 0; i < count; i++){
        const angle = rand(0, Math.PI * 2);
        const speed = rand(0.8, power);
        state.particles.push({
          x, y,
          vx: Math.cos(angle) * speed,
          vy: Math.sin(angle) * speed,
          life: rand(18, 42),
          size: rand(2, 6),
          color,
          glow: 1
        });
      }
    }

    function ringBurst(x, y, color, count=24, radius=12){
      for(let i = 0; i < count; i++){
        const ang = (Math.PI * 2 / count) * i;
        state.particles.push({
          x: x + Math.cos(ang) * radius,
          y: y + Math.sin(ang) * radius,
          vx: Math.cos(ang) * rand(1.5, 4.5),
          vy: Math.sin(ang) * rand(1.5, 4.5),
          life: rand(18, 34),
          size: rand(2, 5),
          color,
          glow: 1
        });
      }
    }

    function dropItem(x, y){
      const roll = Math.random();
      let type = "energy";
      if(roll < 0.20) type = "heal";
      else if(roll < 0.35) type = "burst";

      state.items.push({
        x, y,
        r: 10,
        type,
        pulse: 0,
        vy: rand(-0.2, 0.2)
      });
    }

    function applyItem(item){
      if(item.type === "heal"){
        state.hp = clamp(state.hp + 20, 0, 100);
        maybeAxiLine("heal");
        explosion(item.x, item.y, "#00ff99", 28, 5);
      }else if(item.type === "burst"){
        state.energy = clamp(state.energy + 18, 0, 100);
        state.score += 120;
        axiSpeak("overclock fragment acquired.");
        explosion(item.x, item.y, "#ffe066", 32, 5);
      }else{
        state.energy = clamp(state.energy + 14, 0, 100);
        maybeAxiLine("heal");
        explosion(item.x, item.y, "#00f5ff", 24, 4);
      }
      triggerFlash(0.3);
    }

    function resetGame(){
      state.mode = "title";
      state.bullets = [];
      state.enemies = [];
      state.particles = [];
      state.items = [];
      state.score = 0;
      state.hp = 100;
      state.energy = 100;
      state.wave = 1;
      state.killsInWave = 0;
      state.targetKillsForWave = 8;
      state.shootCooldown = 0;
      state.enemySpawnTimer = 0;
      state.gameOver = false;
      state.time = 0;
      state.dialogueCooldown = 0;
      state.screenShake = 0;
      state.flash = 0;
      state.bossSpawned = false;
      state.boss = null;
      state.player.x = canvas.width / 2;
      state.player.y = canvas.height / 2;
      state.player.angle = 0;
      bossBox.classList.remove("active");
      logEl.textContent =
`> booting AXI beta core...
> synchronizing neon shell...
> enemy signature: BUG swarm
> mission: survive and purge hostile code`;
      updateHUD();
    }

    function startGame(){
      state.mode = "play";
      state.gameOver = false;
      state.bullets = [];
      state.enemies = [];
      state.particles = [];
      state.items = [];
      state.score = 0;
      state.hp = 100;
      state.energy = 100;
      state.wave = 1;
      state.killsInWave = 0;
      state.targetKillsForWave = 8;
      state.shootCooldown = 0;
      state.enemySpawnTimer = 10;
      state.dialogueCooldown = 0;
      state.screenShake = 0;
      state.flash = 0;
      state.bossSpawned = false;
      state.boss = null;
      state.player.x = canvas.width / 2;
      state.player.y = canvas.height / 2;
      state.player.angle = 0;
      bossBox.classList.remove("active");
      logEl.textContent =
`> booting AXI beta core...
> synchronizing neon shell...
> enemy signature: BUG swarm
> mission: survive and purge hostile code`;
      maybeAxiLine("start");
      updateHUD();
    }

    function nextWave(){
      state.wave += 1;
      state.killsInWave = 0;
      state.targetKillsForWave = 8 + state.wave * 3;
      state.energy = clamp(state.energy + 20, 0, 100);
      addLog(`wave ${state.wave} detected // bug swarm intensifying`);
      maybeAxiLine("wave");
      explosion(state.player.x, state.player.y, "#00f5ff", 40, 5);
      ringBurst(state.player.x, state.player.y, "#ff00aa", 28, 18);

      if(state.wave % 3 === 0 && !state.boss){
        spawnBoss();
      }
    }

    function updatePlayer(){
      const p = state.player;

      let dx = 0;
      let dy = 0;

      if(state.keys["arrowup"] || state.keys["w"]) dy -= 1;
      if(state.keys["arrowdown"] || state.keys["s"]) dy += 1;
      if(state.keys["arrowleft"] || state.keys["a"]) dx -= 1;
      if(state.keys["arrowright"] || state.keys["d"]) dx += 1;

      if(dx !== 0 || dy !== 0){
        const len = Math.hypot(dx, dy) || 1;
        p.x += (dx / len) * p.speed;
        p.y += (dy / len) * p.speed;
        p.angle = Math.atan2(dy, dx);
        state.energy = clamp(state.energy - 0.02, 0, 100);

        if(Math.random() < 0.55){
          state.particles.push({
            x: p.x - Math.cos(p.angle) * 12 + rand(-3, 3),
            y: p.y - Math.sin(p.angle) * 12 + rand(-3, 3),
            vx: rand(-0.8, 0.8),
            vy: rand(-0.8, 0.8),
            life: rand(8, 16),
            size: rand(1, 3),
            color: Math.random() < 0.5 ? "#00f5ff" : "#ff00aa",
            glow: 1
          });
        }
      }

      p.x = clamp(p.x, p.r, canvas.width - p.r);
      p.y = clamp(p.y, p.r, canvas.height - p.r);
      state.energy = clamp(state.energy + 0.03, 0, 100);
    }

    function updateBullets(){
      for(let i = state.bullets.length - 1; i >= 0; i--){
        const b = state.bullets[i];
        b.x += b.vx;
        b.y += b.vy;
        b.life--;

        if(Math.random() < 0.8){
          state.particles.push({
            x: b.x,
            y: b.y,
            vx: rand(-0.3, 0.3),
            vy: rand(-0.3, 0.3),
            life: rand(6, 12),
            size: rand(1, 3),
            color: b.color,
            glow: 1
          });
        }

        if(
          b.life <= 0 ||
          b.x < -20 || b.x > canvas.width + 20 ||
          b.y < -20 || b.y > canvas.height + 20
        ){
          state.bullets.splice(i, 1);
        }
      }
    }

    function damagePlayer(amount){
      state.hp = clamp(state.hp - amount, 0, 100);
      state.screenShake = Math.max(state.screenShake, 10);
      triggerFlash(0.6);
      explosion(state.player.x, state.player.y, "#ff4d6d", 35, 5);
      maybeAxiLine("hit");

      if(state.hp <= 0){
        state.gameOver = true;
        addLog("AXI beta core offline // press R to reboot");
      }
    }

    function handleEnemyDeath(e, i){
      state.enemies.splice(i, 1);
      state.score += 100;
      state.killsInWave += 1;
      state.energy = clamp(state.energy + 6, 0, 100);

      explosion(e.x, e.y, "#ff00aa", 36, 5);
      ringBurst(e.x, e.y, "#00f5ff", 20, 8);
      state.screenShake = Math.max(state.screenShake, 4);
      triggerFlash(0.22);

      if(Math.random() < 0.42){
        addLog("BUG purged successfully");
      }

      if(Math.random() < 0.26){
        dropItem(e.x, e.y);
      }

      if(state.killsInWave >= state.targetKillsForWave && !state.boss){
        nextWave();
      }
    }

    function updateEnemies(){
      for(let i = state.enemies.length - 1; i >= 0; i--){
        const e = state.enemies[i];
        const p = state.player;
        const angle = Math.atan2(p.y - e.y, p.x - e.x);

        e.x += Math.cos(angle) * e.speed;
        e.y += Math.sin(angle) * e.speed;
        e.pulse += 0.08;

        if(Math.random() < 0.25){
          state.particles.push({
            x: e.x + rand(-5, 5),
            y: e.y + rand(-5, 5),
            vx: rand(-0.4, 0.4),
            vy: rand(-0.4, 0.4),
            life: rand(10, 18),
            size: rand(1, 3),
            color: Math.random() < 0.5 ? "#ff4d6d" : "#ff00aa",
            glow: 1
          });
        }

        const distToPlayer = Math.hypot(p.x - e.x, p.y - e.y);
        if(distToPlayer < p.r + e.r){
          state.enemies.splice(i, 1);
          damagePlayer(14);
          continue;
        }

        for(let j = state.bullets.length - 1; j >= 0; j--){
          const b = state.bullets[j];
          const dist = Math.hypot(b.x - e.x, b.y - e.y);

          if(dist < b.r + e.r){
            state.bullets.splice(j, 1);
            e.hp -= 1;
            explosion(b.x, b.y, "#00f5ff", 10, 3);
            triggerFlash(0.12);

            if(e.hp <= 0){
              handleEnemyDeath(e, i);
            }
            break;
          }
        }
      }
    }

    function updateBoss(){
      if(!state.boss) return;
      const b = state.boss;
      const p = state.player;

      b.pulse += 0.05;
      b.phase += 0.02;

      if(b.y < 110){
        b.y += 1.3;
      }else{
        const targetX = canvas.width / 2 + Math.sin(b.phase) * 240;
        b.x += (targetX - b.x) * 0.03;
      }

      if(Math.random() < 0.45){
        state.particles.push({
          x: b.x + rand(-b.r * 0.6, b.r * 0.6),
          y: b.y + rand(-b.r * 0.6, b.r * 0.6),
          vx: rand(-0.7, 0.7),
          vy: rand(-0.7, 0.7),
          life: rand(12, 22),
          size: rand(2, 4),
          color: Math.random() < 0.5 ? "#ff4d6d" : "#ff00aa",
          glow: 1
        });
      }

      if(Math.random() < 0.05){
        const angle = Math.atan2(p.y - b.y, p.x - b.x);
        state.enemies.push({
          kind: "bug",
          x: b.x + Math.cos(angle) * 20,
          y: b.y + Math.sin(angle) * 20,
          r: rand(12, 18),
          speed: rand(1.2, 2.2),
          hp: 1 + Math.floor(state.wave / 3),
          pulse: rand(0, Math.PI * 2)
        });
      }

      const distToPlayer = Math.hypot(p.x - b.x, p.y - b.y);
      if(distToPlayer < p.r + b.r){
        damagePlayer(26);
      }

      for(let j = state.bullets.length - 1; j >= 0; j--){
        const bullet = state.bullets[j];
        const dist = Math.hypot(bullet.x - b.x, bullet.y - b.y);

        if(dist < bullet.r + b.r){
          state.bullets.splice(j, 1);
          b.hp -= 1;
          explosion(bullet.x, bullet.y, "#ffe066", 14, 3);
          ringBurst(bullet.x, bullet.y, "#ff00aa", 12, 4);
          state.screenShake = Math.max(state.screenShake, 2);
          triggerFlash(0.14);

          if(b.hp <= 0){
            state.score += 1200;
            explosion(b.x, b.y, "#ff4d6d", 80, 7);
            ringBurst(b.x, b.y, "#00f5ff", 42, 20);
            ringBurst(b.x, b.y, "#ffe066", 34, 10);
            state.screenShake = 16;
            triggerFlash(1);
            state.boss = null;
            state.bossSpawned = false;
            bossBox.classList.remove("active");
            maybeAxiLine("victory");
            state.wave += 1;
            state.killsInWave = 0;
            state.targetKillsForWave = 8 + state.wave * 3;
            state.energy = clamp(state.energy + 30, 0, 100);
          }
          break;
        }
      }
    }

    function updateItems(){
      for(let i = state.items.length - 1; i >= 0; i--){
        const item = state.items[i];
        item.pulse += 0.08;
        item.y += Math.sin(item.pulse) * 0.25 + item.vy;

        const dist = Math.hypot(state.player.x - item.x, state.player.y - item.y);
        if(dist < state.player.r + item.r + 2){
          applyItem(item);
          state.items.splice(i, 1);
        }
      }
    }

    function updateParticles(){
      for(let i = state.particles.length - 1; i >= 0; i--){
        const p = state.particles[i];
        p.x += p.vx;
        p.y += p.vy;
        p.vx *= 0.99;
        p.vy *= 0.99;
        p.life--;

        if(p.life <= 0){
          state.particles.splice(i, 1);
        }
      }
    }

    function updateHUD(){
      hpText.textContent = Math.round(state.hp);
      energyText.textContent = Math.round(state.energy);
      scoreText.textContent = state.score;
      waveText.textContent = state.wave;

      hpBar.style.width = `${state.hp}%`;
      energyBar.style.width = `${state.energy}%`;
      scoreBar.style.width = `${Math.min((state.score % 2000) / 20, 100)}%`;
      waveBar.style.width = `${Math.min((state.killsInWave / state.targetKillsForWave) * 100, 100)}%`;

      if(state.boss){
        bossHpText.textContent = `${state.boss.hp}`;
        bossBar.style.width = `${(state.boss.hp / state.boss.maxHp) * 100}%`;
      }else{
        bossHpText.textContent = "0";
        bossBar.style.width = "0%";
      }

      if(state.flash > 0){
        flashEl.style.opacity = String(Math.min(state.flash, 1) * 0.7);
      }else{
        flashEl.style.opacity = "0";
      }
    }

    function update(){
      state.time += 0.016;
      if(state.dialogueCooldown > 0) state.dialogueCooldown--;
      if(state.shootCooldown > 0) state.shootCooldown--;
      state.flash *= 0.86;
      state.screenShake *= 0.88;

      if(state.mode !== "play" || state.gameOver){
        updateHUD();
        return;
      }

      updatePlayer();
      updateBullets();
      updateEnemies();
      updateBoss();
      updateItems();
      updateParticles();

      state.enemySpawnTimer--;
      if(state.enemySpawnTimer <= 0 && !state.boss){
        spawnEnemy();
        state.enemySpawnTimer = Math.max(18, 55 - state.wave * 4);
      }

      updateHUD();
    }

    function drawGrid(){
      ctx.save();
      ctx.strokeStyle = "rgba(0,245,255,0.08)";
      ctx.lineWidth = 1;
      const offset = (state.time * 30) % 32;

      for(let x = -32; x < canvas.width + 32; x += 32){
        ctx.beginPath();
        ctx.moveTo(x + offset, 0);
        ctx.lineTo(x + offset, canvas.height);
        ctx.stroke();
      }

      for(let y = -32; y < canvas.height + 32; y += 32){
        ctx.beginPath();
        ctx.moveTo(0, y + offset);
        ctx.lineTo(canvas.width, y + offset);
        ctx.stroke();
      }

      ctx.strokeStyle = "rgba(255,0,170,0.05)";
      for(let x = 0; x < canvas.width; x += 96){
        ctx.beginPath();
        ctx.moveTo(x, 0);
        ctx.lineTo(x, canvas.height);
        ctx.stroke();
      }
      ctx.restore();
    }

    function drawBackgroundBloom(){
      const cx = canvas.width / 2;
      const cy = canvas.height / 2;
      const g = ctx.createRadialGradient(cx, cy, 30, cx, cy, 300);
      g.addColorStop(0, "rgba(0,245,255,0.08)");
      g.addColorStop(1, "rgba(0,245,255,0)");
      ctx.save();
      ctx.fillStyle = g;
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.restore();
    }

    function drawPlayer(){
      const p = state.player;
      ctx.save();
      ctx.translate(p.x, p.y);
      ctx.rotate(p.angle);

      const pulse = 1 + Math.sin(state.time * 6) * 0.05;

      ctx.shadowBlur = 24;
      ctx.shadowColor = "#00f5ff";

      ctx.strokeStyle = "#00f5ff";
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.arc(0, 0, p.r * pulse, 0, Math.PI * 2);
      ctx.stroke();

      ctx.fillStyle = "#091427";
      ctx.beginPath();
      ctx.arc(0, 0, p.r * 0.78, 0, Math.PI * 2);
      ctx.fill();

      ctx.fillStyle = "#7df9ff";
      ctx.beginPath();
      ctx.ellipse(0, 0, 8, 4.5, 0, 0, Math.PI * 2);
      ctx.fill();

      ctx.fillStyle = "#ff00aa";
      ctx.beginPath();
      ctx.moveTo(p.r + 8, 0);
      ctx.lineTo(p.r - 5, -7);
      ctx.lineTo(p.r - 5, 7);
      ctx.closePath();
      ctx.fill();

      ctx.strokeStyle = "rgba(255,0,170,.95)";
      ctx.lineWidth = 1.6;
      ctx.beginPath();
      ctx.arc(0, 0, p.r + 5 + Math.sin(state.time * 5) * 1.5, -0.7, 1.7);
      ctx.stroke();

      ctx.restore();
    }

    function drawBullets(){
      for(const b of state.bullets){
        ctx.save();
        ctx.shadowBlur = 14;
        ctx.shadowColor = b.color;
        ctx.fillStyle = b.color;
        ctx.beginPath();
        ctx.arc(b.x, b.y, b.r, 0, Math.PI * 2);
        ctx.fill();
        ctx.restore();
      }
    }

    function drawEnemy(e){
      const pulse = e.r + Math.sin(e.pulse) * 2;
      ctx.save();
      ctx.translate(e.x, e.y);

      ctx.shadowBlur = 20;
      ctx.shadowColor = "#ff4d6d";

      ctx.fillStyle = "#ff4d6d";
      ctx.beginPath();
      ctx.arc(0, 0, pulse, 0, Math.PI * 2);
      ctx.fill();

      ctx.strokeStyle = "#ff00aa";
      ctx.lineWidth = 2;
      ctx.beginPath();
      for(let i = 0; i < 8; i++){
        const ang = (Math.PI * 2 / 8) * i;
        const rr = pulse + (i % 2 === 0 ? 7 : 3);
        const x = Math.cos(ang) * rr;
        const y = Math.sin(ang) * rr;
        if(i === 0) ctx.moveTo(x, y);
        else ctx.lineTo(x, y);
      }
      ctx.closePath();
      ctx.stroke();

      ctx.restore();
    }

    function drawBoss(){
      if(!state.boss) return;
      const b = state.boss;

      ctx.save();
      ctx.translate(b.x, b.y);

      ctx.shadowBlur = 30;
      ctx.shadowColor = "#ff4d6d";

      ctx.fillStyle = "#35081a";
      ctx.beginPath();
      ctx.arc(0, 0, b.r, 0, Math.PI * 2);
      ctx.fill();

      ctx.strokeStyle = "#ff4d6d";
      ctx.lineWidth = 4;
      ctx.beginPath();
      ctx.arc(0, 0, b.r + Math.sin(b.pulse * 2) * 2, 0, Math.PI * 2);
      ctx.stroke();

      ctx.strokeStyle = "#ff00aa";
      ctx.lineWidth = 2;
      for(let k = 0; k < 2; k++){
        ctx.beginPath();
        for(let i = 0; i < 12; i++){
          const ang = (Math.PI * 2 / 12) * i + (k * 0.15);
          const rr = b.r + (i % 2 === 0 ? 14 : 6);
          const x = Math.cos(ang) * rr;
          const y = Math.sin(ang) * rr;
          if(i === 0) ctx.moveTo(x, y);
          else ctx.lineTo(x, y);
        }
        ctx.closePath();
        ctx.stroke();
      }

      ctx.fillStyle = "#ffe066";
      ctx.beginPath();
      ctx.arc(0, 0, 8 + Math.sin(b.pulse * 4) * 2, 0, Math.PI * 2);
      ctx.fill();

      ctx.restore();
    }

    function drawItems(){
      for(const item of state.items){
        ctx.save();
        ctx.translate(item.x, item.y);
        ctx.shadowBlur = 18;

        if(item.type === "heal"){
          ctx.shadowColor = "#00ff99";
          ctx.strokeStyle = "#00ff99";
          ctx.lineWidth = 3;
          ctx.beginPath();
          ctx.moveTo(-6, 0);
          ctx.lineTo(6, 0);
          ctx.moveTo(0, -6);
          ctx.lineTo(0, 6);
          ctx.stroke();
        }else if(item.type === "burst"){
          ctx.shadowColor = "#ffe066";
          ctx.fillStyle = "#ffe066";
          ctx.beginPath();
          for(let i = 0; i < 5; i++){
            const a = -Math.PI / 2 + i * (Math.PI * 2 / 5);
            const x = Math.cos(a) * 10;
            const y = Math.sin(a) * 10;
            if(i === 0) ctx.moveTo(x, y);
            else ctx.lineTo(x, y);
          }
          ctx.closePath();
          ctx.fill();
        }else{
          ctx.shadowColor = "#00f5ff";
          ctx.strokeStyle = "#00f5ff";
          ctx.lineWidth = 2;
          ctx.beginPath();
          ctx.arc(0, 0, 8 + Math.sin(item.pulse) * 1.5, 0, Math.PI * 2);
          ctx.stroke();
        }

        ctx.restore();
      }
    }

    function drawParticles(){
      for(const p of state.particles){
        ctx.save();
        ctx.globalAlpha = Math.max(0, p.life / 40);
        ctx.shadowBlur = p.glow ? 12 : 0;
        ctx.shadowColor = p.color;
        ctx.fillStyle = p.color;
        ctx.fillRect(p.x, p.y, p.size || 3, p.size || 3);
        ctx.restore();
      }
    }

    function drawMiniHUD(){
      ctx.save();
      ctx.fillStyle = "rgba(0,0,0,.28)";
      ctx.fillRect(14, 14, 220, 84);

      ctx.strokeStyle = "rgba(0,245,255,.25)";
      ctx.strokeRect(14, 14, 220, 84);

      ctx.fillStyle = "#dffcff";
      ctx.font = "16px Consolas";
      ctx.fillText(`HP     : ${Math.round(state.hp)}`, 28, 40);
      ctx.fillText(`SCORE  : ${state.score}`, 28, 62);
      ctx.fillText(`WAVE   : ${state.wave}`, 28, 84);

      if(state.boss){
        ctx.fillStyle = "#ff8aa8";
        ctx.fillText(`BOSS   : ACTIVE`, 28, 106);
      }
      ctx.restore();
    }

    function drawTitleScreen(){
      ctx.save();
      ctx.fillStyle = "rgba(0,0,0,.35)";
      ctx.fillRect(0,0,canvas.width,canvas.height);

      const cx = canvas.width / 2;
      const cy = canvas.height / 2;

      ctx.textAlign = "center";
      ctx.shadowBlur = 24;
      ctx.shadowColor = "#00f5ff";

      ctx.fillStyle = "#00f5ff";
      ctx.font = "bold 52px Consolas";
      ctx.fillText("AXI β", cx, cy - 90);

      ctx.fillStyle = "#dffcff";
      ctx.font = "24px Consolas";
      ctx.fillText("BUG HUNT PROTOCOL", cx, cy - 46);

      ctx.font = "18px Consolas";
      ctx.fillStyle = "#7df9ff";
      ctx.fillText("未完成のAXIβがBUG掃除に出る軽量ミニシューティング", cx, cy - 6);

      ctx.fillStyle = "rgba(0,0,0,.42)";
      ctx.fillRect(cx - 220, cy + 36, 440, 112);
      ctx.strokeStyle = "rgba(0,245,255,.28)";
      ctx.strokeRect(cx - 220, cy + 36, 440, 112);

      ctx.fillStyle = "#dffcff";
      ctx.font = "18px Consolas";
      ctx.fillText("MOVE : Arrow Keys / WASD", cx, cy + 70);
      ctx.fillText("SHOOT : Space", cx, cy + 98);
      ctx.fillText("PRESS ENTER TO START", cx, cy + 132);

      ctx.restore();
    }

    function drawGameOver(){
      if(!state.gameOver) return;
      ctx.save();
      ctx.fillStyle = "rgba(0,0,0,.56)";
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      ctx.textAlign = "center";
      ctx.fillStyle = "#ff4d6d";
      ctx.font = "bold 54px Consolas";
      ctx.fillText("SYSTEM DOWN", canvas.width / 2, canvas.height / 2 - 18);

      ctx.fillStyle = "#dffcff";
      ctx.font = "24px Consolas";
      ctx.fillText(`FINAL SCORE : ${state.score}`, canvas.width / 2, canvas.height / 2 + 24);
      ctx.fillText("Press R to reboot AXI β", canvas.width / 2, canvas.height / 2 + 64);
      ctx.restore();
    }

    function render(){
      ctx.save();
      const shakeX = state.screenShake > 0 ? rand(-state.screenShake, state.screenShake) : 0;
      const shakeY = state.screenShake > 0 ? rand(-state.screenShake, state.screenShake) : 0;
      ctx.translate(shakeX, shakeY);

      ctx.clearRect(-40, -40, canvas.width + 80, canvas.height + 80);
      drawBackgroundBloom();
      drawGrid();
      drawParticles();
      drawItems();
      for(const e of state.enemies) drawEnemy(e);
      drawBoss();
      drawBullets();
      drawPlayer();
      drawMiniHUD();

      if(state.mode === "title") drawTitleScreen();
      drawGameOver();

      ctx.restore();
    }

    function loop(){
      update();
      render();
      requestAnimationFrame(loop);
    }

    window.addEventListener("keydown", e => {
      const key = e.key.toLowerCase();
      state.keys[key] = true;

      if(["arrowup","arrowdown","arrowleft","arrowright"," "].includes(key) || e.key === " "){
        e.preventDefault();
      }

      if(state.mode === "title" && e.key === "Enter"){
        startGame();
        return;
      }

      if(key === " "){
        fireBullet();
      }

      if(state.gameOver && key === "r"){
        startGame();
      }
    });

    window.addEventListener("keyup", e => {
      state.keys[e.key.toLowerCase()] = false;
    });

    updateHUD();
    loop();
  </script>
</body>
</html>

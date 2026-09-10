<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Police Chase Runner</title>
<style>
  * { margin:0; padding:0; box-sizing:border-box; }
  body {
    background:#0b0b12;
    font-family:'Segoe UI', Arial, sans-serif;
    display:flex; align-items:center; justify-content:center;
    height:100vh; overflow:hidden;
  }
  #gameWrap {
    position:relative;
    width:480px; height:800px;
    max-width:100vw; max-height:100vh;
    background:#111;
    box-shadow:0 0 40px rgba(255,0,60,0.25);
    border-radius:8px;
    overflow:hidden;
  }
  canvas { display:block; background:#1a1a24; }

  .overlay {
    position:absolute; inset:0;
    display:flex; flex-direction:column; align-items:center; justify-content:center;
    text-align:center; color:#fff;
    background:linear-gradient(180deg, rgba(10,10,20,0.92), rgba(20,5,15,0.96));
    padding:24px;
  }
  .hidden { display:none !important; }
  h1 { font-size:28px; color:#ff2b4c; text-shadow:0 0 12px rgba(255,43,76,0.7); margin-bottom:6px; letter-spacing:1px;}
  .subtitle { color:#ffd23f; font-size:13px; margin-bottom:18px; letter-spacing:2px; text-transform:uppercase;}
  .story { max-width:380px; font-size:14px; line-height:1.6; color:#ccc; margin-bottom:22px; }
  .story b { color:#ff5a72; }
  .story .police { color:#4fc3ff; }

  button {
    background:linear-gradient(180deg,#ff2b4c,#c4001f);
    color:#fff; border:none; padding:14px 38px;
    font-size:16px; font-weight:700; border-radius:30px;
    cursor:pointer; letter-spacing:1px;
    box-shadow:0 6px 18px rgba(255,43,76,0.4);
    transition:transform .1s ease;
  }
  button:active { transform:scale(0.95); }

  #hud {
    position:absolute; top:0; left:0; right:0;
    display:flex; justify-content:space-between; padding:12px 16px;
    color:#fff; font-weight:700; font-size:15px;
    text-shadow:0 1px 4px rgba(0,0,0,0.8);
    z-index:5; pointer-events:none;
  }
  #hud .coins { color:#ffd23f; }
  #hud .dist { color:#4fc3ff; }
  #heat {
    position:absolute; top:38px; left:16px; right:16px;
    height:6px; background:rgba(255,255,255,0.15); border-radius:4px; overflow:hidden;
    z-index:5;
  }
  #heatBar { height:100%; width:0%; background:linear-gradient(90deg,#ffd23f,#ff2b4c); transition:width .2s; }

  #controls {
    position:absolute; bottom:14px; left:0; right:0;
    display:flex; justify-content:center; gap:10px; z-index:5;
  }
  .ctrlBtn {
    width:58px; height:58px; border-radius:50%;
    background:rgba(255,255,255,0.12); border:2px solid rgba(255,255,255,0.3);
    color:#fff; font-size:20px; display:flex; align-items:center; justify-content:center;
    user-select:none; -webkit-user-select:none; touch-action:manipulation;
  }
  .ctrlBtn:active { background:rgba(255,43,76,0.5); }

  #gameOverBox b.caught { color:#4fc3ff; display:block; margin-top:10px; font-size:15px;}
  .statLine { color:#ffd23f; font-size:15px; margin:4px 0; }
</style>
</head>
<body>

<div id="gameWrap">
  <canvas id="game" width="480" height="800"></canvas>

  <div id="hud" class="hidden">
    <span class="dist">DIST: <span id="distVal">0</span>m</span>
    <span class="coins">COINS: <span id="coinVal">0</span></span>
  </div>
  <div id="heat" class="hidden"><div id="heatBar"></div></div>

  <div id="controls" class="hidden">
    <div class="ctrlBtn" id="btnLeft">◀</div>
    <div class="ctrlBtn" id="btnSlide">▼</div>
    <div class="ctrlBtn" id="btnJump">▲</div>
    <div class="ctrlBtn" id="btnRight">▶</div>
  </div>

  <div id="startScreen" class="overlay">
    <h1>POLICE CHASE</h1>
    <div class="subtitle">Endless Runner</div>
    <div class="story">
      Your crew's cars are lined up, <b>bodyguards</b> loaded with weapons,
      the deal's about to go down on the old highway strip...
      <br><br>
      Then sirens scream. <span class="police">The police have arrived.</span>
      Drop everything and <b>RUN.</b>
      <br><br>
      Weave through 3 lanes of oncoming traffic. Swipe/press
      <b>LEFT / RIGHT</b> to change lanes, <b>UP</b> to jump barriers,
      <b>DOWN</b> to slide under obstacles. One collision and it's over.
      Survive — you earn <b>+10 coins every 20 seconds.</b>
    </div>
    <button id="startBtn">START RUN</button>
  </div>

  <div id="gameOverScreen" class="overlay hidden">
    <h1 style="color:#4fc3ff;">BUSTED!</h1>
    <div id="gameOverBox">
      <div class="story">
        <span class="police">"FREEZE! HANDS WHERE WE CAN SEE THEM!"</span><br>
        The squad car clips you against the guardrail — game over.
        <b class="caught">The police have you surrounded...</b>
      </div>
      <div class="statLine">Distance: <span id="finalDist">0</span>m</div>
      <div class="statLine">Coins collected: <span id="finalCoins">0</span></div>
    </div>
    <button id="restartBtn">RUN AGAIN</button>
  </div>
</div>

<script>
(function(){
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const W = canvas.width, H = canvas.height;

  const startScreen = document.getElementById('startScreen');
  const gameOverScreen = document.getElementById('gameOverScreen');
  const hud = document.getElementById('hud');
  const heatWrap = document.getElementById('heat');
  const heatBar = document.getElementById('heatBar');
  const controls = document.getElementById('controls');
  const distVal = document.getElementById('distVal');
  const coinVal = document.getElementById('coinVal');
  const finalDist = document.getElementById('finalDist');
  const finalCoins = document.getElementById('finalCoins');

  const LANES = [W*0.22, W*0.5, W*0.78]; // left, center, right lane x positions at player depth
  const GROUND_Y = H - 110;

  let state = 'menu'; // menu | running | gameover
  let laneIndex = 1;
  let playerX = LANES[1];
  let playerY = GROUND_Y;
  let jumping = false, jumpVel = 0;
  let sliding = false, slideTimer = 0;
  const SLIDE_DURATION = 0.6; // seconds — quick slide under obstacles
  const GRAVITY = 1600;
  const JUMP_FORCE = -680;

  let speed = 420;           // world scroll speed (px/sec at road scale)
  let baseSpeed = 420;
  let distance = 0;
  let coins = 0;
  let coinTimer = 0;
  let elapsed = 0;
  let heat = 0; // police proximity 0..1, purely cosmetic tension bar tied to speed ramp

  let obstacles = []; // {lane, z (0..1 depth, 1=far,0=at player), type:'car'|'barrier'|'beam', passed}
  let spawnTimer = 0;
  let particles = [];
  let coinPickups = [];

  function resetGame(){
    state = 'running';
    laneIndex = 1;
    playerX = LANES[1];
    jumping = false; jumpVel = 0;
    sliding = false; slideTimer = 0;
    speed = baseSpeed;
    distance = 0; coins = 0; coinTimer = 0; elapsed = 0; heat = 0;
    obstacles = []; spawnTimer = 0; particles = []; coinPickups = [];
    hud.classList.remove('hidden');
    heatWrap.classList.remove('hidden');
    controls.classList.remove('hidden');
    startScreen.classList.add('hidden');
    gameOverScreen.classList.add('hidden');
  }

  function gameOver(){
    state = 'gameover';
    finalDist.textContent = Math.floor(distance);
    finalCoins.textContent = coins;
    hud.classList.add('hidden');
    heatWrap.classList.add('hidden');
    controls.classList.add('hidden');
    gameOverScreen.classList.remove('hidden');
  }

  // ---------- Input ----------
  function goLeft(){ if(state!=='running') return; if(laneIndex>0) laneIndex--; }
  function goRight(){ if(state!=='running') return; if(laneIndex<2) laneIndex++; }
  function doJump(){ if(state!=='running') return; if(!jumping && !sliding){ jumping = true; jumpVel = JUMP_FORCE; } }
  function doSlide(){ if(state!=='running') return; if(!jumping && !sliding){ sliding = true; slideTimer = SLIDE_DURATION; } }

  window.addEventListener('keydown', (e)=>{
    if(e.key==='ArrowLeft'||e.key==='a'||e.key==='A') goLeft();
    else if(e.key==='ArrowRight'||e.key==='d'||e.key==='D') goRight();
    else if(e.key==='ArrowUp'||e.key===' '||e.key==='w'||e.key==='W') doJump();
    else if(e.key==='ArrowDown'||e.key==='s'||e.key==='S') doSlide();
  });

  document.getElementById('btnLeft').addEventListener('click', goLeft);
  document.getElementById('btnRight').addEventListener('click', goRight);
  document.getElementById('btnJump').addEventListener('click', doJump);
  document.getElementById('btnSlide').addEventListener('click', doSlide);

  // touch swipe support
  let touchStartX=0, touchStartY=0;
  canvas.addEventListener('touchstart', (e)=>{
    touchStartX = e.touches[0].clientX; touchStartY = e.touches[0].clientY;
  });
  canvas.addEventListener('touchend', (e)=>{
    const dx = e.changedTouches[0].clientX - touchStartX;
    const dy = e.changedTouches[0].clientY - touchStartY;
    if(Math.abs(dx) > Math.abs(dy)){
      if(Math.abs(dx) > 30) dx>0 ? goRight() : goLeft();
    } else {
      if(Math.abs(dy) > 30) dy>0 ? doSlide() : doJump();
    }
  });

  document.getElementById('startBtn').addEventListener('click', resetGame);
  document.getElementById('restartBtn').addEventListener('click', resetGame);

  // ---------- Spawning ----------
  function spawnObstacle(){
    const lane = Math.floor(Math.random()*3);
    const r = Math.random();
    let type = 'car';
    if(r < 0.33) type = 'car';
    else if(r < 0.66) type = 'barrier'; // needs jump
    else type = 'beam';                 // needs slide
    obstacles.push({ lane, z: 1.0, type, passed:false });

    // occasionally add a coin above/along a safe lane
    if(Math.random() < 0.5){
      const coinLane = Math.floor(Math.random()*3);
      coinPickups.push({ lane: coinLane, z: 1.0, taken:false });
    }
  }

  // ---------- Update ----------
  function update(dt){
    elapsed += dt;
    distance += speed*dt*0.05;
    speed = baseSpeed + Math.min(elapsed*4, 260); // gradual ramp up = police closing in
    heat = Math.min(1, (speed-baseSpeed)/260);
    heatBar.style.width = (heat*100)+'%';

    // coin timer: +10 every 20 seconds survived
    coinTimer += dt;
    if(coinTimer >= 20){
      coinTimer -= 20;
      coins += 10;
      particles.push({x:W-60,y:40,life:1,text:'+10'});
    }

    // player lane easing
    const targetX = LANES[laneIndex];
    playerX += (targetX-playerX)*Math.min(1, dt*12);

    // jump physics
    if(jumping){
      jumpVel += GRAVITY*dt;
      playerY += jumpVel*dt;
      if(playerY >= GROUND_Y){ playerY = GROUND_Y; jumping=false; jumpVel=0; }
    }
    // slide timer
    if(sliding){
      slideTimer -= dt;
      if(slideTimer <= 0){ sliding = false; }
    }

    // spawn control
    spawnTimer -= dt;
    const spawnInterval = Math.max(0.55, 1.3 - elapsed*0.01);
    if(spawnTimer <= 0){
      spawnObstacle();
      spawnTimer = spawnInterval;
    }

    // move obstacles toward player (z: 1 far -> 0 at player)
    const zSpeed = (speed/420) * 0.62;
    for(const ob of obstacles){
      ob.z -= zSpeed*dt;
    }
    for(const c of coinPickups){
      c.z -= zSpeed*dt;
    }

    // collision check near z ~ 0.06-0.16 (player depth band)
    for(const ob of obstacles){
      if(ob.passed) continue;
      if(ob.z <= 0.14 && ob.z >= 0.0){
        ob.passed = true;
        if(ob.lane === laneIndex){
          let avoided = false;
          if(ob.type === 'barrier' && jumping) avoided = true;
          if(ob.type === 'beam' && sliding) avoided = true;
          if(ob.type === 'car') avoided = false; // must change lane
          if(!avoided){ gameOver(); return; }
        }
      }
    }

    // coin pickup check
    for(const c of coinPickups){
      if(c.taken) continue;
      if(c.z <= 0.14 && c.z >= 0.0){
        c.taken = true;
        if(c.lane === laneIndex){ coins += 1; particles.push({x:playerX,y:playerY-60,life:1,text:'+1'}); }
      }
    }

    obstacles = obstacles.filter(o=> o.z > -0.05);
    coinPickups = coinPickups.filter(c=> c.z > -0.05);
    particles.forEach(p=>{ p.life -= dt; p.y -= 30*dt; });
    particles = particles.filter(p=>p.life>0);

    distVal.textContent = Math.floor(distance);
    coinVal.textContent = coins;
  }

  // ---------- Perspective helpers ----------
  function laneXatZ(lane, z){
    const horizonX = W/2;
    const nearX = LANES[lane];
    return horizonX + (nearX-horizonX)*(1-z);
  }
  function scaleAtZ(z){
    return 0.15 + (1-z)*0.85; // small far away, big near
  }
  function yAtZ(z){
    const horizonY = H*0.32;
    return horizonY + (GROUND_Y-horizonY)*(1-z);
  }

  // ---------- Drawing ----------
  function drawRoad(){
    const g = ctx.createLinearGradient(0,0,0,H*0.4);
    g.addColorStop(0,'#1b1030'); g.addColorStop(1,'#2a1730');
    ctx.fillStyle = g; ctx.fillRect(0,0,W,H*0.4);

    ctx.fillStyle = 'rgba(255,43,76,0.08)';
    ctx.fillRect(0,H*0.25,W,H*0.15);

    const horizonY = H*0.32;
    ctx.fillStyle = '#26262f';
    ctx.beginPath();
    ctx.moveTo(W*0.5-40, horizonY);
    ctx.lineTo(W*0.5+40, horizonY);
    ctx.lineTo(W*0.95, H);
    ctx.lineTo(W*0.05, H);
    ctx.closePath();
    ctx.fill();

    ctx.strokeStyle = 'rgba(255,210,63,0.55)';
    ctx.lineWidth = 3;
    const scrollOffset = (elapsed*speed*0.02) % 40;
    for(let div=1; div<=2; div++){
      const t = div/3;
      ctx.beginPath();
      for(let zz=1; zz>=0; zz-=0.02){
        const topX = W*0.5-40 + ((W*0.5+40)-(W*0.5-40))*t;
        const nearX = W*0.05 + (W*0.95-W*0.05)*t;
        const x = topX + (nearX-topX)*(1-zz);
        const y = horizonY + (H-horizonY)*(1-zz);
        if(zz===1) ctx.moveTo(x,y); else ctx.lineTo(x,y);
      }
      ctx.setLineDash([18,18]);
      ctx.lineDashOffset = -scrollOffset;
      ctx.stroke();
      ctx.setLineDash([]);
    }

    const flash = Math.sin(elapsed*10) > 0 ? 'rgba(255,40,60,0.25)' : 'rgba(60,140,255,0.25)';
    ctx.fillStyle = flash;
    ctx.fillRect(0,0,W,H);
  }

  function drawPlayer(){
    const px = playerX;
    let py = playerY;
    let squashY = 1;
    if(sliding) squashY = 0.55;
    const bodyH = 70*squashY;
    const bodyW = 34;

    ctx.fillStyle = 'rgba(0,0,0,0.4)';
    ctx.beginPath();
    ctx.ellipse(px, GROUND_Y+18, 26, 8, 0,0,Math.PI*2);
    ctx.fill();

    const runCycle = Math.sin(elapsed*18);
    ctx.strokeStyle = '#1a1a1a';
    ctx.lineWidth = 8;
    if(!sliding){
      ctx.beginPath();
      ctx.moveTo(px-8, py+ bodyH*0.5 - 10);
      ctx.lineTo(px-8+runCycle*10, py+bodyH*0.5+22);
      ctx.moveTo(px+8, py+ bodyH*0.5 -10);
      ctx.lineTo(px+8-runCycle*10, py+bodyH*0.5+22);
      ctx.stroke();
    }

    ctx.fillStyle = '#ff2b4c';
    ctx.fillRect(px-bodyW/2, py-bodyH, bodyW, bodyH);

    ctx.fillStyle = '#f2c48a';
    ctx.beginPath();
    ctx.arc(px, py-bodyH-14, 15, 0, Math.PI*2);
    ctx.fill();

    ctx.fillStyle = '#222';
    ctx.fillRect(px+bodyW/2-2, py-bodyH+10, 20, 6);
  }

  function drawObstacle(ob){
    const x = laneXatZ(ob.lane, ob.z);
    const y = yAtZ(ob.z);
    const s = scaleAtZ(ob.z);

    if(ob.type === 'car'){
      const w = 70*s, h = 40*s;
      ctx.fillStyle = '#3355ff';
      ctx.fillRect(x-w/2, y-h, w, h);
      ctx.fillStyle = '#aee5ff';
      ctx.fillRect(x-w/2+w*0.15, y-h+h*0.15, w*0.7, h*0.35);
      ctx.fillStyle='#fff9c4';
      ctx.fillRect(x-w/2, y-h*0.3, w*0.15, h*0.2);
      ctx.fillRect(x+w/2-w*0.15, y-h*0.3, w*0.15, h*0.2);
    } else if(ob.type === 'barrier'){
      const w = 56*s, h = 22*s;
      ctx.fillStyle = '#ffcc33';
      ctx.fillRect(x-w/2, y-h, w, h);
      ctx.strokeStyle = '#000'; ctx.lineWidth = Math.max(1,2*s);
      ctx.strokeRect(x-w/2, y-h, w, h);
    } else if(ob.type === 'beam'){
      const w = 70*s;
      const beamY = y - 90*s;
      ctx.fillStyle = '#8888aa';
      ctx.fillRect(x-w/2, beamY, w, 14*s);
      ctx.fillStyle = '#555577';
      ctx.fillRect(x-w/2, beamY, 6*s, 90*s);
      ctx.fillRect(x+w/2-6*s, beamY, 6*s, 90*s);
    }
  }

  function drawCoin(c){
    const x = laneXatZ(c.lane, c.z);
    const y = yAtZ(c.z) - 50*scaleAtZ(c.z);
    const r = 10*scaleAtZ(c.z);
    ctx.fillStyle = '#ffd23f';
    ctx.beginPath();
    ctx.arc(x,y,r,0,Math.PI*2);
    ctx.fill();
    ctx.strokeStyle='#b8860b'; ctx.lineWidth=2; ctx.stroke();
  }

  function drawParticles(){
    ctx.font = 'bold 18px Segoe UI';
    ctx.textAlign = 'center';
    particles.forEach(p=>{
      ctx.fillStyle = `rgba(255,210,63,${Math.max(0,p.life)})`;
      ctx.fillText(p.text, p.x, p.y);
    });
  }

  function draw(){
    ctx.clearRect(0,0,W,H);
    drawRoad();

    const drawList = [...obstacles.map(o=>({...o,isObs:true})), ...coinPickups.map(c=>({...c,isObs:false}))]
      .sort((a,b)=> b.z - a.z);
    for(const item of drawList){
      if(item.isObs) drawObstacle(item); else if(!item.taken) drawCoin(item);
    }

    drawPlayer();
    drawParticles();
  }

  // ---------- Loop ----------
  let lastTime = performance.now();
  function loop(now){
    const dt = Math.min(0.04, (now-lastTime)/1000);
    lastTime = now;
    if(state === 'running') update(dt);
    draw();
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);
})();
</script>
</body>
</html>

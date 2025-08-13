<!doctype html>
<html lang="zh-Hant">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1" />
<title>躲避方塊 Dodge Squares</title>
<style>
  html,body{margin:0;height:100%;background:#111;color:#eee;font-family:system-ui,-apple-system,Segoe UI,Roboto,"Noto Sans TC",sans-serif}
  #wrap{display:flex;flex-direction:column;align-items:center;gap:12px;padding:12px}
  canvas{width:min(96vw,560px);height:min(72vh,420px);background:#000;border:2px solid #444;border-radius:8px;touch-action:none}
  .row{display:flex;gap:8px;flex-wrap:wrap;justify-content:center}
  button{padding:.6em 1em;border-radius:8px;border:1px solid #666;background:#222;color:#eee;cursor:pointer}
  button:active{transform:translateY(1px)}
  #hud{display:flex;gap:16px;justify-content:center;align-items:center;font-weight:700}
  #tip{font-size:.9em;opacity:.8;text-align:center}
  .pad{display:none;gap:10px}
  .pad button{width:64px;height:64px;font-size:18px}
  @media (max-width:780px){
    .pad{display:flex}
  }
</style>
</head>
<body>
<div id="wrap">
  <h1>躲避方塊 Dodge Squares</h1>
  <div id="hud">
    <div>分數：<span id="score">0</span></div>
    <div>最佳：<span id="best">0</span></div>
    <div>等級：<span id="lvl">1</span></div>
  </div>
  <canvas id="game" width="560" height="420" aria-label="Dodge game canvas"></canvas>
  <div class="row">
    <button id="startBtn">開始 / 重新開始</button>
    <button id="pauseBtn">暫停</button>
  </div>
  <div class="pad" aria-label="touch controls">
    <button data-dx="-1" data-dy="0">←</button>
    <button data-dx="1" data-dy="0">→</button>
    <button data-dx="0" data-dy="-1">↑</button>
    <button data-dx="0" data-dy="1">↓</button>
  </div>
  <div id="tip">
    操作：方向鍵 / WASD；手機可用下方方向鍵。<br>
    規則：操控藍色小方塊躲避紅色障礙物，存活越久分數越高。每 15 秒升一級，速度更快、障礙更多！
  </div>
</div>

<script>
(() => {
  // --- 基本設定 ---
  const cvs = document.getElementById('game');
  const ctx = cvs.getContext('2d');
  const W = cvs.width, H = cvs.height;

  const ui = {
    score: document.getElementById('score'),
    best: document.getElementById('best'),
    lvl: document.getElementById('lvl'),
    start: document.getElementById('startBtn'),
    pause: document.getElementById('pauseBtn'),
    pad: document.querySelector('.pad')
  };

  const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
  const rand = (a, b) => a + Math.random() * (b - a);

  // 儲存最佳紀錄（本機）
  let best = Number(localStorage.getItem('dodge_best') || 0);
  ui.best.textContent = best;

  // --- 遊戲狀態 ---
  const state = {
    running: false,
    paused: false,
    score: 0,
    level: 1,
    t0: 0,
    t: 0,
    lastSpawn: 0,
    player: null,
    obstacles: [],
    particles: []
  };

  // --- 物件 ---
  class Player {
    constructor() {
      this.size = 18;
      this.x = W * 0.5;
      this.y = H * 0.8;
      this.vx = 0;
      this.vy = 0;
      this.speed = 240; // px/s
      this.color = '#4db2ff';
      this.invincible = 0; // 受擊後短暫無敵（秒）
    }
    update(dt, input) {
      const accel = this.speed;
      this.vx = (input.right - input.left) * accel;
      this.vy = (input.down - input.up) * accel;
      this.x = clamp(this.x + this.vx * dt, this.size/2, W - this.size/2);
      this.y = clamp(this.y + this.vy * dt, this.size/2, H - this.size/2);
      if (this.invincible > 0) this.invincible -= dt;
    }
    draw() {
      ctx.save();
      if (this.invincible > 0 && (Math.floor(state.t*20)%2===0)) {
        ctx.globalAlpha = 0.4; // 閃爍效果
      }
      ctx.fillStyle = this.color;
      const s = this.size;
      ctx.fillRect(this.x - s/2, this.y - s/2, s, s);
      ctx.restore();
    }
    hitbox() {
      const s = this.size;
      return {x:this.x - s/2, y:this.y - s/2, w:s, h:s};
    }
  }

  class Obstacle {
    constructor() {
      const edge = Math.floor(rand(0, 4)); // 從四邊生成
      const size = rand(14, 26);
      let x, y, vx, vy;
      const base = 60 + state.level * 40; // 速度基礎
      const speed = rand(base, base + 80);

      if (edge === 0) { // 上
        x = rand(0, W); y = -size; vx = rand(-0.5,0.5)*speed; vy = speed;
      } else if (edge === 1) { // 下
        x = rand(0, W); y = H+size; vx = rand(-0.5,0.5)*speed; vy = -speed;
      } else if (edge === 2) { // 左
        x = -size; y = rand(0, H); vx = speed; vy = rand(-0.5,0.5)*speed;
      } else { // 右
        x = W+size; y = rand(0, H); vx = -speed; vy = rand(-0.5,0.5)*speed;
      }
      this.x=x; this.y=y; this.vx=vx; this.vy=vy; this.size=size;
      this.color = '#ff4d4d';
      this.life = 10; // 最多存在秒數（避免記憶體累積）
    }
    update(dt) {
      this.x += this.vx * dt;
      this.y += this.vy * dt;
      this.life -= dt;
    }
    draw() {
      ctx.fillStyle = this.color;
      const s = this.size;
      ctx.fillRect(this.x - s/2, this.y - s/2, s, s);
    }
    hitbox() {
      const s = this.size;
      return {x:this.x - s/2, y:this.y - s/2, w:s, h:s};
    }
    isOff() {
      const s = this.size;
      return (this.x < -s*2 || this.x > W + s*2 || this.y < -s*2 || this.y > H + s*2 || this.life <= 0);
    }
  }

  class Particle {
    constructor(x,y) {
      this.x=x; this.y=y; this.vx=rand(-60,60); this.vy=rand(-60,60);
      this.life=0.6; this.size=rand(2,4);
    }
    update(dt){ this.x+=this.vx*dt; this.y+=this.vy*dt; this.life-=dt; }
    draw(){ ctx.globalAlpha=Math.max(this.life/0.6,0); ctx.fillStyle='#fff'; ctx.fillRect(this.x,this.y,this.size,this.size); ctx.globalAlpha=1; }
  }

  // --- 輸入 ---
  const input = {left:0,right:0,up:0,down:0};
  const keys = {ArrowLeft:'left',ArrowRight:'right',ArrowUp:'up',ArrowDown:'down',a:'left',d:'right',w:'up',s:'down'};

  addEventListener('keydown', e=>{
    if(keys[e.key]){ input[keys[e.key]]=1; e.preventDefault(); }
    if(e.key===' '){ togglePause(); e.preventDefault(); }
  }, {passive:false});
  addEventListener('keyup', e=>{ if(keys[e.key]) input[keys[e.key]]=0; }, {passive:true});

  // 觸控方向鍵
  ui.pad.querySelectorAll('button').forEach(btn=>{
    const dx = Number(btn.dataset.dx), dy = Number(btn.dataset.dy);
    const press = (on)=>{
      input.left  = dx<0? (on?1:0) : (dx>0?0:input.left);
      input.right = dx>0? (on?1:0) : (dx<0?0:input.right);
      input.up    = dy<0? (on?1:0) : (dy>0?0:input.up);
      input.down  = dy>0? (on?1:0) : (dy<0?0:input.down);
    };
    btn.addEventListener('pointerdown', e=>{ press(true); btn.setPointerCapture(e.pointerId); });
    btn.addEventListener('pointerup', ()=>press(false));
    btn.addEventListener('pointercancel', ()=>press(false));
    btn.addEventListener('pointerleave', ()=>press(false));
  });

  // --- 流程控制 ---
  ui.start.onclick = startGame;
  ui.pause.onclick = togglePause;

  function startGame(){
    Object.assign(state, {
      running: true, paused: false,
      score: 0, level: 1,
      t0: performance.now()/1000,
      t: 0, lastSpawn: 0,
      player: new Player(),
      obstacles: [],
      particles: []
    });
    ui.pause.textContent = '暫停';
    loopId = requestAnimationFrame(loop);
  }

  function togglePause(){
    if(!state.running) return;
    state.paused = !state.paused;
    ui.pause.textContent = state.paused ? '繼續' : '暫停';
    if(!state.paused) {
      // 繼續時重置基準時間，避免時間跳躍
      state.t0 = performance.now()/1000 - state.t;
      loopId = requestAnimationFrame(loop);
    }
  }

  // --- 碰撞 ---
  function aabb(a,b){
    return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
  }

  // --- 生成邏輯 ---
  function spawnObstacles(dt){
    const baseRate = 0.9; // 基礎生成間隔（秒）
    const rate = Math.max(0.28, baseRate - state.level*0.08); // 等級越高間隔越短
    if (state.t - state.lastSpawn >= rate){
      const n = 1 + Math.floor(Math.random() * Math.min(1+Math.floor(state.level/3), 3));
      for(let i=0;i<n;i++) state.obstacles.push(new Obstacle());
      state.lastSpawn = state.t;
    }
  }

  // --- 聲音（簡單的 WebAudio beep，不加載外部檔案） ---
  let audioCtx = null;
  function beep(freq = 440, dur = 0.08, vol = 0.1){
    try{
      audioCtx = audioCtx || new (window.AudioContext || window.webkitAudioContext)();
      const o = audioCtx.createOscillator();
      const g = audioCtx.createGain();
      o.frequency.value = freq; o.type='square';
      g.gain.value = vol;
      o.connect(g).connect(audioCtx.destination);
      o.start();
      setTimeout(()=>{ o.stop(); }, dur*1000);
    }catch(e){/* 部分瀏覽器/使用者設定可能阻擋音訊，忽略即可 */}
  }

  // --- 主迴圈 ---
  let loopId = null;
  function loop(ts){
    if(!state.running) return;
    state.t = performance.now()/1000 - state.t0;
    const dt = Math.min(1/30, (ts - (loop.prevTs||ts))/1000); // 安全上限，避免切回視窗時 dt 過大
    loop.prevTs = ts;

    if(state.paused){
      draw(true);
      return; // 不要排下一幀；繼續時會再排
    }

    // 升級：每 15 秒 +1
    const newLvl = 1 + Math.floor(state.t / 15);
    if(newLvl !== state.level){
      state.level = newLvl;
      ui.lvl.textContent = state.level;
      // 小升級特效
      for(let i=0;i<60;i++) state.particles.push(new Particle(state.player.x, state.player.y));
      beep(880, 0.08, 0.06);
    }

    // 更新
    state.player.update(dt, input);
    spawnObstacles(dt);
    for(const o of state.obstacles) o.update(dt);
    state.obstacles = state.obstacles.filter(o=>!o.isOff());

    // 碰撞偵測
    const pbox = state.player.hitbox();
    if(state.player.invincible <= 0){
      for(const o of state.obstacles){
        if(aabb(pbox, o.hitbox())){
          // 受擊：扣大分並短暫無敵；若分數<0則結束
          for(let i=0;i<40;i++) state.particles.push(new Particle(state.player.x, state.player.y));
          beep(200, 0.1, 0.12);
          state.player.invincible = 1.2;
          state.score = Math.max(0, state.score - 80);
          break;
        }
      }
    }

    // 計分：存活時間與等級加成
    state.score += dt * (10 + state.level*2);
    ui.score.textContent = Math.floor(state.score);

    // 粒子
    for(const pt of state.particles) pt.update(dt);
    state.particles = state.particles.filter(pt=>pt.life>0);

    // 繪製
    draw(false);

    loopId = requestAnimationFrame(loop);
  }

  function draw(paused){
    // 背景漸層
    const g = ctx.createLinearGradient(0,0,0,H);
    g.addColorStop(0,'#000');
    g.addColorStop(1,'#0a0a0a');
    ctx.fillStyle = g;
    ctx.fillRect(0,0,W,H);

    // 邊框裝飾
    ctx.strokeStyle = '#222';
    ctx.lineWidth = 2;
    ctx.strokeRect(1,1,W-2,H-2);

    // 粒子
    for(const pt of state.particles) pt.draw();

    // 物件
    for(const o of state.obstacles) o.draw();
    state.player?.draw();

    // UI 前景
    ctx.fillStyle = '#888';
    ctx.font = '12px system-ui';
    ctx.fillText('SPACE 暫停/繼續', 12, H-12);

    if(!state.running) return;

    if(paused){
      overlay('暫停中', '按下「繼續」或空白鍵回到遊戲');
    }
  }

  function overlay(title, subtitle){
    ctx.save();
    ctx.fillStyle = 'rgba(0,0,0,.6)';
    ctx.fillRect(0,0,W,H);
    ctx.fillStyle = '#fff';
    ctx.textAlign = 'center';
    ctx.font = 'bold 28px system-ui';
    ctx.fillText(title, W/2, H/2 - 10);
    ctx.font = '16px system-ui';
    ctx.fillText(subtitle, W/2, H/2 + 18);
    ctx.restore();
  }

  // --- 結束條件：當玩家多次受擊讓分數歸零且再次受擊 ---
  // 為了單純，這裡改為：當受擊時若分數已是 0，視為 Game Over
  const _hitCheck = setInterval(()=>{
    if(!state.running) return;
    if(state.player?.invincible > 0) {
      // 如果這次受擊前分數已是 0，就結束
      if(Math.floor(state.score) === 0){
        gameOver();
      }
    }
  }, 60);

  function gameOver(){
    state.running = false;
    cancelAnimationFrame(loopId);
    // 更新最佳
    if(state.score > best){
      best = Math.floor(state.score);
      localStorage.setItem('dodge_best', best);
      ui.best.textContent = best;
    }
    // 最終畫面
    overlay('遊戲結束', `分數：${Math.floor(state.score)} 　等級：${state.level}　按「開始」再來！`);
    beep(140, 0.16, 0.15);
  }

  // 初始畫面
  ctx.fillStyle = '#000'; ctx.fillRect(0,0,W,H);
  overlay('躲避方塊', '按「開始 / 重新開始」開始遊戲');
})();
</script>
</body>
</html>

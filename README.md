<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>テトリス</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    background: #1a1a2e;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    font-family: 'Courier New', monospace;
    color: #eee;
  }
  #game-wrapper { display: flex; gap: 20px; align-items: flex-start; }
  canvas#board {
    border: 3px solid #e94560;
    display: block;
    background: #0f3460;
    box-shadow: 0 0 20px #e94560;
  }
  #side-panel { width: 140px; display: flex; flex-direction: column; gap: 16px; }
  .panel-box {
    background: #0f3460;
    border: 2px solid #e94560;
    border-radius: 6px;
    padding: 12px;
    text-align: center;
  }
  .panel-label { font-size: 11px; text-transform: uppercase; letter-spacing: 2px; color: #e94560; margin-bottom: 6px; }
  .panel-value { font-size: 24px; font-weight: bold; color: #fff; }
  #overlay {
    position: absolute; top: 0; left: 0; width: 100%; height: 100%;
    background: rgba(10,20,50,0.85);
    display: flex; flex-direction: column; justify-content: center; align-items: center; gap: 16px;
  }
  #overlay h1 { font-size: 36px; color: #e94560; text-shadow: 0 0 10px #e94560; }
  #start-btn {
    padding: 12px 32px; font-size: 16px; font-family: inherit;
    background: #e94560; color: #fff; border: none; border-radius: 4px; cursor: pointer;
  }
  #start-btn:hover { background: #c73652; }
  .ctrl-row { font-size: 11px; color: #aaa; line-height: 2; }
  .ctrl-key { background: #1a1a2e; border: 1px solid #e94560; padding: 1px 5px; border-radius: 3px; }
  #time-bar-wrap { width: 100%; height: 8px; background: #1a1a2e; border-radius: 4px; overflow: hidden; margin-top: 6px; }
  #time-bar { height: 100%; width: 100%; background: #e94560; border-radius: 4px; }
</style>
</head>
<body>
<div id="game-wrapper">
  <div style="position:relative">
    <canvas id="board" width="300" height="600"></canvas>
    <div id="overlay">
      <h1>テトリス</h1>
      <button id="start-btn">スタート</button>
      <p id="game-over-msg" style="display:none;color:#e94560;font-size:18px;"></p>
    </div>
  </div>
  <div id="side-panel">
    <div class="panel-box">
      <div class="panel-label">SCORE</div>
      <div class="panel-value" id="score">0</div>
    </div>
    <div class="panel-box">
      <div class="panel-label">SPEED</div>
      <div class="panel-value" id="speed-lv">1</div>
      <div id="time-bar-wrap"><div id="time-bar"></div></div>
    </div>
    <div class="panel-box">
      <div class="panel-label">LINES</div>
      <div class="panel-value" id="lines">0</div>
    </div>
    <div class="panel-box">
      <div class="panel-label">NEXT</div>
      <canvas id="next-canvas" width="120" height="80" style="display:block;margin:0 auto"></canvas>
    </div>
    <div class="panel-box">
      <div class="panel-label">操作方法</div>
      <div class="ctrl-row"><span class="ctrl-key">← →</span> 移動</div>
      <div class="ctrl-row"><span class="ctrl-key">↑</span> 右回転</div>
      <div class="ctrl-row"><span class="ctrl-key">↓</span> 左回転</div>
      <div class="ctrl-row"><span class="ctrl-key">Space</span> 即落下</div>
      <div class="ctrl-row"><span class="ctrl-key">P</span> 一時停止</div>
    </div>
  </div>
</div>
<script>
(function(){
const COLS=10,ROWS=20,B=30;
const cv=document.getElementById('board'),ctx=cv.getContext('2d');
const nc=document.getElementById('next-canvas'),nctx=nc.getContext('2d');
const overlay=document.getElementById('overlay');
const startBtn=document.getElementById('start-btn');
const gameOverMsg=document.getElementById('game-over-msg');
const scoreEl=document.getElementById('score');
const speedLvEl=document.getElementById('speed-lv');
const timeBarEl=document.getElementById('time-bar');
const linesEl=document.getElementById('lines');

const COLORS=['','#00f0f0','#0000f0','#f0a000','#f0f000','#00f000','#a000f0','#f00000'];
const SHAPES=[
  [],
  [[1,1,1,1]],
  [[1,0,0],[1,1,1]],
  [[0,0,1],[1,1,1]],
  [[1,1],[1,1]],
  [[0,1,1],[1,1,0]],
  [[0,1,0],[1,1,1]],
  [[1,1,0],[0,1,1]]
];

const SPEED_INTERVAL=10000;
const BASE_DROP=1000;
const MIN_DROP=60;

let board,piece,next,score,lines,running,paused,dropInterval,lastT,raf;
let gameStartTime,speedLevel,pauseStartTime,totalPausedTime;

const mkBoard=()=>Array.from({length:ROWS},()=>Array(COLS).fill(0));
const mkPiece=()=>{
  const id=Math.floor(Math.random()*7)+1;
  const s=SHAPES[id].map(r=>[...r]);
  return{id,s,x:Math.floor((COLS-s[0].length)/2),y:0};
};
const rotR=s=>{
  const R=s.length,C=s[0].length,n=Array.from({length:C},()=>Array(R).fill(0));
  for(let r=0;r<R;r++)for(let c=0;c<C;c++)n[c][R-1-r]=s[r][c];
  return n;
};
const rotL=s=>rotR(rotR(rotR(s)));

const valid=(p,dx=0,dy=0,sh=null)=>{
  const s=sh||p.s;
  for(let r=0;r<s.length;r++)for(let c=0;c<s[r].length;c++){
    if(!s[r][c])continue;
    const nx=p.x+c+dx,ny=p.y+r+dy;
    if(nx<0||nx>=COLS||ny>=ROWS)return false;
    if(ny>=0&&board[ny][nx])return false;
  }
  return true;
};
const merge=()=>{
  for(let r=0;r<piece.s.length;r++)
    for(let c=0;c<piece.s[r].length;c++)
      if(piece.s[r][c])board[piece.y+r][piece.x+c]=piece.id;
};
const clearL=()=>{
  let n=0;
  for(let r=ROWS-1;r>=0;r--){
    if(board[r].every(v=>v)){board.splice(r,1);board.unshift(Array(COLS).fill(0));n++;r++;}
  }
  if(n){
    const pts=[0,100,300,500,800];
    score+=(pts[n]||800);
    lines+=n;
    scoreEl.textContent=score;
    linesEl.textContent=lines;
  }
};
const doRotate=fn=>{
  const r2=fn(piece.s);
  if(valid(piece,0,0,r2))piece.s=r2;
  else if(valid(piece,1,0,r2)){piece.s=r2;piece.x++;}
  else if(valid(piece,-1,0,r2)){piece.s=r2;piece.x--;}
  else if(valid(piece,2,0,r2)){piece.s=r2;piece.x+=2;}
  else if(valid(piece,-2,0,r2)){piece.s=r2;piece.x-=2;}
};
const drawBlock=(c,x,y,sz,c2)=>{
  sz=sz||B; c2=c2||ctx;
  c2.fillStyle=COLORS[c];
  c2.fillRect(x+1,y+1,sz-2,sz-2);
  c2.fillStyle='rgba(255,255,255,0.3)';
  c2.fillRect(x+1,y+1,sz-2,5);
  c2.fillStyle='rgba(0,0,0,0.2)';
  c2.fillRect(x+1,y+sz-6,sz-2,5);
};
const draw=()=>{
  ctx.fillStyle='#0f3460';ctx.fillRect(0,0,cv.width,cv.height);
  ctx.strokeStyle='rgba(255,255,255,0.04)';ctx.lineWidth=0.5;
  for(let c=0;c<=COLS;c++){ctx.beginPath();ctx.moveTo(c*B,0);ctx.lineTo(c*B,ROWS*B);ctx.stroke();}
  for(let r=0;r<=ROWS;r++){ctx.beginPath();ctx.moveTo(0,r*B);ctx.lineTo(COLS*B,r*B);ctx.stroke();}
  for(let r=0;r<ROWS;r++)for(let c=0;c<COLS;c++)if(board[r][c])drawBlock(board[r][c],c*B,r*B);
  let gy=piece.y;while(valid(piece,0,gy-piece.y+1))gy++;
  ctx.globalAlpha=0.18;
  for(let r=0;r<piece.s.length;r++)for(let c=0;c<piece.s[r].length;c++)if(piece.s[r][c])drawBlock(piece.id,(piece.x+c)*B,(gy+r)*B);
  ctx.globalAlpha=1;
  for(let r=0;r<piece.s.length;r++)for(let c=0;c<piece.s[r].length;c++)if(piece.s[r][c])drawBlock(piece.id,(piece.x+c)*B,(piece.y+r)*B);
};
const drawNext=()=>{
  nctx.fillStyle='#0f3460';nctx.fillRect(0,0,nc.width,nc.height);
  const sz=20,sh=next.s;
  const ox=Math.floor((6-sh[0].length)/2),oy=Math.floor((4-sh.length)/2);
  for(let r=0;r<sh.length;r++)for(let c=0;c<sh[r].length;c++)if(sh[r][c])drawBlock(next.id,(ox+c)*sz,(oy+r)*sz,sz,nctx);
};
const updateSpeed=elapsed=>{
  const nl=Math.floor(elapsed/SPEED_INTERVAL)+1;
  if(nl!==speedLevel){
    speedLevel=nl;
    speedLvEl.textContent=speedLevel;
    dropInterval=Math.max(MIN_DROP,BASE_DROP-(speedLevel-1)*60);
  }
  const progress=(elapsed%SPEED_INTERVAL)/SPEED_INTERVAL;
  timeBarEl.style.width=(100-progress*100)+'%';
};
const loop=ts=>{
  if(!running)return;
  if(paused){raf=requestAnimationFrame(loop);return;}
  if(!lastT)lastT=ts;
  const elapsed=ts-gameStartTime-totalPausedTime;
  updateSpeed(elapsed);
  if(ts-lastT>=dropInterval){
    lastT=ts;
    if(valid(piece,0,1)){piece.y++;}
    else{merge();clearL();piece=next;next=mkPiece();drawNext();if(!valid(piece)){endGame();return;}}
  }
  draw();
  raf=requestAnimationFrame(loop);
};
const endGame=()=>{
  running=false;cancelAnimationFrame(raf);
  gameOverMsg.textContent='ゲームオーバー！スコア: '+score;
  gameOverMsg.style.display='block';
  startBtn.textContent='もう一度';
  overlay.style.display='flex';
};
const startGame=()=>{
  board=mkBoard();score=0;lines=0;
  dropInterval=BASE_DROP;speedLevel=1;
  lastT=0;totalPausedTime=0;
  scoreEl.textContent='0';speedLvEl.textContent='1';linesEl.textContent='0';
  timeBarEl.style.width='100%';
  piece=mkPiece();next=mkPiece();drawNext();
  running=true;paused=false;
  gameOverMsg.style.display='none';overlay.style.display='none';
  if(raf)cancelAnimationFrame(raf);
  raf=requestAnimationFrame(ts=>{gameStartTime=ts;lastT=ts;loop(ts);});
};
document.addEventListener('keydown',e=>{
  if(!running)return;
  if(e.key==='p'||e.key==='P'){
    if(!paused)pauseStartTime=performance.now();
    else totalPausedTime+=performance.now()-pauseStartTime;
    paused=!paused;
    return;
  }
  if(paused)return;
  if(['ArrowLeft','ArrowRight','ArrowDown','ArrowUp',' '].includes(e.key))e.preventDefault();
  switch(e.key){
    case'ArrowLeft':if(valid(piece,-1))piece.x--;break;
    case'ArrowRight':if(valid(piece,1))piece.x++;break;
    case'ArrowUp':doRotate(rotR);break;
    case'ArrowDown':doRotate(rotL);break;
    case' ':while(valid(piece,0,1)){piece.y++;score+=2;}scoreEl.textContent=score;lastT=0;break;
  }
  draw();
});
startBtn.addEventListener('click',startGame);
ctx.fillStyle='#0f3460';ctx.fillRect(0,0,cv.width,cv.height);
})();
</script>
</body>
</html>

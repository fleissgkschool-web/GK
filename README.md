<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>GK Positioning Visualizer</title>
<style>
  body { background:#111; color:#fff; font-family:sans-serif; text-align:center; }
  #wrap { display:flex; justify-content:center; gap:20px; margin-top:10px; }
  canvas { background:#2e7d32; border:2px solid #fff; }
  #state { margin-top:10px; font-size:14px; }
</style>
</head>
<body>

<h1>GK Positioning Visualizer</h1>

<div id="wrap">
  <canvas id="top" width="800" height="500"></canvas>
  <canvas id="front" width="350" height="500"></canvas>
</div>

<div id="state"></div>

<script>
/* ============================
   基本設定（表示保証版）
============================ */
const top = document.getElementById("top");
const tctx = top.getContext("2d");
const front = document.getElementById("front");
const fctx = front.getContext("2d");
const stateDiv = document.getElementById("state");

// 俯瞰図スケール（縮小してズレ防止）
const M = 4; // ★ 1m = 4px（ズレ防止）
const pitchL = 105 * M;
const pitchW = 68 * M;

// ★ 描画ズレ防止のため offset を 0 に固定
const offsetX = 0;
const offsetY = 0;

// ゴール（俯瞰）
const goalWidth = 7.32;      // m
const goalHeight = 2.44;     // m
const goalYCenter = pitchW / 2;
const goalX = offsetX + 10;  // 左端から少し余白

// ゴールポスト（俯瞰）
const leftPost = { x: goalX, y: offsetY + goalYCenter - (goalWidth * M) / 2 };
const rightPost = { x: goalX, y: offsetY + goalYCenter + (goalWidth * M) / 2 };

// ボール・GK（俯瞰）
let ball = { x: goalX + 20 * M, y: offsetY + goalYCenter };
let gk   = { x: goalX + 6 * M,  y: offsetY + goalYCenter };

// 等身大設定
const GK_HEIGHT = 1.75; // m
const GK_WIDTH  = 0.50; // m
const BALL_DIAM = 0.22; // m
const CAM_HEIGHT = 1.70; // m（ボール位置から170cm視点）

let dragBall = false;
let dragGK = false;

function normalize(v){ const l=Math.hypot(v.x,v.y); return {x:v.x/l,y:v.y/l}; }

/* ============================
   俯瞰図描画（ズレ防止版）
============================ */
function drawPitchTop(){
  tctx.clearRect(0,0,top.width,top.height);
  tctx.fillStyle="#2e7d32";
  tctx.fillRect(0,0,top.width,top.height);

  // ピッチ枠
  tctx.strokeStyle="#fff";
  tctx.lineWidth = 2;
  tctx.strokeRect(offsetX, offsetY, pitchL, pitchW);

  // センターライン
  tctx.beginPath();
  tctx.moveTo(offsetX + pitchL/2, offsetY);
  tctx.lineTo(offsetX + pitchL/2, offsetY + pitchW);
  tctx.stroke();

  // センターサークル
  const centerX = offsetX + pitchL/2;
  const centerY = offsetY + pitchW/2;
  tctx.beginPath();
  tctx.arc(centerX, centerY, 9.15*M, 0, Math.PI*2);
  tctx.stroke();

  // ペナルティエリア
  const boxDepth = 16.5 * M;
  const boxWidth = 40.32 * M;
  tctx.strokeStyle="#ffffffaa";
  tctx.strokeRect(
    goalX,
    offsetY + goalYCenter - boxWidth/2,
    boxDepth,
    boxWidth
  );

  // ゴールエリア
  const gaDepth = 5.5 * M;
  const gaWidth = 18.32 * M;
  tctx.strokeRect(
    goalX,
    offsetY + goalYCenter - gaWidth/2,
    gaDepth,
    gaWidth
  );

  // ゴールライン
  tctx.strokeStyle="#fff";
  tctx.lineWidth=4;
  tctx.beginPath();
  tctx.moveTo(leftPost.x, leftPost.y);
  tctx.lineTo(rightPost.x, rightPost.y);
  tctx.stroke();
}
/* ============================
   シュートコース（ゴール4角 → ボール）
============================ */
function drawShootingLinesTop(){
  const goalTopY = offsetY + goalYCenter - (goalWidth * M)/2;
  const goalBotY = offsetY + goalYCenter + (goalWidth * M)/2;

  tctx.strokeStyle="#ffeb3b";
  tctx.lineWidth=2;

  // 左下角（俯瞰では同じ位置）
  tctx.beginPath();
  tctx.moveTo(ball.x, ball.y);
  tctx.lineTo(goalX, goalTopY);
  tctx.stroke();

  // 左上角（俯瞰では同じ位置）
  tctx.beginPath();
  tctx.moveTo(ball.x, ball.y);
  tctx.lineTo(goalX, goalBotY);
  tctx.stroke();
}

/* ============================
   GK横ライン（シュートコース幅）
============================ */
function drawGKHorizontalCoverageTop(){
  const goalTopY = offsetY + goalYCenter - (goalWidth * M)/2;
  const goalBotY = offsetY + goalYCenter + (goalWidth * M)/2;

  const yGK = gk.y;

  function intersect(postY){
    const dy = postY - ball.y;
    const dx = goalX - ball.x;
    const t = (yGK - ball.y)/dy;
    return { x: ball.x + dx*t, y: yGK };
  }

  const p1 = intersect(goalTopY);
  const p2 = intersect(goalBotY);

  tctx.strokeStyle="#e0f7fa";
  tctx.lineWidth=4;
  tctx.beginPath();
  tctx.moveTo(p1.x, p1.y);
  tctx.lineTo(p2.x, p2.y);
  tctx.stroke();
}

/* ============================
   ボール・GK（俯瞰）
============================ */
function drawBallTop(){
  tctx.fillStyle="red";
  tctx.beginPath();
  tctx.arc(ball.x, ball.y, (BALL_DIAM*M)/2, 0, Math.PI*2);
  tctx.fill();
}

function drawGKTop(){
  const v = { x: ball.x - gk.x, y: ball.y - gk.y };
  const n = normalize(v);
  const angle = Math.atan2(n.y, n.x);

  const w = GK_WIDTH * M;
  const h = GK_HEIGHT * M * 0.4;

  tctx.save();
  tctx.translate(gk.x, gk.y);
  tctx.rotate(angle);
  tctx.fillStyle="#03a9f4";
  tctx.fillRect(-w/2,-h/2,w,h);
  tctx.restore();

  tctx.strokeStyle="#ffffffaa";
  tctx.lineWidth=2;
  tctx.beginPath();
  tctx.moveTo(gk.x, gk.y);
  tctx.lineTo(ball.x, ball.y);
  tctx.stroke();
}

/* ============================
   3D→正面図への投影
============================ */
function projectToGoalPlane(px, py, pz){
  const cam = {
    x: (ball.x - offsetX)/M,
    y: (ball.y - offsetY)/M,
    z: CAM_HEIGHT
  };

  const pt  = {
    x: (px - offsetX)/M,
    y: (py - offsetY)/M,
    z: pz
  };

  const planeX = (goalX - offsetX)/M;
  const t = (planeX - cam.x) / (pt.x - cam.x);
  const y = cam.y + t*(pt.y - cam.y);
  const z = cam.z + t*(pt.z - cam.z);

  const scaleY = front.height / (goalWidth*1.5);
  const scaleZ = front.height / (goalHeight*2.0);

  const fy = front.height/2 + (y - goalYCenter/M)*scaleY;
  const fz = front.height - (z * scaleZ);

  return { x: front.width*0.5, y: fy, z: fz };
}

/* ============================
   正面図描画
============================ */
function drawGoalFront(){
  fctx.clearRect(0,0,front.width,front.height);
  fctx.fillStyle="#000";
  fctx.fillRect(0,0,front.width,front.height);

  const gw = goalWidth;
  const gh = goalHeight;
  const sx = front.width*0.5;
  const sy = front.height*0.5;
  const scaleY = front.height / (gw*1.5);
  const scaleZ = front.height / (gh*2.0);

  const topY = sy - (gw/2)*scaleY;
  const botY = sy + (gw/2)*scaleY;
  const barZ = front.height - gh*scaleZ;

  fctx.strokeStyle="#fff";
  fctx.lineWidth=2;
  fctx.beginPath();
  fctx.moveTo(sx, topY);
  fctx.lineTo(sx, botY);
  fctx.moveTo(sx, topY);
  fctx.lineTo(sx, barZ);
  fctx.moveTo(sx, botY);
  fctx.lineTo(sx, barZ);
  fctx.stroke();
}

/* ============================
   GK・ボール投影（正面図）
============================ */
function drawGKFront(){
  const gTop3D = projectToGoalPlane(gk.x, gk.y, GK_HEIGHT);
  const gBot3D = projectToGoalPlane(gk.x, gk.y, 0);

  const gMidZ = (gTop3D.z + gBot3D.z)/2;
  const gH = Math.abs(gTop3D.z - gBot3D.z);

  const gw = goalWidth;
  const scaleY = front.height / (gw*1.5);
  const gW = GK_WIDTH * scaleY * 3;

  fctx.fillStyle="#03a9f4";
  fctx.fillRect(front.width*0.5 - gW/2, gMidZ - gH/2, gW, gH);
}

function drawBallFront(){
  const b3D = projectToGoalPlane(ball.x, ball.y, BALL_DIAM/2);
  fctx.fillStyle="red";
  fctx.beginPath();
  fctx.arc(b3D.x, b3D.z, 4, 0, Math.PI*2);
  fctx.fill();
}

/* ============================
   状態表示
============================ */
function updateState(){
  const distPx = Math.hypot(ball.x - gk.x, ball.y - gk.y);
  const distM = distPx / M;

  let posture="", action="";
  if(distM > 5.5){ posture="通常の構え"; action="ポジショニング継続"; }
  else if(distM >=5.0 && distM<=5.5){ posture="やや低い構え"; action="フロントダイブで奪う距離"; }
  else{ posture="低い構え（ブロッキング）"; action="ブロッキングで対応すべき距離"; }

  stateDiv.textContent =
    `ボールとの距離: ${distM.toFixed(2)}m ｜ 姿勢: ${posture} ｜ アクション: ${action}`;
}

/* ============================
   全描画
============================ */
function drawAll(){
  drawPitchTop();
  drawShootingLinesTop();
  drawGKHorizontalCoverageTop();
  drawBallTop();
  drawGKTop();

  drawGoalFront();
  drawGKFront();
  drawBallFront();

  updateState();
}

/* ============================
   ドラッグ操作
============================ */
top.addEventListener("mousedown",e=>{
  const r=top.getBoundingClientRect();
  const x=e.clientX-r.left, y=e.clientY-r.top;

  if(Math.hypot(x-ball.x,y-ball.y)<15) dragBall=true;
  else if(Math.hypot(x-gk.x,y-gk.y)<20) dragGK=true;
});

top.addEventListener("mousemove",e=>{
  const r=top.getBoundingClientRect();
  const x=e.clientX-r.left, y=e.clientY-r.top;

  if(dragBall){
    ball.x=x; ball.y=y;
  }else if(dragGK){
    gk.x=x; gk.y=y;
  }
});

top.addEventListener("mouseup",()=>{ dragBall=false; dragGK=false; });
top.addEventListener("mouseleave",()=>{ dragBall=false; dragGK=false; });

function loop(){ drawAll(); requestAnimationFrame(loop); }
loop();

</script>
</body>
</html>

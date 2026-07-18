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
    <canvas id="top" width="600" height="400"></canvas>
    <canvas id="front" width="300" height="400"></canvas>
  </div>
  <div id="state"></div>

  <script>
    const top = document.getElementById("top");
    const tctx = top.getContext("2d");
    const front = document.getElementById("front");
    const fctx = front.getContext("2d");
    const stateDiv = document.getElementById("state");

    const M = 5; // 1m = 5px（俯瞰図）
    const pitchL = 105 * M;
    const pitchW = 68 * M;

    // ゴール（俯瞰）
    const goalWidth = 7.32;      // m
    const goalHeight = 2.44;     // m
    const goalYCenter = pitchW / 2;
    const goalX = 5 * M;         // 左端から少し余白
    const leftPost = { x: goalX, y: goalYCenter - (goalWidth * M) / 2 };
    const rightPost = { x: goalX, y: goalYCenter + (goalWidth * M) / 2 };

    // ボール・GK（俯瞰）
    let ball = { x: goalX + 20 * M, y: goalYCenter }; // 20m前
    let gk   = { x: goalX + 6 * M,  y: goalYCenter }; // 6m前

    const GK_HEIGHT = 1.75; // m
    const GK_WIDTH  = 0.5;  // m
    const BALL_DIAM = 0.22; // m
    const CAM_HEIGHT = 1.70; // m（ボール位置から170cm）

    let dragBall = false;
    let dragGK = false;

    function normalize(v){ const l=Math.hypot(v.x,v.y); return {x:v.x/l,y:v.y/l}; }

    // 俯瞰図描画
    function drawTop(){
      tctx.clearRect(0,0,top.width,top.height);
      tctx.fillStyle="#2e7d32";
      tctx.fillRect(0,0,top.width,top.height);

      // ピッチ枠
      tctx.strokeStyle="#fff";
      tctx.strokeRect(0,(top.height-pitchW)/2,pitchL,pitchW);

      const offsetY = (top.height-pitchW)/2;

      // ゴール
      tctx.lineWidth=4;
      tctx.beginPath();
      tctx.moveTo(leftPost.x, leftPost.y+offsetY);
      tctx.lineTo(rightPost.x, rightPost.y+offsetY);
      tctx.stroke();

      // ペナルティエリア
      const boxDepth = 16.5 * M;
      const boxWidth = 40.32 * M;
      tctx.strokeStyle="#ffffffaa";
      tctx.strokeRect(
        goalX,
        goalYCenter - boxWidth/2 + offsetY,
        boxDepth,
        boxWidth
      );

      // ゴールエリア
      const gaDepth = 5.5 * M;
      const gaWidth = 18.32 * M;
      tctx.strokeRect(
        goalX,
        goalYCenter - gaWidth/2 + offsetY,
        gaDepth,
        gaWidth
      );

      // シュートコース（4角からボールへ）
      const goalTop = goalYCenter - (goalWidth * M)/2;
      const goalBot = goalYCenter + (goalWidth * M)/2;
      tctx.strokeStyle="#ffeb3b";
      tctx.lineWidth=2;
      tctx.beginPath();
      tctx.moveTo(ball.x, ball.y+offsetY);
      tctx.lineTo(goalX, goalTop+offsetY);
      tctx.moveTo(ball.x, ball.y+offsetY);
      tctx.lineTo(goalX, goalBot+offsetY);
      tctx.stroke();

      // GK横ライン（シュートコース幅）
      const yGK = gk.y;
      function intersect(postY){
        const dy = postY - ball.y;
        const dx = goalX - ball.x;
        const t = (yGK - ball.y)/dy;
        return { x: ball.x + dx*t, y: yGK };
      }
      const p1 = intersect(goalTop);
      const p2 = intersect(goalBot);
      tctx.strokeStyle="#e0f7fa";
      tctx.lineWidth=4;
      tctx.beginPath();
      tctx.moveTo(p1.x, p1.y+offsetY);
      tctx.lineTo(p2.x, p2.y+offsetY);
      tctx.stroke();

      // ボール
      tctx.fillStyle="red";
      tctx.beginPath();
      tctx.arc(ball.x, ball.y+offsetY, (BALL_DIAM*M)/2, 0, Math.PI*2);
      tctx.fill();

      // GK（向きはボール方向）
      const v = { x: ball.x - gk.x, y: ball.y - gk.y };
      const n = normalize(v);
      const angle = Math.atan2(n.y, n.x);

      const w = GK_WIDTH * M;
      const h = GK_HEIGHT * M * 0.4; // 俯瞰なので高さを少しだけ表現
      tctx.save();
      tctx.translate(gk.x, gk.y+offsetY);
      tctx.rotate(angle);
      tctx.fillStyle="#03a9f4";
      tctx.fillRect(-w/2,-h/2,w,h);
      tctx.restore();
    }

    // 3D→正面図への簡易投影
    function projectToGoalPlane(px, py, pz){
      // カメラ＝ボール位置＋高さ
      const cam = { x: ball.x/M, y: (ball.y - (top.height-pitchW)/2)/M, z: CAM_HEIGHT };
      const pt  = { x: px/M, y: (py - (top.height-pitchW)/2)/M, z: pz };

      const planeX = goalX/M; // ゴール面 x
      const t = (planeX - cam.x) / (pt.x - cam.x);
      const y = cam.y + t*(pt.y - cam.y);
      const z = cam.z + t*(pt.z - cam.z);

      // ゴール正面図座標へ変換
      const scaleY = front.height / (goalWidth*1.5);   // 余白込み
      const scaleZ = front.height / (goalHeight*2.0);  // 余白込み
      const fy = front.height/2 + (y - goalYCenter/M)*scaleY;
      const fz = front.height - (z * scaleZ);
      return { x: front.width*0.5, y: fy, z: fz }; // xは中央固定（簡易）
    }

    // 正面図描画
    function drawFront(){
      fctx.clearRect(0,0,front.width,front.height);
      fctx.fillStyle="#000";
      fctx.fillRect(0,0,front.width,front.height);

      // ゴール枠
      fctx.strokeStyle="#fff";
      fctx.lineWidth=2;
      const gw = goalWidth;
      const gh = goalHeight;
      const sx = front.width*0.5;
      const sy = front.height*0.5;
      const scaleY = front.height / (gw*1.5);
      const scaleZ = front.height / (gh*2.0);

      const topY = sy - (gw/2)*scaleY;
      const botY = sy + (gw/2)*scaleY;
      const barZ = front.height - gh*scaleZ;

      fctx.beginPath();
      fctx.moveTo(sx, topY);
      fctx.lineTo(sx, botY);
      fctx.moveTo(sx, topY);
      fctx.lineTo(sx, barZ);
      fctx.moveTo(sx, botY);
      fctx.lineTo(sx, barZ);
      fctx.stroke();

      // GK投影（175cm × 50cm）
      const offsetY = (top.height-pitchW)/2;
      const gTop3D = projectToGoalPlane(gk.x, gk.y+offsetY, GK_HEIGHT);
      const gBot3D = projectToGoalPlane(gk.x, gk.y+offsetY, 0);
      const gMidY = (gTop3D.y + gBot3D.y)/2;
      const gMidZ = (gTop3D.z + gBot3D.z)/2;
      const gH = Math.abs(gTop3D.z - gBot3D.z);
      const gW = GK_WIDTH * scaleY * 3; // 少し強調

      fctx.fillStyle="#03a9f4";
      fctx.fillRect(sx - gW/2, gMidZ - gH/2, gW, gH);

      // ボール投影
      const b3D = projectToGoalPlane(ball.x, ball.y+offsetY, 0.22/2);
      fctx.fillStyle="red";
      fctx.beginPath();
      fctx.arc(b3D.x, b3D.z, 4, 0, Math.PI*2);
      fctx.fill();
    }

    function updateState(){
      const offsetY = (top.height-pitchW)/2;
      const distPx = Math.hypot(ball.x - gk.x, ball.y - gk.y);
      const distM = distPx / M;
      let posture="", action="";
      if(distM > 5.5){ posture="通常の構え"; action="ポジショニング継続"; }
      else if(distM >=5.0 && distM<=5.5){ posture="やや低い構え"; action="フロントダイブで奪う距離"; }
      else{ posture="低い構え（ブロッキング）"; action="ブロッキングで対応すべき距離"; }
      stateDiv.textContent = `ボールとの距離: ${distM.toFixed(2)}m ｜ 姿勢: ${posture} ｜ アクション: ${action}`;
    }

    function drawAll(){
      drawTop();
      drawFront();
      updateState();
    }

    top.addEventListener("mousedown",e=>{
      const r=top.getBoundingClientRect();
      const x=e.clientX-r.left, y=e.clientY-r.top;
      const offsetY=(top.height-pitchW)/2;
      if(Math.hypot(x-ball.x,y-(ball.y+offsetY))<15) dragBall=true;
      else if(Math.hypot(x-gk.x,y-(gk.y+offsetY))<20) dragGK=true;
    });
    top.addEventListener("mousemove",e=>{
      const r=top.getBoundingClientRect();
      const x=e.clientX-r.left, y=e.clientY-r.top;
      const offsetY=(top.height-pitchW)/2;
      if(dragBall){
        ball.x=x; ball.y=y-offsetY;
      }else if(dragGK){
        gk.x=x; gk.y=y-offsetY;
      }
    });
    top.addEventListener("mouseup",()=>{ dragBall=false; dragGK=false; });
    top.addEventListener("mouseleave",()=>{ dragBall=false; dragGK=false; });

    function loop(){ drawAll(); requestAnimationFrame(loop); }
    loop();
  </script>
</body>
</html>

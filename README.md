<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>GK Positioning Visualizer</title>
  <style>
    body {
      background: #1e1e1e;
      color: #fff;
      font-family: sans-serif;
      text-align: center;
    }
    h1 {
      margin-top: 10px;
    }
    canvas {
      background: #2e7d32;
      border: 2px solid #fff;
      margin-top: 10px;
    }
    #state {
      margin-top: 10px;
      font-size: 14px;
    }
  </style>
</head>
<body>
  <h1>GK Positioning Visualizer</h1>
  <canvas id="pitch" width="900" height="600"></canvas>
  <div id="state"></div>

  <script>
    const canvas = document.getElementById("pitch");
    const ctx = canvas.getContext("2d");

    const W = canvas.width;
    const H = canvas.height;

    // 1m ≒ 10px
    const METER = 10;

    // ゴール幅 7.32m → 約73px
    const goalWidth = 7.32 * METER;
    const goalX = 50;
    const goalYCenter = H / 2;
    const leftPost = { x: goalX, y: goalYCenter - goalWidth / 2 };
    const rightPost = { x: goalX, y: goalYCenter + goalWidth / 2 };

    // 初期ボール・GK位置
    let ball = { x: 350, y: goalYCenter };      // だいたい中央付近
    let gk   = { x: 140, y: goalYCenter };      // ゴール前付近

    let draggingBall = false;
    let draggingGK = false;

    const stateDiv = document.getElementById("state");

    function normalize(v) {
      const len = Math.hypot(v.x, v.y);
      return { x: v.x / len, y: v.y / len };
    }

    function drawPitch() {
      ctx.clearRect(0, 0, W, H);

      // ピッチ背景
      ctx.fillStyle = "#2e7d32";
      ctx.fillRect(0, 0, W, H);

      // センターライン
      ctx.strokeStyle = "#ffffff55";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(W / 2, 0);
      ctx.lineTo(W / 2, H);
      ctx.stroke();

      // ゴール
      ctx.strokeStyle = "white";
      ctx.lineWidth = 6;
      ctx.beginPath();
      ctx.moveTo(leftPost.x, leftPost.y);
      ctx.lineTo(rightPost.x, rightPost.y);
      ctx.stroke();

      // ペナルティエリア（簡易）
      const boxDepth = 16.5 * METER;
      const boxWidth = 40.32 * METER; // 図の値に合わせる
      ctx.strokeStyle = "#ffffffaa";
      ctx.lineWidth = 2;
      ctx.strokeRect(
        goalX,
        goalYCenter - boxWidth / 2,
        boxDepth,
        boxWidth
      );

      // ゴールエリア（5.5m）
      const goalAreaDepth = 5.5 * METER;
      const goalAreaWidth = 18.32 * METER;
      ctx.strokeRect(
        goalX,
        goalYCenter - goalAreaWidth / 2,
        goalAreaDepth,
        goalAreaWidth
      );
    }

    // シュートコース（ボール→左右ポスト）
    function drawShootingCone() {
      ctx.strokeStyle = "#ffeb3b";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(ball.x, ball.y);
      ctx.lineTo(leftPost.x, leftPost.y);
      ctx.moveTo(ball.x, ball.y);
      ctx.lineTo(rightPost.x, rightPost.y);
      ctx.stroke();
    }

    // GKが本来守るべき横ライン（シュートコースの幅そのもの）
    function drawGKHorizontalCoverage() {
      const y = gk.y;
      const bp = ball;

      function intersectWithPost(post) {
        const dy = post.y - bp.y;
        const dx = post.x - bp.x;
        const t = (y - bp.y) / dy;
        const x = bp.x + dx * t;
        return { x, y };
      }

      const p1 = intersectWithPost(leftPost);
      const p2 = intersectWithPost(rightPost);

      ctx.strokeStyle = "#e0f7fa";
      ctx.lineWidth = 4;
      ctx.beginPath();
      ctx.moveTo(p1.x, p1.y);
      ctx.lineTo(p2.x, p2.y);
      ctx.stroke();
    }

    function drawBall() {
      ctx.fillStyle = "red";
      ctx.beginPath();
      ctx.arc(ball.x, ball.y, 10, 0, Math.PI * 2);
      ctx.fill();
    }

    function drawGK() {
      const distPx = Math.hypot(ball.x - gk.x, ball.y - gk.y);
      const distM = distPx / METER;

      let color = "blue";
      if (distM >= 5.0 && distM <= 5.5) {
        color = "#03a9f4"; // フロントダイブゾーン
      } else if (distM < 5.0) {
        color = "#00e5ff"; // ブロッキングゾーン
      }

      // GK本体（円で簡易表示）
      ctx.fillStyle = color;
      ctx.beginPath();
      ctx.arc(gk.x, gk.y, 14, 0, Math.PI * 2);
      ctx.fill();

      // ボールに正対しているイメージ（おへそ→ボールの線）
      ctx.strokeStyle = "#ffffffaa";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(gk.x, gk.y);
      ctx.lineTo(ball.x, ball.y);
      ctx.stroke();
    }

    function updateStateText() {
      const distPx = Math.hypot(ball.x - gk.x, ball.y - gk.y);
      const distM = distPx / METER;

      let posture = "";
      let action = "";

      if (distM > 5.5) {
        posture = "通常の構え";
        action = "ポジショニング継続";
      } else if (distM >= 5.0 && distM <= 5.5) {
        posture = "やや低い構え";
        action = "フロントダイブでボールを奪うべき距離";
      } else {
        posture = "低い構え（ブロッキング姿勢）";
        action = "ブロッキングで対応すべき距離";
      }

      stateDiv.textContent =
        `ボールとの距離: ${distM.toFixed(2)}m ｜ 姿勢: ${posture} ｜ アクション: ${action}`;
    }

    function draw() {
      drawPitch();
      drawShootingCone();
      drawGKHorizontalCoverage();
      drawBall();
      drawGK();
      updateStateText();
    }

    // ドラッグ処理
    canvas.addEventListener("mousedown", (e) => {
      const rect = canvas.getBoundingClientRect();
      const mx = e.clientX - rect.left;
      const my = e.clientY - rect.top;

      if (Math.hypot(mx - ball.x, my - ball.y) < 15) {
        draggingBall = true;
      } else if (Math.hypot(mx - gk.x, my - gk.y) < 18) {
        draggingGK = true;
      }
    });

    canvas.addEventListener("mousemove", (e) => {
      const rect = canvas.getBoundingClientRect();
      const mx = e.clientX - rect.left;
      const my = e.clientY - rect.top;

      if (draggingBall) {
        ball.x = mx;
        ball.y = my;
      } else if (draggingGK) {
        gk.x = mx;
        gk.y = my;
      }
    });

    canvas.addEventListener("mouseup", () => {
      draggingBall = false;
      draggingGK = false;
    });

    canvas.addEventListener("mouseleave", () => {
      draggingBall = false;
      draggingGK = false;
    });

    function loop() {
      draw();
      requestAnimationFrame(loop);
    }

    loop();
  </script>
</body>
</html>

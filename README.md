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
    #ui {
      margin-top: 10px;
    }
    canvas {
      background: #2e7d32;
      border: 2px solid #fff;
      margin-top: 10px;
    }
    .label {
      margin: 5px 0;
    }
    select, input[type=range] {
      margin: 5px;
    }
    #state {
      margin-top: 10px;
      font-size: 14px;
    }
  </style>
</head>
<body>
  <h1>GK Positioning Visualizer</h1>

  <div id="ui">
    <div class="label">
      前に出る距離（ボールからの距離・m）：
      <span id="frontLabel">10</span>m
      <input type="range" id="frontDist" min="5" max="18" value="10">
    </div>
  </div>

  <canvas id="pitch" width="900" height="600"></canvas>
  <div id="state"></div>

  <script>
    const canvas = document.getElementById("pitch");
    const ctx = canvas.getContext("2d");

    const W = canvas.width;
    const H = canvas.height;

    // 1m ≒ 10px としてスケール
    const METER = 10;

    // ゴールポスト（左ゴール）
    const leftPost = { x: 50, y: H / 2 - 40 };
    const rightPost = { x: 50, y: H / 2 + 40 };

    // 初期ボール・GK位置
    let ball = { x: 350, y: 280 };
    let gk = { x: 140, y: H / 2 };

    // ドラッグ判定
    let draggingBall = false;
    let draggingGK = false;

    const frontSlider = document.getElementById("frontDist");
    const frontLabel = document.getElementById("frontLabel");
    const stateDiv = document.getElementById("state");

    frontLabel.textContent = frontSlider.value;

    function normalize(v) {
      const len = Math.hypot(v.x, v.y);
      return { x: v.x / len, y: v.y / len };
    }

    // GKをシュート角度の2等分線上に置く（ボールからの距離をスライダーで調整）
    function updateGKOnBisector() {
      const bp = ball;
      const v1 = { x: leftPost.x - bp.x, y: leftPost.y - bp.y };
      const v2 = { x: rightPost.x - bp.x, y: rightPost.y - bp.y };

      const n1 = normalize(v1);
      const n2 = normalize(v2);

      const bis = normalize({ x: n1.x + n2.x, y: n1.y + n2.y });

      const dMeter = Number(frontSlider.value); // ボールからの距離[m]
      const dPx = dMeter * METER;

      gk.x = bp.x + bis.x * dPx;
      gk.y = bp.y + bis.y * dPx;
    }

    // GKが守るべき横のスペース（2等分線内に制限）
    function getGKHorizontalCoverage() {
      const y = gk.y;
      const bp = ball;

      // ボール→ポストの線分上で、y = gk.y となる点を求める
      function intersectWithPost(post) {
        const dy = post.y - bp.y;
        const dx = post.x - bp.x;
        const t = (y - bp.y) / dy;
        const x = bp.x + dx * t;
        return { x, y };
      }

      const p1 = intersectWithPost(leftPost);
      const p2 = intersectWithPost(rightPost);

      return { p1, p2 };
    }

    // 姿勢・対応の状態テキスト
    function updateStateText() {
      const distPx = Math.hypot(ball.x - gk.x, ball.y - gk.y);
      const distM = distPx / METER;

      let posture = "";
      let action = "";

      if (distM > 5.5) {
        posture = "通常の構え（距離がまだある）";
        action = "ポジショニング継続";
      } else if (distM >= 5.0 && distM <= 5.5) {
        posture = "やや低い構え";
        action = "フロントダイブでボールを奪うべきゾーン";
      } else {
        posture = "低い構え（ブロッキング姿勢）";
        action = "ブロッキングで対応すべきゾーン";
      }

      // クロス時の半身姿勢（簡易条件：サイド深い位置）
      let crossNote = "";
      if (isCrossZone(ball)) {
        crossNote = "／クロス対応：中を見られる半身の姿勢が必要";
      }

      // GKがポスト付近にいるときの半身
      let postNote = "";
      const distToPost = Math.min(
        Math.hypot(gk.x - leftPost.x, gk.y - leftPost.y),
        Math.hypot(gk.x - rightPost.x, gk.y - rightPost.y)
      );
      if (distToPost < 30) {
        postNote = "／ポスト付近：半身の姿勢で対応";
      }

      stateDiv.textContent =
        `ボールとの距離: ${distM.toFixed(2)}m ｜ 姿勢: ${posture} ｜ アクション: ${action}${crossNote}${postNote}`;
    }

    // クロス対応ゾーン（簡易モデル）
    function isCrossZone(b) {
      // ペナルティエリアの簡易定義
      const boxDepth = 16.5 * METER; // 16.5m
      const boxLeft = leftPost.x;
      const boxRight = leftPost.x + boxDepth;
      const boxTop = H / 2 - 110;
      const boxBottom = H / 2 + 110;

      const inBox =
        b.x >= boxLeft && b.x <= boxRight &&
        b.y >= boxTop && b.y <= boxBottom;

      const wideDeep =
        (b.y < boxTop || b.y > boxBottom) && b.x < boxRight;

      return !inBox && wideDeep;
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
      const boxWidth = 220;
      ctx.strokeStyle = "#ffffffaa";
      ctx.lineWidth = 2;
      ctx.strokeRect(
        leftPost.x,
        H / 2 - boxWidth / 2,
        boxDepth,
        boxWidth
      );
    }

    function drawConeAndBisector() {
      // シュートコース（ボール→ポスト）
      ctx.strokeStyle = "#ffeb3b";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(ball.x, ball.y);
      ctx.lineTo(leftPost.x, leftPost.y);
      ctx.moveTo(ball.x, ball.y);
      ctx.lineTo(rightPost.x, rightPost.y);
      ctx.stroke();

      // 2等分線
      const bp = ball;
      const v1 = { x: leftPost.x - bp.x, y: leftPost.y - bp.y };
      const v2 = { x: rightPost.x - bp.x, y: rightPost.y - bp.y };
      const n1 = normalize(v1);
      const n2 = normalize(v2);
      const bis = normalize({ x: n1.x + n2.x, y: n1.y + n2.y });

      ctx.strokeStyle = "#ff9800";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(bp.x, bp.y);
      ctx.lineTo(bp.x + bis.x * 400, bp.y + bis.y * 400);
      ctx.stroke();
    }

    function drawBall() {
      ctx.fillStyle = "red";
      ctx.beginPath();
      ctx.arc(ball.x, ball.y, 10, 0, Math.PI * 2);
      ctx.fill();
    }

    function drawGK() {
      // GK本体
      const distPx = Math.hypot(ball.x - gk.x, ball.y - gk.y);
      const distM = distPx / METER;

      let color = "blue";
      if (distM >= 5.0 && distM <= 5.5) {
        color = "#03a9f4"; // フロントダイブゾーン
      } else if (distM < 5.0) {
        color = "#00e5ff"; // ブロッキングゾーン
      }

      ctx.fillStyle = color;
      ctx.beginPath();
      ctx.arc(gk.x, gk.y, 14, 0, Math.PI * 2);
      ctx.fill();

      // ボールに正対しているイメージ（ボール方向への線）
      ctx.strokeStyle = "#ffffffaa";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(gk.x, gk.y);
      ctx.lineTo(ball.x, ball.y);
      ctx.stroke();

      // 5.5mゾーン（フロントダイブ可能ライン）
      const zoneRadius = 5.5 * METER;
      ctx.strokeStyle = "#ffffff33";
      ctx.lineWidth = 1;
      ctx.beginPath();
      ctx.arc(gk.x, gk.y, zoneRadius, 0, Math.PI * 2);
      ctx.stroke();
    }

    function drawGKCoverage() {
      const { p1, p2 } = getGKHorizontalCoverage();

      // GKが守るべき横のスペース（シュートコース内）
      ctx.strokeStyle = "#e0f7fa";
      ctx.lineWidth = 4;
      ctx.beginPath();
      ctx.moveTo(p1.x, p1.y);
      ctx.lineTo(p2.x, p2.y);
      ctx.stroke();
    }

    function draw() {
      drawPitch();
      drawConeAndBisector();
      drawGKCoverage();
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
        // ボールが動いたら、基本は2等分線上にGKを再配置（理論的ポジショニング）
        updateGKOnBisector();
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

    frontSlider.addEventListener("input", () => {
      frontLabel.textContent = frontSlider.value;
      updateGKOnBisector();
    });

    function loop() {
      draw();
      requestAnimationFrame(loop);
    }

    updateGKOnBisector();
    loop();
  </script>
</body>
</html>

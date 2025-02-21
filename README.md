# game
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>360도 미사일 피하기 게임</title>
  <style>
    body {
      margin: 0;
      background-color: #222;
      color: #fff;
      font-family: Arial, sans-serif;
      overflow: hidden;
    }
    canvas {
      background-color: black; /* 우주 배경 */
      display: block;
      margin: 20px auto;
      border: 2px solid #333;
    }
    .info {
      text-align: center;
    }
    #resetBtn {
      display: none;
      margin: 0 auto;
      padding: 10px 20px;
      font-size: 16px;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <div class="info">
    <h1>360도 미사일 피하기 게임</h1>
    <p>화살표 키로 비행기를 조종해 미사일을 피하세요!</p>
  </div>
  <!-- 정사각형 캔버스 (600x600) -->
  <canvas id="gameCanvas" width="600" height="600"></canvas>
  <div style="text-align: center;">
    <button id="resetBtn">Reset Game</button>
  </div>

  <script>
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");
    const resetBtn = document.getElementById("resetBtn");

    // 플레이어 설정 (비행기 모양)
    const player = {
      x: canvas.width / 2,
      y: canvas.height / 2,
      size: 15,  // 비행기 크기
      speed: 5,
      dx: 0,
      dy: 0
    };

    // 게임 변수
    let dodgedCount = 0; // 피한 미사일 총 갯수
    let missiles = [];
    let gameOver = false;
    let missileSpawnCount = 0; // 생성된 미사일 총 개수
    const baseSpawnRate = 0.1; // 초기 미사일 생성 확률

    // 키 이벤트 처리
    document.addEventListener("keydown", keyDownHandler);
    document.addEventListener("keyup", keyUpHandler);

    function keyDownHandler(e) {
      if (e.key === "ArrowLeft") {
        player.dx = -player.speed;
      } else if (e.key === "ArrowRight") {
        player.dx = player.speed;
      } else if (e.key === "ArrowUp") {
        player.dy = -player.speed;
      } else if (e.key === "ArrowDown") {
        player.dy = player.speed;
      }
    }

    function keyUpHandler(e) {
      if (e.key === "ArrowLeft" || e.key === "ArrowRight") {
        player.dx = 0;
      }
      if (e.key === "ArrowUp" || e.key === "ArrowDown") {
        player.dy = 0;
      }
    }

    // 미사일 클래스
    // - 50% 확률로 플레이어를 목표로 날아오고, 나머지는 캔버스 내 임의의 지점을 목표로 합니다.
    // - 매 5번째 미사일은 isBig 플래그가 true로, 반지름이 2배(10)로 생성됩니다.
    class Missile {
      constructor(isBig) {
        const centerX = canvas.width / 2;
        const centerY = canvas.height / 2;
        const spawnRadius = Math.sqrt(centerX * centerX + centerY * centerY) + 20;
        const spawnAngle = Math.random() * Math.PI * 2;
        this.x = centerX + spawnRadius * Math.cos(spawnAngle);
        this.y = centerY + spawnRadius * Math.sin(spawnAngle);
        
        let targetX, targetY;
        // 50% 확률로 플레이어 타겟, 50% 확률은 캔버스 내 임의 지점
        if (Math.random() < 0.5) {
          targetX = player.x;
          targetY = player.y;
        } else {
          targetX = Math.random() * canvas.width;
          targetY = Math.random() * canvas.height;
        }
        
        let dx = targetX - this.x;
        let dy = targetY - this.y;
        const dist = Math.sqrt(dx * dx + dy * dy);
        dx /= dist;
        dy /= dist;
        
        // 미사일 속도: 2~5 사이의 기본 속도
        const baseSpeed = 2 + Math.random() * 3;
        this.dx = dx * baseSpeed;
        this.dy = dy * baseSpeed;
        
        // 미사일 크기: 기본 반지름 5, isBig이면 10
        this.radius = isBig ? 10 : 5;
        this.entered = false;
      }
      update() {
        this.x += this.dx;
        this.y += this.dy;
      }
      draw() {
        ctx.beginPath();
        ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
        ctx.fillStyle = 'yellow';
        ctx.fill();
        ctx.closePath();
      }
    }

    // 플레이어(비행기) 그리기 함수
    function drawPlayer() {
      ctx.save();
      ctx.translate(player.x, player.y);
      ctx.beginPath();
      ctx.moveTo(0, -player.size);             // 코 (앞부분)
      ctx.lineTo(player.size * 0.6, player.size); // 오른쪽 날개
      ctx.lineTo(0, player.size * 0.5);          // 꼬리 중앙
      ctx.lineTo(-player.size * 0.6, player.size);// 왼쪽 날개
      ctx.closePath();
      ctx.fillStyle = 'blue';
      ctx.fill();
      ctx.restore();
    }

    // 미사일과 플레이어 간 충돌 검사
    function checkCollision(missile, player) {
      const dx = missile.x - player.x;
      const dy = missile.y - player.y;
      const distance = Math.sqrt(dx * dx + dy * dy);
      return distance < missile.radius + player.size * 0.6;
    }

    // 게임 초기화 함수
    function initGame() {
      dodgedCount = 0;
      missiles = [];
      gameOver = false;
      missileSpawnCount = 0;
      player.x = canvas.width / 2;
      player.y = canvas.height / 2;
      resetBtn.style.display = "none";
      updateGame();
    }

    // 게임 루프 함수
    function updateGame() {
      if (gameOver) return;
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      // 플레이어 이동 업데이트 및 경계 체크
      player.x += player.dx;
      player.y += player.dy;
      if (player.x < player.size) player.x = player.size;
      if (player.x > canvas.width - player.size) player.x = canvas.width - player.size;
      if (player.y < player.size) player.y = player.size;
      if (player.y > canvas.height - player.size) player.y = canvas.height - player.size;

      drawPlayer();

      // 미사일 업데이트 및 그리기
      for (let i = missiles.length - 1; i >= 0; i--) {
        let missile = missiles[i];
        missile.update();
        missile.draw();

        if (!missile.entered &&
            missile.x > 0 && missile.x < canvas.width &&
            missile.y > 0 && missile.y < canvas.height) {
          missile.entered = true;
        }

        if (checkCollision(missile, player)) {
          gameOver = true;
        }

        if (missile.x < -50 || missile.x > canvas.width + 50 ||
            missile.y < -50 || missile.y > canvas.height + 50) {
          missiles.splice(i, 1);
          if (missile.entered) {
            dodgedCount++;
          }
        }
      }

      // 피한 미사일 수에 따라 spawnRate를 조절 (10단위마다 1.3배 증가)
      const spawnRate = baseSpawnRate * Math.pow(1.3, Math.floor(dodgedCount / 10));
      
      // 미사일 생성 (spawnRate에 따라)
      if (Math.random() < spawnRate) {
        missileSpawnCount++;
        // 매 5번째 미사일은 isBig 플래그 true
        const isBig = (missileSpawnCount % 5 === 0);
        missiles.push(new Missile(isBig));
      }

      // 좌측 상단에 피한 미사일 총 갯수 표시
      ctx.fillStyle = 'white';
      ctx.font = '16px Arial';
      ctx.fillText('Dodged: ' + dodgedCount, 10, 20);

      if (!gameOver) {
        requestAnimationFrame(updateGame);
      } else {
        ctx.fillStyle = 'white';
        ctx.font = '30px Arial';
        ctx.fillText('Game Over', canvas.width / 2 - 70, canvas.height / 2);
        resetBtn.style.display = "block";
      }
    }

    // 리셋 버튼 클릭 시 게임 초기화
    resetBtn.addEventListener("click", function() {
      initGame();
    });

    // 게임 시작
    initGame();
  </script>
</body>
</html>

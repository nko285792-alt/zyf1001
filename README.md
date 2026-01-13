<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>警察抓小偷 - 射擊模式</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #1a202c;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            touch-action: none;
        }
        #game-container {
            position: relative;
            width: 100vw;
            height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
        }
        canvas {
            display: block;
            background: radial-gradient(circle, #2d3748 0%, #1a202c 100%);
            box-shadow: 0 0 50px rgba(0,0,0,0.5);
        }
        #ui-layer {
            position: absolute;
            top: 20px;
            left: 20px;
            right: 20px;
            pointer-events: none;
            display: flex;
            justify-content: space-between;
            color: white;
            font-size: 24px;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.5);
        }
        #overlay {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.8);
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            color: white;
            z-index: 10;
        }
        .btn {
            margin-top: 20px;
            padding: 12px 30px;
            font-size: 20px;
            background: #e53e3e;
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            pointer-events: auto;
            transition: transform 0.2s, background 0.2s;
        }
        .btn:hover {
            background: #c53030;
            transform: scale(1.05);
        }
    </style>
</head>
<body>

<div id="game-container">
    <div id="ui-layer">
        <div>分數: <span id="score">0</span></div>
        <div>生命: <span id="health">3</span></div>
    </div>

    <div id="overlay">
        <h1 id="status-title" class="text-5xl font-bold mb-4">警察抓小偷</h1>
        <p id="status-desc" class="text-xl">移動滑鼠瞄準，點擊左鍵射擊！</p>
        <button id="start-btn" class="btn">開始遊戲</button>
    </div>

    <canvas id="gameCanvas"></canvas>
</div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const scoreEl = document.getElementById('score');
    const healthEl = document.getElementById('health');
    const overlay = document.getElementById('overlay');
    const startBtn = document.getElementById('start-btn');
    const statusTitle = document.getElementById('status-title');
    const statusDesc = document.getElementById('status-desc');

    let score = 0;
    let health = 3;
    let gameActive = false;
    let animationId;

    // 遊戲物件存儲
    let player;
    let bullets = [];
    let enemies = [];
    let particles = [];
    let enemySpawnTimer = 0;

    // 滑鼠座標
    const mouse = { x: 0, y: 0 };

    // 初始化畫布大小
    function resize() {
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
    }
    window.addEventListener('resize', resize);
    resize();

    // 玩家物件 (警察)
    class Player {
        constructor() {
            this.x = canvas.width / 2;
            this.y = canvas.height / 2;
            this.radius = 25;
            this.color = '#4299e1'; // 藍色代表警察
        }

        draw() {
            // 計算槍口方向
            const angle = Math.atan2(mouse.y - this.y, mouse.x - this.x);

            ctx.save();
            ctx.translate(this.x, this.y);
            ctx.rotate(angle);

            // 畫出身體 (警察帽形狀感)
            ctx.beginPath();
            ctx.arc(0, 0, this.radius, 0, Math.PI * 2);
            ctx.fillStyle = this.color;
            ctx.fill();
            ctx.strokeStyle = 'white';
            ctx.lineWidth = 3;
            ctx.stroke();

            // 警察徽章
            ctx.fillStyle = 'gold';
            ctx.fillRect(5, -5, 10, 10);

            // 槍
            ctx.fillStyle = '#4a5568';
            ctx.fillRect(15, -5, 20, 10);

            ctx.restore();
        }

        update() {
            // 警察跟隨滑鼠移動 (平滑感)
            this.x += (mouse.x - this.x) * 0.1;
            this.y += (mouse.y - this.y) * 0.1;
            this.draw();
        }
    }

    // 子彈物件
    class Bullet {
        constructor(x, y, angle) {
            this.x = x;
            this.y = y;
            this.radius = 5;
            this.color = '#f6e05e';
            this.velocity = {
                x: Math.cos(angle) * 10,
                y: Math.sin(angle) * 10
            };
        }

        draw() {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
            ctx.fillStyle = this.color;
            ctx.fill();
        }

        update() {
            this.x += this.velocity.x;
            this.y += this.velocity.y;
            this.draw();
        }
    }

    // 敵人物件 (小偷)
    class Enemy {
        constructor() {
            this.radius = 20 + Math.random() * 10;
            this.color = '#718096'; // 灰色衣服

            // 從邊界隨機生成
            if (Math.random() < 0.5) {
                this.x = Math.random() < 0.5 ? -this.radius : canvas.width + this.radius;
                this.y = Math.random() * canvas.height;
            } else {
                this.x = Math.random() * canvas.width;
                this.y = Math.random() < 0.5 ? -this.radius : canvas.height + this.radius;
            }

            // 朝向玩家移動
            const angle = Math.atan2(player.y - this.y, player.x - this.x);
            const speed = 1.5 + (score / 100); // 難度隨分數提升
            this.velocity = {
                x: Math.cos(angle) * speed,
                y: Math.sin(angle) * speed
            };
        }

        draw() {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
            ctx.fillStyle = this.color;
            ctx.fill();
            
            // 蒙面效果 (黑條)
            ctx.fillStyle = 'black';
            ctx.fillRect(this.x - this.radius, this.y - 5, this.radius * 2, 10);

            // 錢袋
            ctx.fillStyle = '#48bb78';
            ctx.beginPath();
            ctx.arc(this.x + 10, this.y + 10, 8, 0, Math.PI * 2);
            ctx.fill();
        }

        update() {
            this.x += this.velocity.x;
            this.y += this.velocity.y;
            this.draw();
        }
    }

    // 粒子效果 (爆炸)
    class Particle {
        constructor(x, y, color) {
            this.x = x;
            this.y = y;
            this.color = color;
            this.radius = Math.random() * 3;
            this.velocity = {
                x: (Math.random() - 0.5) * 8,
                y: (Math.random() - 0.5) * 8
            };
            this.alpha = 1;
        }

        draw() {
            ctx.save();
            ctx.globalAlpha = this.alpha;
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
            ctx.fillStyle = this.color;
            ctx.fill();
            ctx.restore();
        }

        update() {
            this.velocity.x *= 0.98;
            this.velocity.y *= 0.98;
            this.x += this.velocity.x;
            this.y += this.velocity.y;
            this.alpha -= 0.02;
            this.draw();
        }
    }

    function init() {
        score = 0;
        health = 3;
        scoreEl.innerText = score;
        healthEl.innerText = health;
        bullets = [];
        enemies = [];
        particles = [];
        player = new Player();
    }

    function spawnEnemy() {
        enemySpawnTimer++;
        const spawnRate = Math.max(20, 60 - Math.floor(score / 10)); // 分數越高出怪越快
        if (enemySpawnTimer > spawnRate) {
            enemies.push(new Enemy());
            enemySpawnTimer = 0;
        }
    }

    function animate() {
        animationId = requestAnimationFrame(animate);
        ctx.clearRect(0, 0, canvas.width, canvas.height);

        player.update();

        // 處理粒子
        particles.forEach((particle, index) => {
            if (particle.alpha <= 0) {
                particles.splice(index, 1);
            } else {
                particle.update();
            }
        });

        // 處理子彈
        bullets.forEach((bullet, bIndex) => {
            bullet.update();

            // 移除出鏡子彈
            if (bullet.x < 0 || bullet.x > canvas.width || bullet.y < 0 || bullet.y > canvas.height) {
                bullets.splice(bIndex, 1);
            }
        });

        // 處理敵人
        enemies.forEach((enemy, eIndex) => {
            enemy.update();

            // 碰撞檢測：敵人 vs 玩家
            const distToPlayer = Math.hypot(player.x - enemy.x, player.y - enemy.y);
            if (distToPlayer < player.radius + enemy.radius) {
                enemies.splice(eIndex, 1);
                health--;
                healthEl.innerText = health;
                createExplosion(enemy.x, enemy.y, '#f56565');

                if (health <= 0) {
                    gameOver();
                }
            }

            // 碰撞檢測：敵人 vs 子彈
            bullets.forEach((bullet, bIndex) => {
                const distToBullet = Math.hypot(bullet.x - enemy.x, bullet.y - enemy.y);
                if (distToBullet < enemy.radius + bullet.radius) {
                    // 擊中效果
                    createExplosion(enemy.x, enemy.y, enemy.color);
                    
                    setTimeout(() => {
                        enemies.splice(eIndex, 1);
                        bullets.splice(bIndex, 1);
                        score += 10;
                        scoreEl.innerText = score;
                    }, 0);
                }
            });
        });

        spawnEnemy();
    }

    function createExplosion(x, y, color) {
        for (let i = 0; i < 15; i++) {
            particles.push(new Particle(x, y, color));
        }
    }

    function gameOver() {
        gameActive = false;
        cancelAnimationFrame(animationId);
        overlay.style.display = 'flex';
        statusTitle.innerText = "任務失敗！";
        statusDesc.innerText = `小偷逃跑了。你的最終得分：${score}`;
        startBtn.innerText = "重新部署";
    }

    // 事件監聽
    window.addEventListener('mousemove', (e) => {
        mouse.x = e.clientX;
        mouse.y = e.clientY;
    });

    window.addEventListener('mousedown', () => {
        if (!gameActive) return;
        const angle = Math.atan2(mouse.y - player.y, mouse.x - player.x);
        bullets.push(new Bullet(player.x, player.y, angle));
    });

    startBtn.addEventListener('click', () => {
        overlay.style.display = 'none';
        gameActive = true;
        init();
        animate();
    });

</script>
</body>
</html>

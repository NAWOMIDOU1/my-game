# my-game

<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>陰陽少女の妖怪退治</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #1a1a24;
            color: #fff;
            font-family: 'Hiragino Kaku Gothic ProN', sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            overflow: hidden;
        }
        #game-container {
            position: relative;
            box-shadow: 0 10px 30px rgba(0,0,0,0.5);
            border: 3px solid #ff7979;
            border-radius: 8px;
            overflow: hidden;
        }
        canvas {
            display: block;
            background: #2f3542;
            image-rendering: pixelated; /* ドット絵をくっきり表示 */
        }
        #ui {
            position: absolute;
            top: 10px;
            left: 10px;
            font-size: 16px;
            font-weight: bold;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.8);
            pointer-events: none;
        }
        #controls-info {
            margin-top: 10px;
            font-size: 14px;
            color: #ccc;
            text-align: center;
        }
    </style>
</head>
<body>

<div id="game-container">
    <canvas id="canvas" width="640" height="480"></canvas>
    <div id="ui">
        <div>撃破数: <span id="scoreText">0</span></div>
        <div>HP: <span id="hpText" style="color: #ff6b81;">❤️❤️❤️</span></div>
    </div>
</div>

<div id="controls-info">
    移動: <b>[W][A][S][D]</b> または <b>[矢印キー]</b> ｜ 攻撃: <b>自動連射</b>（向いている方向）
</div>

<script>
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

// 画像の読み込み
const playerImg = new Image();
playerImg.src = 'player.png';

// ----------------------------------------------------
// 【向きの修正マップ】
// 画像の左右が逆転しているため、正しい向きにマッピング
// (col: 列 0=左, 1=中央, 2=右 / row: 行 0=上, 1=中, 2=下)
// ----------------------------------------------------
const spriteMap = {
    'up':           { col: 1, row: 0 },
    'down':         { col: 1, row: 2 },
    'idle':         { col: 1, row: 1 }, // 正面
    // 左右反転を修正：左向きの時は右列(col:2)を、右向きの時は左列(col:0)を使用
    'left':         { col: 2, row: 1 }, 
    'right':        { col: 0, row: 1 }, 
    'up-left':      { col: 2, row: 0 }, 
    'up-right':     { col: 0, row: 0 }, 
    'down-left':    { col: 2, row: 2 }, 
    'down-right':   { col: 0, row: 2 }  
};

// プレイヤー情報
const player = {
    x: canvas.width / 2,
    y: canvas.height / 2,
    speed: 3.5,
    size: 40,
    dir: 'idle',
    aimX: 0,
    aimY: 1, // 初期向き: 下
    hp: 3,
    invincibleTimer: 0
};

let bullets = [];
let enemies = [];
let particles = [];
let score = 0;
let gameOver = false;
let shootCooldown = 0;

// キー入力管理
const keys = {};
window.addEventListener('keydown', e => {
    keys[e.key.toLowerCase()] = true;
    if (gameOver && (e.key === 'r' || e.key === 'R' || e.key === 'Enter')) resetGame();
});
window.addEventListener('keyup', e => {
    keys[e.key.toLowerCase()] = false;
});

function resetGame() {
    player.x = canvas.width / 2;
    player.y = canvas.height / 2;
    player.hp = 3;
    player.dir = 'idle';
    player.aimX = 0;
    player.aimY = 1;
    bullets = [];
    enemies = [];
    particles = [];
    score = 0;
    gameOver = false;
    updateUI();
}

function updateUI() {
    document.getElementById('scoreText').textContent = score;
    let hearts = '';
    for(let i=0; i<player.hp; i++) hearts += '❤️';
    document.getElementById('hpText').textContent = hearts || '💀 撃沈';
}

// ゲームループ
function update() {
    if (gameOver) return;

    // 1. 移動判定 & 向きの判定
    let dx = 0;
    let dy = 0;
    if (keys['arrowup'] || keys['w']) dy -= 1;
    if (keys['arrowdown'] || keys['s']) dy += 1;
    if (keys['arrowleft'] || keys['a']) dx -= 1;
    if (keys['arrowright'] || keys['d']) dx += 1;

    // 斜め移動の速度正規化
    if (dx !== 0 && dy !== 0) {
        dx *= 0.7071;
        dy *= 0.7071;
    }

    player.x += dx * player.speed;
    player.y += dy * player.speed;

    // 画面外に出ないよう制限
    player.x = Math.max(25, Math.min(canvas.width - 25, player.x));
    player.y = Math.max(30, Math.min(canvas.height - 30, player.y));

    // 移動方向からスプライトの向きを設定
    if (dx !== 0 || dy !== 0) {
        player.aimX = dx;
        player.aimY = dy;

        if (dy < 0 && dx < 0) player.dir = 'up-left';
        else if (dy < 0 && dx > 0) player.dir = 'up-right';
        else if (dy > 0 && dx < 0) player.dir = 'down-left';
        else if (dy > 0 && dx > 0) player.dir = 'down-right';
        else if (dy < 0) player.dir = 'up';
        else if (dy > 0) player.dir = 'down';
        else if (dx < 0) player.dir = 'left';
        else if (dx > 0) player.dir = 'right';
    } else {
        player.dir = 'idle';
    }

    // 2. 攻撃（向いている方向にお札弾を発射）
    shootCooldown++;
    if (shootCooldown >= 14) {
        shootCooldown = 0;
        let speed = 7;
        let len = Math.hypot(player.aimX, player.aimY) || 1;
        bullets.push({
            x: player.x,
            y: player.y,
            vx: (player.aimX / len) * speed,
            vy: (player.aimY / len) * speed,
            life: 60
        });
    }

    // 弾の更新
    bullets.forEach((b, index) => {
        b.x += b.vx;
        b.y += b.vy;
        b.life--;
        if (b.life <= 0) bullets.splice(index, 1);
    });

    // 3. 敵の出現
    if (Math.random() < 0.035 + score * 0.0005) {
        const edge = Math.floor(Math.random() * 4);
        let ex, ey;
        if (edge === 0) { ex = Math.random() * canvas.width; ey = -20; }
        else if (edge === 1) { ex = canvas.width + 20; ey = Math.random() * canvas.height; }
        else if (edge === 2) { ex = Math.random() * canvas.width; ey = canvas.height + 20; }
        else { ex = -20; ey = Math.random() * canvas.height; }

        enemies.push({
            x: ex,
            y: ey,
            speed: 1.2 + Math.random() * 0.8,
            size: 16,
            color: Math.random() > 0.5 ? '#70a1ff' : '#a29bfe' // 鬼火の色
        });
    }

    // 敵の更新
    enemies.forEach((en, eIdx) => {
        const angle = Math.atan2(player.y - en.y, player.x - en.x);
        en.x += Math.cos(angle) * en.speed;
        en.y += Math.sin(angle) * en.speed;

        // 弾と敵の当たり判定
        bullets.forEach((b, bIdx) => {
            if (Math.hypot(en.x - b.x, en.y - b.y) < en.size + 8) {
                // 撃破エフェクト
                for (let i = 0; i < 8; i++) {
                    particles.push({
                        x: en.x, y: en.y,
                        vx: (Math.random() - 0.5) * 5,
                        vy: (Math.random() - 0.5) * 5,
                        life: 20, color: en.color
                    });
                }
                enemies.splice(eIdx, 1);
                bullets.splice(bIdx, 1);
                score++;
                updateUI();
            }
        });

        // プレイヤーと敵の当たり判定
        if (player.invincibleTimer <= 0 && Math.hypot(en.x - player.x, en.y - player.y) < 24) {
            player.hp--;
            player.invincibleTimer = 60; // 1秒無敵
            updateUI();
            if (player.hp <= 0) {
                gameOver = true;
            }
        }
    });

    if (player.invincibleTimer > 0) player.invincibleTimer--;

    // パーティクル更新
    particles.forEach((p, idx) => {
        p.x += p.vx;
        p.y += p.vy;
        p.life--;
        if (p.life <= 0) particles.splice(idx, 1);
    });
}

// 描画処理
function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // 床の模様
    ctx.strokeStyle = '#353b48';
    ctx.lineWidth = 1;
    for (let x = 0; x < canvas.width; x += 40) {
        ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, canvas.height); ctx.stroke();
    }
    for (let y = 0; y < canvas.height; y += 40) {
        ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(canvas.width, y); ctx.stroke();
    }

    // 弾の描画（お札風の弾）
    ctx.fillStyle = '#fffa65';
    ctx.shadowColor = '#ffaf40';
    ctx.shadowBlur = 8;
    bullets.forEach(b => {
        ctx.beginPath();
        ctx.arc(b.x, b.y, 6, 0, Math.PI * 2);
        ctx.fill();
    });
    ctx.shadowBlur = 0; // リセット

    // 敵の描画（鬼火・妖怪魂）
    enemies.forEach(en => {
        ctx.fillStyle = en.color;
        ctx.beginPath();
        ctx.arc(en.x, en.y, en.size, 0, Math.PI * 2);
        ctx.fill();
        // 中心光
        ctx.fillStyle = '#fff';
        ctx.beginPath();
        ctx.arc(en.x, en.y, en.size * 0.4, 0, Math.PI * 2);
        ctx.fill();
    });

    // パーティクルの描画
    particles.forEach(p => {
        ctx.fillStyle = p.color;
        ctx.fillRect(p.x, p.y, 3, 3);
    });

    // プレイヤーの描画
    if (player.invincibleTimer % 6 < 3) { // 無敵時の点滅
        const pos = spriteMap[player.dir] || spriteMap['idle'];

        if (playerImg.complete && playerImg.naturalWidth > 0) {
            // スプライトシートから1キャラ分を切り出す計算
            const cellW = playerImg.naturalWidth / 3;
            const cellH = playerImg.naturalHeight / 3;

            // 文字（TOP-LEFTなど）を避けてキャラだけを切り抜くためのトリミング調整
            const cropX = pos.col * cellW + cellW * 0.15;
            const cropY = pos.row * cellH + cellH * 0.22; // 上の文字部分をスキップ
            const cropW = cellW * 0.7;
            const cropH = cellH * 0.75;

            const drawW = 56; // 画面上での描画サイズ
            const drawH = 60;

            ctx.drawImage(
                playerImg,
                cropX, cropY, cropW, cropH,
                player.x - drawW / 2, player.y - drawH / 2, drawW, drawH
            );
        } else {
            // 画像未読み込み時のフォールバック
            ctx.fillStyle = '#ff7979';
            ctx.beginPath();
            ctx.arc(player.x, player.y, 18, 0, Math.PI * 2);
            ctx.fill();
        }
    }

    // ゲームオーバー画面
    if (gameOver) {
        ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
        ctx.fillRect(0, 0, canvas.width, canvas.height);

        ctx.fillStyle = '#ff4757';
        ctx.font = 'bold 36px sans-serif';
        ctx.textAlign = 'center';
        ctx.fillText('GAME OVER', canvas.width / 2, canvas.height / 2 - 20);

        ctx.fillStyle = '#fff';
        ctx.font = '18px sans-serif';
        ctx.fillText(`倒した妖怪: ${score} 匹`, canvas.width / 2, canvas.height / 2 + 20);
        ctx.fillText('[ Enter ] キー または [ R ] キーでリトライ', canvas.width / 2, canvas.height / 2 + 60);
    }
}

function gameLoop() {
    update();
    draw();
    requestAnimationFrame(gameLoop);
}

// 起動
updateUI();
requestAnimationFrame(gameLoop);
</script>

</body>
</html>

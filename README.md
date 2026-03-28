<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>Famista Retro+ MVP</title>
<style>
body { background: #333; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; }
canvas { background: #228B22; border: 4px solid #fff; image-rendering: pixelated; }
.instructions { position: absolute; top: 10px; color: white; text-align: center; font-family: monospace;}
</style>
</head>
<body>
<div class="instructions">←→移動 / スペースでスイング<br>3アウトでゲーム終了！</div>
<canvas id="gameCanvas" width="400" height="500"></canvas>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// --- 状態 ---
let batterX = 200;
const batterY = 420;
let isSwinging = 0;

let ballActive = false;
let ballX = 200, ballY = 120, ballSize = 4;

let ballVX = 0, ballVY = 0;
let hit = false;

let msg = "";
let msgTimer = 0;
let pitchTimer = 60;
let score = 0;
let outs = 0;

let keys = {};

// --- 演出 ---
let shakeIntensity = 0;
let pitcherReaction = 0;
let slowMo = 0;

// --- 定数 ---
const HOMERUN_LINE = 80;

window.addEventListener('keydown', e => {
    keys[e.code] = true;
    if (e.code === 'Space' && isSwinging === 0) isSwinging = 10;
});
window.addEventListener('keyup', e => keys[e.code] = false);

// --- 音 ---
const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
function playHitSound() {
    const osc = audioCtx.createOscillator();
    osc.frequency.value = 600;
    osc.connect(audioCtx.destination);
    osc.start();
    osc.stop(audioCtx.currentTime + 0.1);
}

// --- 描画 ---
function drawPixelMan(x, y, color, isBatter) {
    ctx.fillStyle = "#FFD7AF";
    ctx.fillRect(x - 8, y - 40, 16, 16);

    if (!isBatter && pitcherReaction > 0) {
        ctx.fillStyle = "white";
    } else {
        ctx.fillStyle = color;
    }
    ctx.fillRect(x - 12, y - 24, 24, 24);

    if (isBatter) {
        ctx.fillStyle = "#964B00";
        if (isSwinging > 0) {
            ctx.fillRect(x + 10, y - 15, 35, 8);
        } else {
            ctx.fillRect(x + 10, y - 45, 8, 35);
        }
    }
}

let pitchType = 0;

// --- 更新 ---
function update() {

    if (slowMo > 0) {
        slowMo--;
        return;
    }

    if (keys['ArrowLeft'] && batterX > 100) batterX -= 4;
    if (keys['ArrowRight'] && batterX < 300) batterX += 4;

    if (shakeIntensity > 0) shakeIntensity *= 0.9;
    if (pitcherReaction > 0) pitcherReaction--;

    if (!ballActive && !hit) {
        pitchTimer--;
        if (pitchTimer <= 0) {
            ballActive = true;
            ballY = 120;
            ballX = 200 + (Math.random() * 40 - 20);
            ballSize = 4;
            pitchType = Math.random();
        }
    }

    if (ballActive) {
        if (pitchType < 0.33) {
            ballY += 6;
        } else if (pitchType < 0.66) {
            ballY += 4;
            ballX += Math.sin(ballY * 0.05) * 2;
        } else {
            ballY += 8;
        }

        if (ballY > 300) ballSize = 8;

        if (ballY > 500) {
            ballActive = false;
            pitchTimer = 60;
            msg = "STRIKE!";
            msgTimer = 40;
            outs++;
        }
    }

    if (hit) {
        ballX += ballVX;
        ballY += ballVY;
        ballVY += 0.3;

        // ホームラン判定
        if (ballY < HOMERUN_LINE) {
            msg = "HOMERUN!!";
            score += 100;
            hit = false;
            shakeIntensity = 20;
            pitcherReaction = 100;
            slowMo = 20;
            playHitSound();
        }

        if (ballY > 500) {
            hit = false;
            pitchTimer = 80;
        }
    }

    if (isSwinging > 0) {
        isSwinging--;

        if (ballActive && ballY > 380 && ballY < 440) {

            let contact = Math.abs(ballX - (batterX + 25));

            if (contact < 15) {
                ballVX = (ballX - batterX) * 0.25;
                ballVY = -12;
                hit = true;
                shakeIntensity = 15;
                playHitSound();
            } else if (contact < 25) {
                msg = "HIT!";
                score += 10;
                ballVX = (ballX - batterX) * 0.15;
                ballVY = -8;
                hit = true;
                shakeIntensity = 5;
                playHitSound();
            } else if (contact < 35) {
                msg = "FOUL";
                ballVX = (Math.random() > 0.5 ? 1 : -1) * 6;
                ballVY = -4;
                hit = true;
            } else {
                msg = "STRIKE!";
                outs++;
            }

            msgTimer = 50;
            ballActive = false;
        }
    }

    if (outs >= 3) {
        msg = "GAME OVER";
        score = 0;
        outs = 0;
    }

    if (msgTimer > 0) msgTimer--;
}

// --- 描画 ---
function draw() {
    ctx.save();

    if (shakeIntensity > 0) {
        let dx = (Math.random() - 0.5) * shakeIntensity;
        let dy = (Math.random() - 0.5) * shakeIntensity;
        ctx.translate(dx, dy);
    }

    // 背景
    ctx.fillStyle = "#444";
    ctx.fillRect(0, 0, 400, HOMERUN_LINE);

    ctx.fillStyle = "#228B22";
    ctx.fillRect(0, HOMERUN_LINE, 400, 500);

    ctx.fillStyle = "#D2B48C";
    ctx.fillRect(80, 400, 240, 70);
    ctx.fillRect(170, 110, 60, 30);

    ctx.fillStyle = "#FFF";
    ctx.fillRect(190, 440, 20, 10);

    drawPixelMan(200, 120, "#0000FF", false);

    if (pitcherReaction > 0) {
        ctx.fillStyle = "white";
        ctx.fillRect(195, 70, 10, 25);
        ctx.fillStyle = "red";
        ctx.fillRect(195, 100, 10, 10);
    }

    drawPixelMan(batterX, batterY, "#FF0000", true);

    if (ballActive || hit) {

        if (hit) {
            let shadowSize = Math.max(2, 12 * (ballY / 500));
            ctx.beginPath();
            ctx.ellipse(ballX, 460, shadowSize * 1.5, shadowSize * 0.5, 0, 0, Math.PI * 2);
            ctx.fillStyle = "rgba(0,0,0,0.4)";
            ctx.fill();
        }

        ctx.beginPath();
        ctx.arc(ballX, ballY, ballSize, 0, Math.PI * 2);
        ctx.fillStyle = "white";
        ctx.fill();
        ctx.stroke();
    }

    // UI
    ctx.fillStyle = "white";
    ctx.font = "16px monospace";
    ctx.fillText("SCORE: " + score, 260, 30);
    ctx.fillText("OUT: " + outs, 20, 30);

    if (msgTimer > 0) {
        ctx.font = "bold 30px monospace";
        ctx.textAlign = "center";
        ctx.fillText(msg, 200, 250);
    }

    ctx.restore();

    requestAnimationFrame(() => {
        update();
        draw();
    });
}

draw();
</script>
</body>
</html>
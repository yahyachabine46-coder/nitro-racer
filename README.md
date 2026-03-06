<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: The Finish Line</title>
    <style>
        body { background: #000; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; font-family: sans-serif; overflow: hidden; }
        #container { position: relative; width: 360px; height: 600px; border-radius: 30px; border: 4px solid #1a1a1a; overflow: hidden; box-shadow: 0 0 50px rgba(0,0,0,1); }
        canvas { display: block; }
        #ui { position: absolute; top: 0; left: 0; width: 100%; height: 100%; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.8); z-index: 10; transition: 0.5s; }
        h1 { color: #00f2ff; text-shadow: 0 0 20px #00f2ff; font-size: 28px; text-align: center; }
        button { background: #ff00ff; color: white; border: none; padding: 15px 40px; border-radius: 50px; font-weight: bold; font-size: 18px; cursor: pointer; box-shadow: 0 0 20px #ff00ff; }
        .stats { position: absolute; top: 20px; left: 20px; color: #00f2ff; font-weight: bold; z-index: 5; }
        #progress { position: absolute; bottom: 20px; left: 30px; width: 300px; height: 5px; background: #222; border-radius: 5px; overflow: hidden; }
        #progressBar { width: 0%; height: 100%; background: #00f2ff; box-shadow: 0 0 10px #00f2ff; }
    </style>
</head>
<body>

<div id="container">
    <div class="stats">
        <div id="sBox">SCORE: 0 / 10</div>
        <div id="vBox" style="font-size: 12px; color: #ff00ff;">VELOCITY: 0</div>
    </div>
    <div id="progress"><div id="progressBar"></div></div>
    <canvas id="gameCanvas" width="360" height="600"></canvas>
    <div id="ui">
        <h1 id="title">FUNNY CAR DRAGSTER</h1>
        <button id="startBtn" onclick="play()">START RACE</button>
    </div>
</div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const ui = document.getElementById('ui');
    const sBox = document.getElementById('sBox');
    const vBox = document.getElementById('vBox');
    const pBar = document.getElementById('progressBar');

    const LANES = [75, 180, 285];
    const GOAL_SCORE = 10; // Change this to make the race longer
    
    let player = { targetLane: 1, x: 180, velocity: 0, acceleration: 0.05, parachutesDeployed: false };
    let enemies = [];
    let score = 0;
    let active = false;
    let time = 0;
    let spawnTimer = 0;
    let finishLineY = -200; // Starts off-screen
    let loop;

    function drawCar(x, y, color, isPlayer) {
        ctx.save();
        ctx.shadowBlur = 10; ctx.shadowColor = color; ctx.fillStyle = color;
        ctx.beginPath();
        ctx.roundRect(x - 22, y - 10, 44, 100, 8);
        ctx.fill();

        if (isPlayer && player.parachutesDeployed) {
            ctx.fillStyle = "#fff";
            ctx.beginPath(); ctx.arc(x - 20, y + 105, 15, 0, Math.PI * 2);
            ctx.arc(x + 20, y + 105, 15, 0, Math.PI * 2); ctx.fill();
            ctx.strokeStyle = "#fff"; ctx.beginPath();
            ctx.moveTo(x, y + 90); ctx.lineTo(x - 20, y + 105);
            ctx.moveTo(x, y + 90); ctx.lineTo(x + 20, y + 105); ctx.stroke();
        }
        ctx.fillStyle = "rgba(0,0,0,0.8)";
        ctx.fillRect(x - 16, y + 15, 32, 20);
        ctx.restore();
    }

    function drawFinishLine(y) {
        const size = 30;
        ctx.save();
        for (let x = 30; x < 330; x += size) {
            for (let i = 0; i < 2; i++) {
                ctx.fillStyle = ( (x/size + i) % 2 === 0) ? "#fff" : "#000";
                ctx.fillRect(x, y + (i * size), size, size);
            }
        }
        ctx.restore();
    }

    function engine() {
        if (!active) return;
        time += 0.003;
        const nightVal = (Math.sin(time) + 1) / 2;

        // Background & Track
        ctx.fillStyle = `rgb(${10 * nightVal}, ${20 * nightVal}, ${50 * nightVal})`;
        ctx.fillRect(0, 0, 360, 600);
        ctx.fillStyle = "#0a0a0a"; 
        ctx.fillRect(30, 0, 300, 600);

        // Physics
        if (!player.parachutesDeployed) {
            player.velocity += player.acceleration;
            if (player.velocity > 14) player.velocity = 14; 
        } else {
            player.velocity *= 0.95; 
        }

        player.x += (LANES[player.targetLane] - player.x) * 0.15;
        vBox.innerText = `VELOCITY: ${Math.round(player.velocity * 20)} MPH`;
        pBar.style.width = (score / GOAL_SCORE * 100) + "%";

        // Spawn Enemies ONLY if score < GOAL_SCORE
        if (score < GOAL_SCORE) {
            spawnTimer++;
            if (spawnTimer > 90) {
                enemies.push({ x: LANES[Math.floor(Math.random() * 3)], y: -150 });
                spawnTimer = 0;
            }
        } else {
            // Spawn Finish Line
            finishLineY += player.velocity;
            drawFinishLine(finishLineY);
            
            if (finishLineY > 480) {
                winGame();
            }
        }

        // Update Enemies
        for (let i = enemies.length - 1; i >= 0; i--) {
            let e = enemies[i];
            e.y += (player.velocity + 2); 
            drawCar(e.x, e.y, "#ff3300", false);

            if (Math.abs(e.x - player.x) < 38 && e.y + 80 > 480 && e.y < 560) gameOver();
            if (e.y > 700) {
                enemies.splice(i, 1);
                score++;
                sBox.innerText = `SCORE: ${score} / ${GOAL_SCORE}`;
            }
        }

        drawCar(player.x, 480, "#00f2ff", true);
        loop = requestAnimationFrame(engine);
    }

    function play() {
        cancelAnimationFrame(loop);
        active = true; score = 0; player.velocity = 0; spawnTimer = 0;
        player.parachutesDeployed = false; enemies = []; finishLineY = -200;
        sBox.innerText = `SCORE: 0 / ${GOAL_SCORE}`;
        ui.style.opacity = "0";
        setTimeout(() => ui.style.display = 'none', 500);
        engine();
    }

    function gameOver() {
        active = false;
        ui.style.display = 'flex'; ui.style.opacity = "1";
        document.getElementById('title').innerHTML = "CRASHED!<br><span style='font-size:14px;color:#888'>MISSION FAILED</span>";
    }

    function winGame() {
        active = false;
        ui.style.display = 'flex'; ui.style.opacity = "1";
        document.getElementById('title').innerHTML = "VICTORY!<br><span style='font-size:14px;color:#00ffaa'>RACE COMPLETE</span>";
        document.getElementById('startBtn').innerText = "RACE AGAIN";
    }

    window.addEventListener('keydown', e => {
        if (e.key === "ArrowLeft" && player.targetLane > 0) player.targetLane--;
        if (e.key === "ArrowRight" && player.targetLane < 2) player.targetLane++;
        if (e.key === " ") player.parachutesDeployed = true;
    });
</script>
</body>
</html>

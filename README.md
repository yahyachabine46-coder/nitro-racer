<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: Funny Car Dragster</title>
    <style>
        body { background: #000; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; font-family: sans-serif; overflow: hidden; }
        #container { position: relative; width: 360px; height: 600px; border-radius: 30px; border: 4px solid #1a1a1a; overflow: hidden; box-shadow: 0 0 50px rgba(0,0,0,1); }
        canvas { display: block; }
        #ui { position: absolute; top: 0; left: 0; width: 100%; height: 100%; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.8); z-index: 10; transition: 0.5s; }
        h1 { color: #00f2ff; text-shadow: 0 0 20px #00f2ff; font-size: 28px; }
        button { background: #ff00ff; color: white; border: none; padding: 15px 40px; border-radius: 50px; font-weight: bold; font-size: 18px; cursor: pointer; box-shadow: 0 0 20px #ff00ff; }
        .stats { position: absolute; top: 20px; left: 20px; color: #00f2ff; font-weight: bold; z-index: 5; line-height: 1.5; }
    </style>
</head>
<body>

<div id="container">
    <div class="stats">
        <div id="sBox">SCORE: 0</div>
        <div id="vBox" style="font-size: 12px; color: #ff00ff;">VELOCITY: 0</div>
    </div>
    <canvas id="gameCanvas" width="360" height="600"></canvas>
    <div id="ui">
        <h1 id="title">FUNNY CAR DRAGSTER</h1>
        <button onclick="play()">START RACE</button>
        <p style="color: #444; font-size: 10px; margin-top: 10px;">SPACE TO DEPLOY CHUTES</p>
    </div>
</div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const ui = document.getElementById('ui');
    const sBox = document.getElementById('sBox');
    const vBox = document.getElementById('vBox');

    // --- FunnyCar Logic (Translated from Java) ---
    const LANES = [75, 180, 285];
    let player = { 
        targetLane: 1, 
        x: 180, 
        velocity: 0, 
        acceleration: 0.05, // High dragster acceleration
        parachutesDeployed: false 
    };

    let enemies = [];
    let score = 0;
    let active = false;
    let time = 0;
    let loop;

    function drawCar(x, y, color, isPlayer) {
        const isNight = Math.sin(time) < -0.2;
        ctx.save();
        
        // Underglow
        ctx.shadowBlur = isNight ? 20 : 5;
        ctx.shadowColor = color;
        ctx.fillStyle = color;
        
        // Dragster Body (Longer than standard cars)
        ctx.beginPath();
        ctx.roundRect(x - 22, y - 10, 44, 100, 8);
        ctx.fill();

        // Parachutes (Visual Logic)
        if (isPlayer && player.parachutesDeployed) {
            ctx.fillStyle = "#fff";
            ctx.beginPath();
            ctx.arc(x - 20, y + 105, 15, 0, Math.PI * 2);
            ctx.arc(x + 20, y + 105, 15, 0, Math.PI * 2);
            ctx.fill();
            // Strings
            ctx.strokeStyle = "#fff";
            ctx.beginPath();
            ctx.moveTo(x, y + 90); ctx.lineTo(x - 20, y + 105);
            ctx.moveTo(x, y + 90); ctx.lineTo(x + 20, y + 105);
            ctx.stroke();
        }

        // Windshield
        ctx.shadowBlur = 0;
        ctx.fillStyle = "rgba(0,0,0,0.8)";
        ctx.fillRect(x - 16, y + 10, 32, 20);
        
        ctx.restore();
    }

    function engine() {
        if (!active) return;
        time += 0.005;

        // 1. Day/Night Environment
        const nightVal = (Math.sin(time) + 1) / 2;
        ctx.fillStyle = `rgb(${5 * nightVal}, ${15 * nightVal}, ${40 * nightVal})`;
        ctx.fillRect(0, 0, 360, 600);
        ctx.fillStyle = "#111"; // Track
        ctx.fillRect(30, 0, 300, 600);

        // 2. FUNNY CAR PHYSICS (Your Java Logic)
        if (!player.parachutesDeployed) {
            player.velocity += player.acceleration;
            if (player.velocity > 15) player.velocity = 15; // Cap speed
        } else {
            player.velocity *= 0.96; // Drag simulation
        }

        // Smooth lane movement
        const destX = LANES[player.targetLane];
        player.x += (destX - player.x) * 0.15;

        // Display Velocity
        vBox.innerText = `VELOCITY: ${Math.round(player.velocity * 20)} MPH`;

        // 3. ENEMIES (Move relative to player speed)
        if (Math.random() < 0.03) {
            enemies.push({ x: LANES[Math.floor(Math.random() * 3)], y: -100 });
        }

        for (let i = enemies.length - 1; i >= 0; i--) {
            let e = enemies[i];
            // If player is fast, enemies come at them faster
            e.y += (player.velocity + 3); 
            drawCar(e.x, e.y, "#ff00ff", false);

            if (Math.abs(e.x - player.x) < 40 && e.y + 80 > 480 && e.y < 560) {
                gameOver();
            }

            if (e.y > 700) {
                enemies.splice(i, 1);
                score++;
                sBox.innerText = "SCORE: " + score;
            }
        }

        drawCar(player.x, 480, "#00f2ff", true);
        loop = requestAnimationFrame(engine);
    }

    function play() {
        cancelAnimationFrame(loop);
        active = true;
        score = 0; 
        player.velocity = 0;
        player.parachutesDeployed = false;
        enemies = [];
        ui.style.opacity = "0";
        setTimeout(() => ui.style.display = 'none', 500);
        engine();
    }

    function gameOver() {
        active = false;
        ui.style.display = 'flex';
        ui.style.opacity = "1";
        document.getElementById('title').innerText = "CRASHED";
    }

    // --- INPUTS ---
    window.addEventListener('keydown', e => {
        if (e.key === "ArrowLeft" && player.targetLane > 0) player.targetLane--;
        if (e.key === "ArrowRight" && player.targetLane < 2) player.targetLane++;
        if (e.key === " ") player.parachutesDeployed = true; // SPACE to deploy
    });

    canvas.addEventListener('touchstart', e => {
        const touchX = e.touches[0].clientX - canvas.getBoundingClientRect().left;
        if (touchX < 180 && player.targetLane > 0) player.targetLane--;
        else if (touchX > 180 && player.targetLane < 2) player.targetLane++;
    });
</script>
</body>
</html>

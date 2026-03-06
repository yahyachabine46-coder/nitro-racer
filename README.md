<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: Levels & Garage</title>
    <style>
        body { background: #000; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; font-family: sans-serif; overflow: hidden; color: white; }
        #container { position: relative; width: 360px; height: 600px; border-radius: 30px; border: 4px solid #1a1a1a; overflow: hidden; box-shadow: 0 0 50px rgba(0,0,0,1); background: #050505; }
        canvas { display: block; }
        
        /* UI Screens */
        .screen { position: absolute; top: 0; left: 0; width: 100%; height: 100%; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 10; text-align: center; }
        h1 { color: #00f2ff; text-shadow: 0 0 20px #00f2ff; margin: 10px; }
        button { background: #ff00ff; color: white; border: none; padding: 15px 30px; border-radius: 50px; font-weight: bold; cursor: pointer; margin: 5px; box-shadow: 0 0 15px #ff00ff; }
        
        /* Garage Specific */
        .garage-options { background: #111; padding: 15px; border-radius: 15px; width: 80%; border: 1px solid #333; margin: 15px 0; }
        .color-dot { width: 30px; height: 30px; border-radius: 50%; display: inline-block; cursor: pointer; margin: 5px; border: 2px solid transparent; }
        .color-dot.active { border-color: white; transform: scale(1.2); }

        .stats { position: absolute; top: 20px; left: 20px; color: #00f2ff; font-weight: bold; z-index: 5; pointer-events: none; }
        #creditsDisplay { color: #ffcc00; font-size: 14px; }
    </style>
</head>
<body>

<div id="container">
    <div class="stats">
        <div id="lvlBox">LEVEL 1</div>
        <div id="sBox">SCORE: 0 / 10</div>
        <div id="creditsDisplay">CREDITS: 0</div>
    </div>

    <canvas id="gameCanvas" width="360" height="600"></canvas>

    <div id="menuScreen" class="screen">
        <h1>NITRO RACER</h1>
        <button onclick="showGarage()">GARAGE</button>
        <button onclick="play()">START LEVEL</button>
    </div>

    <div id="garageScreen" class="screen" style="display:none;">
        <h1>GARAGE</h1>
        <div class="garage-options">
            <p>PAINT COLOR</p>
            <div class="color-dot" style="background:#00f2ff;" onclick="setBodyColor('#00f2ff', this)"></div>
            <div class="color-dot" style="background:#ff00ff;" onclick="setBodyColor('#ff00ff', this)"></div>
            <div class="color-dot" style="background:#00ffaa;" onclick="setBodyColor('#00ffaa', this)"></div>
            <div class="color-dot" style="background:#ffcc00;" onclick="setBodyColor('#ffcc00', this)"></div>
            
            <p>UNDERGLOW</p>
            <button onclick="setGlow(20)">NEON ON</button>
            <button onclick="setGlow(0)">STEALTH</button>
        </div>
        <button onclick="hideGarage()">BACK TO MENU</button>
    </div>
</div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    
    // UI Elements
    const screens = { menu: document.getElementById('menuScreen'), garage: document.getElementById('garageScreen') };
    const sBox = document.getElementById('sBox');
    const lvlBox = document.getElementById('lvlBox');
    const credBox = document.getElementById('creditsDisplay');

    // Game Logic State
    let currentLevel = 1;
    let credits = 0;
    let goalScore = 10;
    let active = false;
    let loop;

    // Car Customization
    let carColor = '#00f2ff';
    let glowSize = 20;

    // Racing Variables
    let player = { targetLane: 1, x: 180, velocity: 0, acceleration: 0.05, parachutesDeployed: false };
    let enemies = [];
    let score = 0;
    let spawnTimer = 0;
    let finishLineY = -300;

    function drawCar(x, y, color, isPlayer) {
        ctx.save();
        if (isPlayer) {
            ctx.shadowBlur = glowSize;
            ctx.shadowColor = color;
        }
        ctx.fillStyle = color;
        ctx.beginPath();
        ctx.roundRect(x - 22, y - 10, 44, 100, 8);
        ctx.fill();

        if (isPlayer && player.parachutesDeployed) {
            ctx.fillStyle = "#fff";
            ctx.beginPath(); ctx.arc(x-20, y+105, 15, 0, Math.PI*2); ctx.arc(x+20, y+105, 15, 0, Math.PI*2); ctx.fill();
        }
        ctx.shadowBlur = 0;
        ctx.fillStyle = "rgba(0,0,0,0.8)";
        ctx.fillRect(x - 16, y + 15, 32, 20);
        ctx.restore();
    }

    function drawFinishLine(y) {
        const size = 30;
        for (let x = 30; x < 330; x += size) {
            for (let i = 0; i < 2; i++) {
                ctx.fillStyle = ((x/size + i) % 2 === 0) ? "#fff" : "#000";
                ctx.fillRect(x, y + (i * size), size, size);
            }
        }
    }

    function engine() {
        if (!active) return;
        ctx.fillStyle = "#0a0a0a"; ctx.fillRect(0, 0, 360, 600);
        ctx.fillStyle = "#111"; ctx.fillRect(30, 0, 300, 600);

        // Physics
        if (!player.parachutesDeployed) {
            player.velocity += player.acceleration;
            if (player.velocity > (12 + currentLevel)) player.velocity = (12 + currentLevel);
        } else {
            player.velocity *= 0.95;
        }

        player.x += (([75, 180, 285][player.targetLane]) - player.x) * 0.15;

        // Progress & Finish Line
        if (score < goalScore) {
            spawnTimer++;
            if (spawnTimer > Math.max(40, 100 - (currentLevel * 10))) {
                enemies.push({ x: [75, 180, 285][Math.floor(Math.random() * 3)], y: -150 });
                spawnTimer = 0;
            }
        } else {
            finishLineY += player.velocity;
            drawFinishLine(finishLineY);
            if (finishLineY > 480) winGame();
        }

        // Enemies
        for (let i = enemies.length - 1; i >= 0; i--) {
            let e = enemies[i];
            e.y += (player.velocity + 2);
            drawCar(e.x, e.y, "#ff3300", false);
            if (Math.abs(e.x - player.x) < 38 && e.y + 80 > 480 && e.y < 560) gameOver();
            if (e.y > 700) { enemies.splice(i, 1); score++; sBox.innerText = `SCORE: ${score} / ${goalScore}`; }
        }

        drawCar(player.x, 480, carColor, true);
        loop = requestAnimationFrame(engine);
    }

    // Navigation & Garage
    function showGarage() { screens.menu.style.display = 'none'; screens.garage.style.display = 'flex'; }
    function hideGarage() { screens.garage.style.display = 'none'; screens.menu.style.display = 'flex'; }
    function setBodyColor(c, el) { carColor = c; document.querySelectorAll('.color-dot').forEach(d => d.classList.remove('active')); el.classList.add('active'); }
    function setGlow(s) { glowSize = s; }

    function play() {
        goalScore = 10 + (currentLevel * 5);
        active = true; score = 0; player.velocity = 0; enemies = []; finishLineY = -300; player.parachutesDeployed = false;
        sBox.innerText = `SCORE: 0 / ${goalScore}`;
        lvlBox.innerText = `LEVEL ${currentLevel}`;
        screens.menu.style.display = 'none';
        engine();
    }

    function winGame() {
        active = false; cancelAnimationFrame(loop);
        credits += 100;
        currentLevel++;
        credBox.innerText = `CREDITS: ${credits}`;
        screens.menu.style.display = 'flex';
        document.getElementById('title').innerHTML = "LEVEL COMPLETE!<br><span style='color:#00ffaa'>+100 CREDITS</span>";
    }

    function gameOver() {
        active = false; cancelAnimationFrame(loop);
        screens.menu.style.display = 'flex';
        document.getElementById('title').innerHTML = "CRASHED!<br><span style='color:red'>RETRY LEVEL</span>";
    }

    window.addEventListener('keydown', e => {
        if (e.key === "ArrowLeft" && player.targetLane > 0) player.targetLane--;
        if (e.key === "ArrowRight" && player.targetLane < 2) player.targetLane++;
        if (e.key === " ") player.parachutesDeployed = true;
    });
</script>
</body>
</html>

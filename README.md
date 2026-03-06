<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Nitro Racer: Ghost Protocol</title>
    <style>
        * { box-sizing: border-box; touch-action: none; user-select: none; }
        body { background: #000; margin: 0; padding: 0; display: flex; justify-content: center; align-items: center; height: 100vh; width: 100vw; font-family: 'Segoe UI', sans-serif; overflow: hidden; color: white; }
        #container { position: relative; width: 100%; height: 100%; background: #050505; display: flex; justify-content: center; }
        canvas { display: block; height: 100%; aspect-ratio: 9 / 16; background: #080808; }
        .screen { position: absolute; top: 0; left: 0; width: 100%; height: 100%; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.95); z-index: 10; text-align: center; backdrop-filter: blur(15px); }
        h1 { font-size: 3rem; color: #00f2ff; text-shadow: 0 0 20px #00f2ff; margin: 5px; }
        .ui-group { background: #111; padding: 15px; border-radius: 20px; width: 340px; border: 1px solid #333; margin: 10px 0; max-height: 300px; overflow-y: auto; }
        button { background: #ff00ff; color: white; border: none; padding: 15px 30px; border-radius: 50px; font-weight: bold; cursor: pointer; margin: 5px; width: 85%; box-shadow: 0 0 10px #ff00ff; }
        .stats { position: absolute; top: 30px; left: 30px; z-index: 5; pointer-events: none; }
        .stat-line { font-weight: bold; color: #00f2ff; text-shadow: 2px 2px 4px #000; font-size: 1.2rem; }
        #ghost-msg { position: absolute; top: 100px; width: 100%; text-align: center; color: rgba(255,255,255,0.5); font-style: italic; font-size: 0.8rem; pointer-events: none; }
    </style>
</head>
<body>

<div id="container">
    <div class="stats">
        <div class="stat-line" id="lvlBox">LEVEL 1</div>
        <div class="stat-line" id="timerBox">TIME: 0.0s</div>
    </div>
    <div id="ghost-msg">GHOST ACTIVE: PREVIOUS BEST</div>
    <canvas id="gameCanvas"></canvas>

    <div id="menuScreen" class="screen">
        <h1>NITRO RACER</h1>
        <button onclick="play()">RACE AGAINST SELF</button>
        <button onclick="openTab('garage')">GARAGE</button>
        <button onclick="openTab('leaderboard')">RECORDS</button>
    </div>

    <div id="leaderboardScreen" class="screen" style="display:none;">
        <h1>FASTEST RUNS</h1>
        <div class="ui-group" id="leaderboardList"></div>
        <button onclick="openTab('menu')">BACK</button>
    </div>
    
    <div id="garageScreen" class="screen" style="display:none;">
        <h1>SELECT CAR</h1>
        <button onclick="selectCar('dragster')">🏎️ DRAGSTER (SPEED)</button>
        <button onclick="selectCar('truck')">🚛 TRUCK (ARMOR)</button>
        <button onclick="openTab('menu')">CONFIRM</button>
    </div>
</div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    
    // --- PERSISTENCE ---
    let records = JSON.parse(localStorage.getItem('nitro_records')) || [];
    let bestRunData = JSON.parse(localStorage.getItem('ghost_data')) || []; // [{x, y, frame}]
    
    // --- GAME STATE ---
    let active = false, score = 0, goalScore = 12, currentLevel = 1;
    let velocity = 0, startTime = 0, currentX = 540, laneTarget = 1;
    let enemies = [], lives = 1, currentRunData = [];
    let frameCount = 0, selectedCarType = 'dragster';

    function resize() { canvas.width = 1080; canvas.height = 1920; }
    window.addEventListener('resize', resize); resize();

    function selectCar(t) { selectedCarType = t; lives = t==='truck'?2:1; }

    function openTab(tab) {
        document.querySelectorAll('.screen').forEach(s => s.style.display = 'none');
        if(tab === 'leaderboard') renderLeaderboard();
        document.getElementById(tab + 'Screen').style.display = 'flex';
    }

    function renderLeaderboard() {
        const list = document.getElementById('leaderboardList');
        list.innerHTML = records.length ? "" : "No records yet!";
        records.sort((a,b) => a.time - b.time).slice(0, 5).forEach((r, i) => {
            list.innerHTML += `<div style="margin:8px">${i+1}. ${r.name} - ${r.time}s</div>`;
        });
    }

    function drawCar(x, y, color, alpha = 1, type = 'dragster') {
        ctx.save();
        ctx.globalAlpha = alpha;
        ctx.fillStyle = color;
        if(type === 'truck') {
            ctx.beginPath(); ctx.roundRect(x - 80, y - 20, 160, 300, 10); ctx.fill();
        } else {
            ctx.beginPath(); ctx.roundRect(x - 60, y - 20, 120, 260, 20); ctx.fill();
        }
        ctx.restore();
    }

    function engine() {
        if (!active) return;
        frameCount++;
        ctx.fillStyle = "#050505"; ctx.fillRect(0, 0, 1080, 1920);
        ctx.fillStyle = "#111"; ctx.fillRect(100, 0, 880, 1920);

        let elapsed = ((Date.now() - startTime) / 1000).toFixed(1);
        document.getElementById('timerBox').innerText = `TIME: ${elapsed}s`;

        // Physics
        velocity += (selectedCarType === 'truck' ? 0.04 : 0.07);
        if (velocity > 20) velocity = 20;
        currentX += ([250, 540, 830][laneTarget] - currentX) * 0.12;

        // Record Current Path
        currentRunData.push({x: currentX, v: velocity});

        // Draw Ghost
        if (bestRunData[frameCount]) {
            drawCar(bestRunData[frameCount].x, 1500, "#ffffff", 0.3, 'dragster');
        }

        // Enemies
        if (score < goalScore && Math.random() < 0.03) {
            enemies.push({ x: [250, 540, 830][Math.floor(Math.random() * 3)], y: -300 });
        }

        for (let i = enemies.length - 1; i >= 0; i--) {
            let e = enemies[i]; e.y += velocity + 5;
            drawCar(e.x, e.y, "#f30", 1, 'dragster');
            
            if (Math.abs(e.x - currentX) < 110 && e.y + 240 > 1500 && e.y < 1740) {
                lives--; enemies.splice(i, 1);
                if (lives <= 0) gameOver();
            }
            if (e.y > 2000) { enemies.splice(i, 1); score++; }
        }

        if (score >= goalScore) winGame(elapsed);

        drawCar(currentX, 1500, selectedCarType==='truck'?'#ffcc00':'#00f2ff', 1, selectedCarType);
        requestAnimationFrame(engine);
    }

    function play() {
        active = true; velocity = 0; score = 0; enemies = []; frameCount = 0;
        currentRunData = []; lives = selectedCarType === 'truck' ? 2 : 1;
        startTime = Date.now();
        document.getElementById('ghost-msg').style.display = bestRunData.length ? 'block' : 'none';
        openTab('none'); engine();
    }

    function winGame(t) {
        active = false;
        // Check if this is the new absolute best time
        let prevBest = records.length ? Math.min(...records.map(r => r.time)) : Infinity;
        if (parseFloat(t) < prevBest) {
            localStorage.setItem('ghost_data', JSON.stringify(currentRunData));
            bestRunData = currentRunData;
            alert("NEW GHOST SAVED!");
        }
        let name = prompt("LEVEL CLEAR! Time: " + t + "s. Name:");
        if(name) {
            records.push({ name, time: parseFloat(t) });
            localStorage.setItem('nitro_records', JSON.stringify(records));
        }
        openTab('menu');
    }

    function gameOver() { active = false; openTab('menu'); alert("CRASHED!"); }

    window.addEventListener('keydown', e => {
        if (e.key === "ArrowLeft" && laneTarget > 0) laneTarget--;
        if (e.key === "ArrowRight" && laneTarget < 2) laneTarget++;
    });
</script>
</body>
</html>

<!DOCTYPE html>
<html>
<head>
    <title>Nitro Racer: Ultimate</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: 'Segoe UI', sans-serif; }
        #game-wrap { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; aspect-ratio: 16 / 9; background: #080808; border: 2px solid #333; overflow: hidden; }
        canvas { width: 100%; height: 100%; display: block; }
        .screen { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 20; text-align: center; }
        .ui-box { background: #111; padding: 20px; border-radius: 15px; border: 1px solid #444; width: 400px; margin: 10px; }
        button { background: #ff00ff; color: white; border: none; padding: 12px 25px; font-size: 18px; font-weight: bold; cursor: pointer; border-radius: 5px; margin: 5px; width: 80%; box-shadow: 0 4px 0 #b300b3; }
        button:active { transform: translateY(2px); box-shadow: 0 2px 0 #b300b3; }
        .stats { position: absolute; top: 15px; left: 20px; z-index: 10; font-weight: bold; color: #00f2ff; pointer-events: none; line-height: 1.5; }
        #nitro-bar { position: absolute; bottom: 20px; left: 20px; width: 150px; height: 15px; background: #222; border: 1px solid #444; }
        #nitro-fill { height: 100%; background: #00f2ff; width: 100%; transition: 0.1s; }
    </style>
</head>
<body>

<div id="game-wrap">
    <div class="stats" id="statBox">STAGE 1 | 0.0s | $0</div>
    <div id="nitro-bar"><div id="nitro-fill"></div></div>

    <div id="menuScreen" class="screen">
        <h1 id="titleText" style="color:#00f2ff; font-size: 4vw; margin: 10px;">NITRO RACER</h1>
        <button onclick="startGame()">START RACE</button>
        <button onclick="showTab('garage')">GARAGE</button>
        <button onclick="showTab('settings')">SETTINGS</button>
    </div>

    <div id="garageScreen" class="screen" style="display:none;">
        <h1>GARAGE</h1>
        <div class="ui-box">
            <button onclick="setCar('dragster')">🏎️ DRAGSTER (SPEED)</button>
            <button onclick="setCar('truck')">🚛 TRUCK (ARMOR)</button>
        </div>
        <button onclick="showTab('menu')">BACK</button>
    </div>

    <div id="settingsScreen" class="screen" style="display:none;">
        <h1>SETTINGS</h1>
        <div class="ui-box">
            <button id="unitBtn" onclick="toggleUnits()">UNITS: MPH</button>
            <button id="soundBtn" onclick="toggleSound()">SOUND: ON</button>
        </div>
        <button onclick="showTab('menu')">BACK</button>
    </div>

    <canvas id="c"></canvas>
</div>

<script>
    const canvas = document.getElementById('c');
    const ctx = canvas.getContext('2d');
    
    let credits = parseInt(localStorage.getItem('nr_creds')) || 0;
    let bestRun = JSON.parse(localStorage.getItem('nr_ghost')) || [];
    let unitMode = 'MPH', soundOn = true, carColor = '#00f2ff', carType = 'dragster';
    
    let playing = false, lane = 2, playerX = 800, speed = 0, startTime = 0;
    let enemies = [], kits = [], nitro = 100, isNitro = false, frame = 0, currentPath = [];
    let lives = 1, policeActive = false, policeY = 1200, level = 1;

    const LANES = [250, 525, 800, 1075, 1350];
    canvas.width = 1600; canvas.height = 900;

    let audioCtx, osc, gain;
    function initAudio() {
        if (!audioCtx) {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            osc = audioCtx.createOscillator(); gain = audioCtx.createGain();
            osc.type = 'sawtooth'; osc.connect(gain); gain.connect(audioCtx.destination);
            gain.gain.value = 0; osc.start();
        }
    }

    function showTab(t) {
        document.querySelectorAll('.screen').forEach(s => s.style.display = 'none');
        if(t !== 'none') document.getElementById(t + 'Screen').style.display = 'flex';
    }

    function setCar(t) { carType = t; lives = t === 'truck' ? 2 : 1; }
    function toggleUnits() { unitMode = (unitMode === 'MPH' ? 'KPH' : 'MPH'); document.getElementById('unitBtn').innerText = "UNITS: " + unitMode; }
    function toggleSound() { soundOn = !soundOn; document.getElementById('soundBtn').innerText = "SOUND: " + (soundOn ? "ON" : "OFF"); }

    function drawCar(x, y, color, type = 'dragster', isPolice = false) {
        ctx.save();
        ctx.fillStyle = color;
        if (type === 'truck') ctx.fillRect(x - 70, y - 50, 140, 250);
        else ctx.fillRect(x - 50, y - 70, 100, 200);
        if (isPolice) {
            ctx.fillStyle = (frame % 10 < 5) ? "red" : "blue";
            ctx.fillRect(x - 40, y - 80, 80, 20);
        }
        ctx.fillStyle = "rgba(0,0,0,0.5)";
        ctx.fillRect(x - 30, y - 10, 60, 30);
        ctx.restore();
    }

    function drawKit(x, y) {
        ctx.fillStyle = "#00ff00";
        ctx.shadowBlur = 15; ctx.shadowColor = "#00ff00";
        ctx.beginPath(); ctx.arc(x, y, 30, 0, Math.PI*2); ctx.fill();
        ctx.fillStyle = "#fff"; ctx.fillRect(x-5, y-20, 10, 40); ctx.fillRect(x-20, y-5, 40, 10);
        ctx.shadowBlur = 0;
    }

    function startGame() {
        initAudio(); playing = true; speed = 10; lane = 2; frame = 0; level = 1;
        currentPath = []; enemies = []; kits = []; startTime = Date.now();
        policeActive = false; policeY = 1200; setCar(carType);
        showTab('none'); loop();
    }

    function loop() {
        if (!playing) return;
        frame++;
        ctx.fillStyle = "#050505"; ctx.fillRect(0,0,1600,900);
        ctx.fillStyle = "#111"; ctx.fillRect(150, 0, 1300, 900);

        if(isNitro && nitro > 0) { speed += 0.25; nitro -= 1.5; } 
        else { speed += 0.005; if(nitro < 100) nitro += 0.2; isNitro = false; }
        document.getElementById('nitro-fill').style.width = nitro + "%";
        playerX += (LANES[lane] - playerX) * 0.15;
        currentPath.push({x: playerX});

        if (speed > 25) policeActive = true;
        if (policeActive) {
            policeY -= (policeY - 850) * 0.02;
            drawCar(playerX, policeY, "#111", 'dragster', true);
            if (policeY < 870) { lives--; policeActive = false; policeY = 1200; if(lives <= 0) endGame("BUSTED!"); }
        }

        if(bestRun[frame]) drawCar(bestRun[frame].x, 750, "#ffffff44", 'dragster');

        if(soundOn) {
            osc.frequency.setTargetAtTime(60 + (speed * 12), audioCtx.currentTime, 0.1);
            gain.gain.setTargetAtTime(0.05, audioCtx.currentTime, 0.1);
        }

        // Spawning Logic
        if(Math.random() < 0.03 + (level * 0.005)) enemies.push({x: LANES[Math.floor(Math.random()*5)], y: -200});
        if(Math.random() < 0.002) kits.push({x: LANES[Math.floor(Math.random()*5)], y: -200});

        // Update Kits
        for(let i=kits.length-1; i>=0; i--) {
            kits[i].y += speed; drawKit(kits[i].x, kits[i].y);
            if(Math.abs(kits[i].x - playerX) < 80 && Math.abs(kits[i].y - 750) < 100) { kits.splice(i, 1); lives++; }
            else if(kits[i].y > 1000) kits.splice(i,1);
        }

        // Update Enemies
        for(let i=enemies.length-1; i>=0; i--) {
            enemies[i].y += speed; drawCar(enemies[i].x, enemies[i].y, "#f30");
            if(Math.abs(enemies[i].x - playerX) < 100 && Math.abs(enemies[i].y - 750) < 150) {
                lives--; enemies.splice(i, 1); if(lives <= 0) endGame("WRECKED!");
            } else if(enemies[i].y > 1000) { enemies.splice(i,1); credits += 10; }
        }

        drawCar(playerX, 750, carColor, carType);
        
        let elapsed = ((Date.now() - startTime) / 1000).toFixed(1);
        level = 1 + Math.floor(elapsed / 30);
        let displaySpeed = unitMode === 'MPH' ? Math.round(speed * 10) : Math.round(speed * 16);
        document.getElementById('statBox').innerHTML = `LEVEL ${level} | ${displaySpeed} ${unitMode} | $${credits}<br>LIVES: ${lives}`;

        requestAnimationFrame(loop);
    }

    function endGame(msg) {
        playing = false; gain.gain.value = 0;
        document.getElementById('titleText').innerText = msg;
        if(currentPath.length > bestRun.length) { localStorage.setItem('nr_ghost', JSON.stringify(currentPath)); bestRun = currentPath; }
        localStorage.setItem('nr_creds', credits);
        showTab('menu');
    }

    window.addEventListener('keydown', e => {
        if((e.key === "ArrowLeft" || e.key === "a") && lane > 0) lane--;
        if((e.key === "ArrowRight" || e.key === "d") && lane < 4) lane++;
        if(e.key === "Shift") isNitro = true;
    });
    window.addEventListener('keyup', e => { if(e.key === "Shift") isNitro = false; });
</script>
</body>
</html>

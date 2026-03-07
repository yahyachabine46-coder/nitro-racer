<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: Infinite Apex</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: 'Segoe UI', sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #050505; border: 3px solid #222; overflow: hidden; }
        canvas { width: 100%; height: 100%; display: block; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 100; }
        .btn { background: #f1c40f; color: #000; border: none; padding: 15px 40px; font-size: 20px; font-weight: bold; cursor: pointer; border-radius: 5px; margin: 10px; transition: 0.2s; min-width: 250px; }
        .btn:hover { transform: scale(1.05); background: #fff; }
        .stats { position: absolute; top: 20px; left: 20px; font-size: 24px; color: #f1c40f; font-weight: bold; pointer-events: none; z-index: 10; text-shadow: 2px 2px #000; }
        #lvl-up { position: absolute; top: 40%; width: 100%; text-align: center; font-size: 80px; font-weight: bold; color: #f1c40f; display: none; pointer-events: none; z-index: 50; text-shadow: 0 0 20px #f1c40f; }
    </style>
</head>
<body>

<div id="wrapper">
    <div id="lvl-up">LEVEL UP!</div>
    <div class="stats" id="ui-stats">CASH: $0 | LVL: 1</div>
    
    <div id="ui-menu" class="ui">
        <h1 style="font-size: 60px; color: #f1c40f; text-shadow: 0 0 30px #f1c40f; margin: 0;">NITRO RACER</h1>
        <p id="menu-sub" style="letter-spacing: 5px; margin-bottom: 20px;">INFINITE APEX</p>
        <button class="btn" onclick="startGame()">START RACE</button>
        <button class="btn" onclick="openGarage()">GARAGE</button>
    </div>

    <div id="garage-menu" class="ui" style="display:none;">
        <h1 style="color: #f1c40f;">GARAGE</h1>
        <p id="garage-cash" style="font-size: 24px; margin-bottom: 20px;">CASH: $0</p>
        <button class="btn" onclick="selectCar('Countach', '#ffffff', 0)">COUNTACH (OWNED)</button>
        <button class="btn" onclick="selectCar('Aventador', '#f1c40f', 500)">AVENTADOR ($500)</button>
        <button class="btn" onclick="selectCar('Veneno', '#3498db', 2000)">VENENO ($2000)</button>
        <button class="btn" style="background: #444; color: #fff;" onclick="closeGarage()">BACK</button>
    </div>

    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d');
    const statText = document.getElementById('ui-stats');
    const lvlFlash = document.getElementById('lvl-up');

    canvas.width = 1600; canvas.height = 900;

    let active = false, lane = 2, playerX = 800, playerY = 750;
    let speed = 0, frames = 0, isNitro = false, lineOffset = 0;
    let enemies = [], level = 1, distance = 0, finishLineY = -5000;

    const LANES = [250, 525, 800, 1075, 1350];
    const LVL_COLORS = ["#00f2ff", "#f1c40f", "#e74c3c", "#9b59b6", "#2ecc71"];

    let cash = parseInt(localStorage.getItem('lambo_cash')) || 0;
    let unlocked = JSON.parse(localStorage.getItem('lambo_unlocked')) || ['Countach'];
    let selectedCar = { name: 'Countach', color: '#ffffff' };

    function updateCashUI() {
        statText.innerText = `CASH: $${cash} | LVL: ${level}`;
        document.getElementById('garage-cash').innerText = `CASH: $${cash}`;
        localStorage.setItem('lambo_cash', cash);
    }

    function openGarage() {
        document.getElementById('ui-menu').style.display = 'none';
        document.getElementById('garage-menu').style.display = 'flex';
        updateCashUI();
    }

    function closeGarage() {
        document.getElementById('garage-menu').style.display = 'none';
        document.getElementById('ui-menu').style.display = 'flex';
    }

    function selectCar(name, color, price) {
        if (unlocked.includes(name)) {
            selectedCar = { name, color };
            alert(name + " Selected!");
        } else if (cash >= price) {
            cash -= price;
            unlocked.push(name);
            selectedCar = { name, color };
            localStorage.setItem('lambo_unlocked', JSON.stringify(unlocked));
            updateCashUI();
            alert("Unlocked " + name + "!");
        } else { alert("Insufficient Funds!"); }
    }

    function drawSpeedometer(currentSpeed) {
        const x = 1450, y = 800, r = 100;
        const speedVal = Math.round(currentSpeed * 12);
        const angle = (speedVal / 500) * Math.PI - Math.PI;
        ctx.beginPath(); ctx.arc(x, y, r, Math.PI, 0);
        ctx.strokeStyle = "#333"; ctx.lineWidth = 15; ctx.stroke();
        ctx.beginPath(); ctx.arc(x, y, r, Math.PI, angle);
        ctx.strokeStyle = isNitro ? "#f00" : LVL_COLORS[(level-1)%5];
        ctx.lineWidth = 15; ctx.stroke();
        ctx.fillStyle = "#fff"; ctx.font = "bold 30px Arial"; ctx.textAlign = "center";
        ctx.fillText(speedVal, x, y - 20);
    }

    function drawLambo(x, y, color, type, isPlayer, nitro) {
        ctx.save(); ctx.translate(x, y);
        if (isPlayer && nitro) {
            ctx.fillStyle = "#3498db"; ctx.shadowBlur = 20; ctx.shadowColor = "#3498db";
            ctx.fillRect(-35, 85, 20, 50 + Math.random()*30); ctx.fillRect(15, 85, 20, 50 + Math.random()*30);
            ctx.shadowBlur = 0;
        }
        ctx.fillStyle = color;
        ctx.beginPath(); ctx.moveTo(-55, 85); ctx.lineTo(-45, -90); ctx.lineTo(45, -90); ctx.lineTo(55, 85);
        ctx.closePath(); ctx.fill();
        ctx.fillStyle = "#111"; ctx.fillRect(-30, -35, 60, 45);
        if (type === 'Veneno') { ctx.fillStyle = "#000"; ctx.fillRect(-75, 70, 150, 15); }
        ctx.fillStyle = "#ff0000"; ctx.fillRect(-50, 80, 100, 5);
        ctx.restore();
    }

    function startGame() {
        active = true; speed = 18; lane = 2; playerX = LANES[2];
        enemies = []; frames = 0; level = 1; distance = 0; finishLineY = -5000;
        document.getElementById('ui-menu').style.display = 'none';
        updateCashUI();
        update();
    }

    function update() {
        if (!active) return;
        frames++;
        let currentSpeed = isNitro ? speed * 1.8 : speed;
        distance += currentSpeed;
        finishLineY += currentSpeed;
        lineOffset = (lineOffset + currentSpeed) % 100;

        ctx.fillStyle = "#050505"; ctx.fillRect(0,0,1600,900);
        ctx.fillStyle = "#111"; ctx.fillRect(150, 0, 1300, 900);

        // Side Rails
        ctx.fillStyle = LVL_COLORS[(level-1)%5];
        ctx.fillRect(140, 0, 10, 900);
        ctx.fillRect(1450, 0, 10, 900);

        // Lane Lines
        ctx.strokeStyle = "rgba(255,255,255,0.4)"; ctx.setLineDash([40, 60]);
        ctx.lineDashOffset = -lineOffset; ctx.lineWidth = 4;
        [387, 662, 937, 1212].forEach(x => { ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, 900); ctx.stroke(); });

        // Draw Finish Line
        if (finishLineY > -200 && finishLineY < 1200) {
            ctx.fillStyle = "#fff";
            for(let i=0; i<13; i++) {
                for(let j=0; j<2; j++) {
                    if((i+j)%2==0) ctx.fillRect(150 + (i*100), finishLineY + (j*50), 100, 50);
                }
            }
        }

        // Level Up Logic
        if (finishLineY > 800) {
            level++;
            cash += 500;
            finishLineY = -10000 - (level * 2000);
            enemies = [];
            lvlFlash.style.display = 'block';
            lvlFlash.innerText = `LEVEL ${level}`;
            setTimeout(() => { lvlFlash.style.display = 'none'; }, 2000);
            updateCashUI();
        }

        // Player
        playerX += (LANES[lane] - playerX) * 0.15;
        speed += 0.002;

        // Traffic
        if (Math.random() < 0.03 + (level * 0.01) && finishLineY < -500) {
            enemies.push({ x: LANES[Math.floor(Math.random()*5)], y: -200, color: '#e74c3c' });
        }

        for (let i = enemies.length-1; i>=0; i--) {
            let e = enemies[i]; e.y += currentSpeed * 0.7; // Traffic slower than player
            drawLambo(e.x, e.y, e.color, 'Countach', false, false);
            if (Math.abs(e.x - playerX) < 100 && Math.abs(e.y - playerY) < 150) {
                active = false;
                document.getElementById('ui-menu').style.display = 'flex';
                document.getElementById('ui-title').innerText = "WRECKED";
            }
            if (e.y > 1000) { enemies.splice(i,1); cash += 10; updateCashUI(); }
        }

        drawLambo(playerX, playerY, selectedCar.color, selectedCar.name, true, isNitro);
        drawSpeedometer(currentSpeed);
        requestAnimationFrame(update);
    }

    window.onkeydown = (e) => {
        if ((e.key === "ArrowLeft" || e.key === "a") && lane > 0) lane--;
        if ((e.key === "ArrowRight" || e.key === "d") && lane < 4) lane++;
        if (e.key === "Shift") isNitro = true;
    };
    window.onkeyup = (e) => { if (e.key === "Shift") isNitro = false; };
</script>
</body>
</html>

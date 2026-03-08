<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: Repaired</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #0a0a0a; overflow: hidden; border: 2px solid #333; }
        canvas { width: 100%; height: 100%; display: block; image-rendering: optimizeSpeed; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.92); z-index: 100; backdrop-filter: blur(5px); }
        #garage-menu, #settings-menu { display: none; }
        .stats { position: absolute; top: 20px; left: 20px; font-size: 18px; color: #f1c40f; pointer-events: none; z-index: 10; font-weight: bold; text-shadow: 2px 2px #000; }
        .btn { background: #f1c40f; color: #000; border: none; padding: 12px 25px; font-weight: bold; cursor: pointer; margin: 8px; min-width: 150px; text-transform: uppercase; }
        .btn:hover { background: #fff; }
        .btn.active { background: #3498db; color: #fff; }
        .scroll-area { width: 80%; height: 50%; overflow-y: auto; background: #111; border: 1px solid #333; margin: 15px 0; }
        .row { display: flex; justify-content: space-between; align-items: center; padding: 10px 20px; border-bottom: 1px solid #222; }
    </style>
</head>
<body>

<div id="wrapper">
    <div class="stats" id="ui-stats">CASH: $0 | LVL: 1</div>
    
    <div id="ui-menu" class="ui">
        <h1 style="letter-spacing:10px; color:#f1c40f;">NITRO RACER</h1>
        <p id="damage-warn" style="color:#e74c3c; display:none;">CAR DAMAGED - REPAIR RECOMMENDED</p>
        <button class="btn" onclick="startGame()">Race</button>
        <button class="btn" id="repair-btn" style="background:#2ecc71; display:none;" onclick="repairCar()">Repair Car ($50)</button>
        <button class="btn" style="background:#333; color:#ccc;" onclick="openGarage()">Garage</button>
        <button class="btn" style="background:#333; color:#ccc;" onclick="openSettings()">Settings</button>
    </div>

    <div id="garage-menu" class="ui">
        <h1>SHOWROOM</h1>
        <p id="garage-cash" style="color:#f1c40f">$0</p>
        <div class="scroll-area" id="garage-list"></div>
        <button class="btn" style="background:#e74c3c; color:#fff" onclick="closeGarage()">Back</button>
    </div>

    <div id="settings-menu" class="ui">
        <h1>SETTINGS</h1>
        <div class="row" style="width:320px"><span>Units</span><button class="btn" id="unit-btn" onclick="toggleUnits()">MPH</button></div>
        <button class="btn" style="background:#e74c3c; color:#fff" onclick="closeSettings()">Back</button>
    </div>

    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d', { alpha: false });
    canvas.width = 1280; canvas.height = 720;

    const CAR_DATA = [
        { name: "SVR", color: "#f1c40f", price: 0 },
        { name: "Interceptor", color: "#3498db", price: 500 },
        { name: "Venom", color: "#e74c3c", price: 1200 },
        { name: "Ghost", color: "#ecf0f1", price: 3000 }
    ];

    let active = false, lane = 2, playerX = 640, playerY = 620;
    let speed = 0, isBraking = false, isAccelerating = false, isDamaged = false;
    let enemies = [], level = 1, lineOffset = 0;
    let useKPH = localStorage.getItem('units') === 'KPH';
    let cash = parseInt(localStorage.getItem('lambo_cash')) || 0;
    let unlocked = JSON.parse(localStorage.getItem('lambo_unlocked')) || [0];
    let selectedCar = parseInt(localStorage.getItem('lambo_selected')) || 0;
    const LANES = [350, 500, 640, 780, 930];

    function toggleUnits() {
        useKPH = !useKPH;
        localStorage.setItem('units', useKPH ? 'KPH' : 'MPH');
        document.getElementById('unit-btn').innerText = useKPH ? 'KPH' : 'MPH';
    }

    function repairCar() {
        if(cash >= 50) {
            cash -= 50; isDamaged = false;
            document.getElementById('repair-btn').style.display = 'none';
            document.getElementById('damage-warn').style.display = 'none';
            localStorage.setItem('lambo_cash', cash);
            updateStats();
        }
    }

    function openGarage() { document.getElementById('ui-menu').style.display='none'; document.getElementById('garage-menu').style.display='flex'; updateGarageUI(); }
    function closeGarage() { document.getElementById('garage-menu').style.display='none'; document.getElementById('ui-menu').style.display='flex'; }
    function openSettings() { document.getElementById('ui-menu').style.display='none'; document.getElementById('settings-menu').style.display='flex'; }
    function closeSettings() { document.getElementById('settings-menu').style.display='none'; document.getElementById('ui-menu').style.display='flex'; }

    function updateGarageUI() {
        document.getElementById('garage-cash').innerText = `CASH: $${cash}`;
        const list = document.getElementById('garage-list');
        list.innerHTML = '';
        CAR_DATA.forEach((car, i) => {
            const isOwned = unlocked.includes(i);
            list.innerHTML += `<div class="row"><span>${car.name}</span><button class="btn ${selectedCar === i ? 'active' : ''}" onclick="buyOrSelectCar(${i})">${selectedCar === i ? 'SELECTED' : (isOwned ? 'SELECT' : '$'+car.price)}</button></div>`;
        });
    }

    function buyOrSelectCar(i) {
        if (unlocked.includes(i)) { selectedCar = i; } 
        else if (cash >= CAR_DATA[i].price) {
            cash -= CAR_DATA[i].price; unlocked.push(i); selectedCar = i;
            localStorage.setItem('lambo_unlocked', JSON.stringify(unlocked));
        }
        localStorage.setItem('lambo_selected', selectedCar);
        localStorage.setItem('lambo_cash', cash);
        updateGarageUI();
    }

    function updateStats() {
        const factor = useKPH ? 8 : 5;
        const unit = useKPH ? 'KPH' : 'MPH';
        document.getElementById('ui-stats').innerText = `CASH: $${cash} | LVL: ${level} | ${Math.round(speed * factor)} ${unit}`;
    }

    // FIXED CAR DRAWING FUNCTION
    function drawCar(x, y, color, braking, damaged) {
        ctx.save();
        ctx.translate(x, y);
        
        // Shadow
        ctx.fillStyle = "rgba(0,0,0,0.3)";
        ctx.fillRect(-22, -45, 50, 100);

        // Body (Full Rectangle Fix)
        ctx.fillStyle = color;
        ctx.fillRect(-25, -50, 50, 100);

        // Cockpit
        ctx.fillStyle = "#111";
        ctx.fillRect(-18, -35, 36, 30);

        // Brake Lights
        ctx.fillStyle = braking ? "#ff3300" : "#600";
        ctx.fillRect(-22, 42, 12, 6);
        ctx.fillRect(10, 42, 12, 6);

        // Damage Smoke Effect
        if(damaged) {
            ctx.fillStyle = "rgba(100,100,100,0.5)";
            ctx.beginPath(); ctx.arc(0, -60, 10 + Math.random()*5, 0, Math.PI*2); ctx.fill();
        }
        
        ctx.restore();
    }

    function startGame() {
        active = true; speed = 0; enemies = []; 
        document.getElementById('ui-menu').style.display = 'none';
        requestAnimationFrame(update);
    }

    function update() {
        if (!active) return;

        if (isBraking) speed -= 0.6;
        else if (isAccelerating) speed += 0.08;
        else speed -= 0.03;
        speed = Math.max(0, Math.min(speed, 35 + level));
        lineOffset = (lineOffset + speed) % 100;

        ctx.fillStyle = "#1a1a1a"; ctx.fillRect(0, 0, 1280, 720); // Road
        ctx.fillStyle = "#333"; ctx.fillRect(300, 0, 680, 720); // Lanes
        
        ctx.strokeStyle = "#444"; ctx.setLineDash([40, 60]); ctx.lineDashOffset = -lineOffset;
        ctx.beginPath(); [470, 640, 810].forEach(lx => { ctx.moveTo(lx, 0); ctx.lineTo(lx, 720); }); ctx.stroke();

        if (Math.random() < 0.012) enemies.push({ x: LANES[Math.floor(Math.random()*5)], y: -200, bSpeed: 10 + Math.random()*10 });

        for (let i = enemies.length - 1; i >= 0; i--) {
            let e = enemies[i];
            e.y += (speed - e.baseSpeed || speed - 15);
            drawCar(e.x, e.y, "#444", false, false);

            if (Math.abs(e.x - playerX) < 45 && Math.abs(e.y - playerY) < 95) {
                active = false; isDamaged = true;
                localStorage.setItem('lambo_cash', cash);
                document.getElementById('repair-btn').style.display = 'block';
                document.getElementById('damage-warn').style.display = 'block';
                document.getElementById('ui-menu').style.display = 'flex';
            }
            if (e.y > 850 || e.y < -500) { 
                if(e.y > 850) { cash += 10; updateStats(); }
                enemies.splice(i, 1); 
            }
        }

        playerX += (LANES[lane] - playerX) * 0.12;
        drawCar(playerX, playerY, CAR_DATA[selectedCar].color, isBraking, isDamaged);
        requestAnimationFrame(update);
    }

    window.onkeydown = (e) => {
        if (e.key === "ArrowLeft" && lane > 0) lane--;
        if (e.key === "ArrowRight" && lane < 4) lane--;
        if (e.key === "ArrowUp") isAccelerating = true;
        if (e.key === " ") isBraking = true;
    };
    window.onkeyup = (e) => {
        if (e.key === "ArrowUp") isAccelerating = false;
        if (e.key === " ") isBraking = false;
    };
    updateStats();
</script>
</body>
</html>

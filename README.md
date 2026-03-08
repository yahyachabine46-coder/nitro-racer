<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: Pro Highway</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: 'Segoe UI', sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #0a0a0a; border: 2px solid #333; overflow: hidden; }
        canvas { width: 100%; height: 100%; display: block; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 100; backdrop-filter: blur(10px); }
        #garage-menu, #settings-menu { display: none; padding: 20px; }
        .scroll-area { width: 80%; height: 55%; overflow-y: auto; background: rgba(0,0,0,0.5); padding: 15px; border-radius: 10px; margin: 20px 0; border: 1px solid #444; }
        .row { display: flex; justify-content: space-between; align-items: center; background: #151515; margin-bottom: 8px; padding: 12px 20px; border-radius: 4px; }
        .btn { background: #f1c40f; color: #000; border: none; padding: 12px 24px; font-size: 16px; font-weight: bold; cursor: pointer; border-radius: 3px; transition: 0.2s; text-transform: uppercase; letter-spacing: 1px; }
        .btn:hover { background: #fff; transform: scale(1.05); }
        .btn.active { background: #3498db; color: #fff; }
        .stats { position: absolute; top: 25px; left: 25px; font-size: 20px; color: #f1c40f; font-weight: 300; pointer-events: none; z-index: 10; }
        input[type=range] { cursor: pointer; accent-color: #f1c40f; }
    </style>
</head>
<body>

<div id="wrapper">
    <div class="stats" id="ui-stats">CASH: $0 | LVL: 1 | 0 MPH</div>
    
    <div id="ui-menu" class="ui">
        <h1>Nitro Racer</h1>
        <p style="color: #666;">HIGHWAY SIMULATOR</p>
        <button class="btn" style="width: 200px; margin-top: 20px;" onclick="startGame()">Race</button>
        <button class="btn" style="width: 200px; margin-top: 10px; background: #333; color: #ccc;" onclick="openGarage()">Garage</button>
        <button class="btn" style="width: 200px; margin-top: 10px; background: #333; color: #ccc;" onclick="openSettings()">Settings</button>
    </div>

    <div id="settings-menu" class="ui">
        <h1>Settings</h1>
        <div class="scroll-area">
            <div class="row">
                <span>Speed Units</span>
                <button id="unit-btn" class="btn" onclick="toggleUnits()">MPH</button>
            </div>
            <div class="row">
                <span>Engine Pitch</span>
                <input type="range" id="sound-range" min="100" max="1000" value="500">
            </div>
        </div>
        <button class="btn" style="background: #e74c3c; color: #fff;" onclick="closeSettings()">Save & Back</button>
    </div>

    <div id="garage-menu" class="ui">
        <h1>Showroom</h1>
        <p id="garage-cash" style="color: #f1c40f;">$0</p>
        <div class="scroll-area" id="garage-list"></div>
        <button class="btn" style="background: #e74c3c; color: #fff;" onclick="closeGarage()">Back</button>
    </div>

    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d');
    canvas.width = 1600; canvas.height = 900;

    // Car Data
    const carNames = ["Aventador", "Huracán", "Veneno", "Sian", "Countach", "Murciélago"];
    const CAR_TYPES = carNames.map((name, i) => ({
        id: i, name: name, color: `hsl(${(i * 50) % 360}, 80%, 45%)`, price: i * 500
    }));

    // State
    let active = false, lane = 2, playerX = 800, playerY = 780;
    let speed = 0, isNitro = false, isBraking = false, isAccelerating = false;
    let lineOffset = 0, enemies = [], level = 1, finishLineY = -8000;
    let useKPH = localStorage.getItem('units') === 'KPH';
    const LANES = [450, 625, 800, 975, 1150];

    let cash = parseInt(localStorage.getItem('lambo_cash')) || 0;
    let unlocked = JSON.parse(localStorage.getItem('lambo_unlocked_ids')) || [0];
    let selectedCarIndex = 0;

    function toggleUnits() {
        useKPH = !useKPH;
        localStorage.setItem('units', useKPH ? 'KPH' : 'MPH');
        document.getElementById('unit-btn').innerText = useKPH ? 'KPH' : 'MPH';
    }

    function openSettings() { document.getElementById('ui-menu').style.display='none'; document.getElementById('settings-menu').style.display='flex'; }
    function closeSettings() { document.getElementById('settings-menu').style.display='none'; document.getElementById('ui-menu').style.display='flex'; }
    function openGarage() { document.getElementById('ui-menu').style.display='none'; document.getElementById('garage-menu').style.display='flex'; updateCashUI(); }
    function closeGarage() { document.getElementById('garage-menu').style.display='none'; document.getElementById('ui-menu').style.display='flex'; }

    function updateCashUI() {
        const factor = useKPH ? 8 : 5;
        const unitLabel = useKPH ? 'KPH' : 'MPH';
        document.getElementById('ui-stats').innerText = `CASH: $${cash} | LVL: ${level} | ${Math.round(speed * factor)} ${unitLabel}`;
        document.getElementById('garage-cash').innerText = `TOTAL FUNDS: $${cash}`;
        localStorage.setItem('lambo_cash', cash);
        renderGarage();
    }

    function renderGarage() {
        const list = document.getElementById('garage-list');
        list.innerHTML = '';
        CAR_TYPES.forEach((car, i) => {
            const isUnlocked = unlocked.includes(i);
            list.innerHTML += `<div class="row">
                <span>${car.name}</span>
                <button class="btn ${selectedCarIndex === i ? 'active' : ''}" onclick="selectCar(${i})">
                ${selectedCarIndex === i ? 'Selected' : (isUnlocked ? 'Select' : '$'+car.price)}</button>
            </div>`;
        });
    }

    function selectCar(i) {
        if (unlocked.includes(i)) selectedCarIndex = i;
        else if (cash >= CAR_TYPES[i].price) { cash -= CAR_TYPES[i].price; unlocked.push(i); selectedCarIndex = i; }
        updateCashUI();
    }

    function drawCar(x, y, color, braking) {
        ctx.save();
        ctx.translate(x, y);
        // Body
        ctx.fillStyle = color;
        ctx.beginPath(); ctx.moveTo(-28, 55); ctx.bezierCurveTo(-32, 20, -32, -20, -25, -60);
        ctx.bezierCurveTo(25, -60, 32, 20, 28, 55); ctx.fill();
        // Glass
        ctx.fillStyle = "#111";
        ctx.fillRect(-20, -25, 40, 30);
        // Brake Lights
        ctx.fillStyle = braking ? "#ff0000" : "#600";
        if(braking) ctx.shadowBlur = 15, ctx.shadowColor = "red";
        ctx.fillRect(-26, 50, 15, 6); ctx.fillRect(11, 50, 15, 6);
        ctx.restore();
    }

    function startGame() {
        active = true; speed = 0; enemies = []; 
        document.getElementById('ui-menu').style.display = 'none';
        update();
    }

    function update() {
        if (!active) return;

        // Physics
        if (isBraking) speed -= 0.6;
        else if (isAccelerating) speed += 0.1;
        else speed -= 0.03;
        speed = Math.max(0, Math.min(speed, 40 + level * 2));

        lineOffset = (lineOffset + speed) % 100;
        finishLineY += speed;

        // Road
        ctx.fillStyle = "#151515"; ctx.fillRect(0,0,1600,900);
        ctx.fillStyle = "#1a1a1a"; ctx.fillRect(350, 0, 900, 900);
        ctx.strokeStyle = "rgba(255,255,255,0.1)"; ctx.setLineDash([40, 60]); ctx.lineDashOffset = -lineOffset;
        [537, 712, 887, 1062].forEach(x => { ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, 900); ctx.stroke(); });

        // Traffic
        if (Math.random() < 0.02) enemies.push({ x: LANES[Math.floor(Math.random()*5)], y: -200, baseSpeed: 15 + Math.random()*10 });

        for (let i = enemies.length-1; i>=0; i--) {
            let e = enemies[i];
            // Independent movement: Enemy moves relative to their own speed vs yours
            e.y += (speed - e.baseSpeed) * -1; 
            drawCar(e.x, e.y, "#444", false);
            
            if (Math.abs(e.x - playerX) < 50 && Math.abs(e.y - playerY) < 100) {
                active = false; document.getElementById('ui-menu').style.display = 'flex';
            }
            if (e.y > 1000) { enemies.splice(i,1); cash += 10; updateCashUI(); }
        }

        playerX += (LANES[lane] - playerX) * 0.15;
        drawCar(playerX, playerY, CAR_TYPES[selectedCarIndex].color, isBraking);
        
        updateCashUI();
        requestAnimationFrame(update);
    }

    window.onkeydown = (e) => {
        if (e.key === "ArrowLeft") if(lane>0) lane--;
        if (e.key === "ArrowRight") if(lane<4) lane--;
        if (e.key === "ArrowUp") isAccelerating = true;
        if (e.key === " ") isBraking = true;
    };
    window.onkeyup = (e) => {
        if (e.key === "ArrowUp") isAccelerating = false;
        if (e.key === " ") isBraking = false;
    };

    document.getElementById('unit-btn').innerText = useKPH ? 'KPH' : 'MPH';
    updateCashUI();
</script>
</body>
</html>

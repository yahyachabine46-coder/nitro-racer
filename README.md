<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: JDM Legends</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #050505; overflow: hidden; border: 2px solid #444; }
        canvas { width: 100%; height: 100%; display: block; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 100; backdrop-filter: blur(10px); }
        .stats { position: absolute; top: 20px; left: 20px; font-size: 18px; color: #f1c40f; z-index: 10; font-weight: bold; }
        .btn { background: #f1c40f; color: #000; border: none; padding: 12px 20px; font-weight: bold; cursor: pointer; margin: 5px; border-radius: 4px; }
        #garage-menu { display: none; }
        .garage-container { width: 85%; height: 75%; display: flex; flex-direction: column; background: #111; border: 1px solid #333; padding: 20px; border-radius: 8px; }
        #search-bar { width: 100%; padding: 12px; margin-bottom: 15px; background: #222; border: 1px solid #444; color: #fff; font-size: 16px; border-radius: 4px; outline: none; }
        #search-bar:focus { border-color: #f1c40f; }
        .scroll-area { flex-grow: 1; overflow-y: auto; }
        .car-card { display: flex; justify-content: space-between; align-items: center; background: #1a1a1a; margin-bottom: 8px; padding: 15px; border-radius: 4px; }
        .car-info { display: flex; align-items: center; gap: 15px; }
        .swatch { width: 30px; height: 15px; border-radius: 2px; }
    </style>
</head>
<body>

<div id="wrapper">
    <div class="stats" id="ui-stats">CASH: $0 | LVL: 1</div>
    <div id="ui-menu" class="ui">
        <h1 style="letter-spacing:10px; color:#f1c40f;">NITRO RACER</h1>
        <button class="btn" onclick="startGame()">START RACE</button>
        <button class="btn" style="background:#444; color:#fff;" onclick="openGarage()">OPEN GARAGE</button>
    </div>
    <div id="garage-menu" class="ui">
        <div class="garage-container">
            <h2>GARAGE</h2>
            <input type="text" id="search-bar" placeholder="Search (Try 'Nissan')..." oninput="filterGarage()">
            <div class="scroll-area" id="garage-list"></div>
            <div style="display:flex; justify-content:space-between; margin-top:15px;">
                <span id="garage-cash" style="color:#f1c40f;">$0</span>
                <button class="btn" style="background:#e74c3c; color:#fff;" onclick="closeGarage()">CLOSE</button>
            </div>
        </div>
    </div>
    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d');
    canvas.width = 1280; canvas.height = 720;

    const CAR_DATABASE = [
        { name: "Starter Sedan", color: "#888", price: 0, w: 50, h: 90 },
        { name: "Nissan Skyline GT-R R34", color: "#2e86de", price: 8000, w: 52, h: 95 },
        { name: "Nissan GT-R R35", color: "#57606f", price: 15000, w: 64, h: 92 },
        { name: "Yellow Taxi", color: "#f1c40f", price: 500, w: 50, h: 90 },
        { name: "Sport GT", color: "#e74c3c", price: 1200, w: 52, h: 85 },
        { name: "V12 Hyper", color: "#9b59b6", price: 25000, w: 58, h: 82 },
        { name: "Aventador SV", color: "#ff9f43", price: 45000, w: 60, h: 85 }
    ];

    let active = false, lane = 2, playerX = 640, playerY = 600, speed = 0;
    let isBraking = false, isAccelerating = false, lineOffset = 0, enemies = [];
    let cash = parseInt(localStorage.getItem('lambo_cash')) || 0;
    let unlocked = JSON.parse(localStorage.getItem('lambo_unlocked')) || [0];
    let selectedIdx = parseInt(localStorage.getItem('lambo_selected')) || 0;
    const LANES = [350, 495, 640, 785, 930];

    function openGarage() {
        document.getElementById('ui-menu').style.display = 'none';
        document.getElementById('garage-menu').style.display = 'flex';
        renderGarage(CAR_DATABASE);
    }
    function closeGarage() { document.getElementById('garage-menu').style.display = 'none'; document.getElementById('ui-menu').style.display = 'flex'; }
    
    function renderGarage(listToRender) {
        const list = document.getElementById('garage-list');
        document.getElementById('garage-cash').innerText = `CASH: $${cash}`;
        list.innerHTML = '';
        listToRender.forEach((car) => {
            const idx = CAR_DATABASE.findIndex(c => c.name === car.name);
            const isOwned = unlocked.includes(idx);
            const isCurrent = selectedIdx === idx;
            list.innerHTML += `
                <div class="car-card">
                    <div class="car-info"><div class="swatch" style="background:${car.color}"></div><b>${car.name}</b></div>
                    <button class="btn" style="background:${isCurrent?'#3498db':(isOwned?'#f1c40f':'#444')}" onclick="handlePurchase(${idx})">
                        ${isCurrent ? 'SELECTED' : (isOwned ? 'SELECT' : '$' + car.price)}
                    </button>
                </div>`;
        });
    }

    function filterGarage() {
        const term = document.getElementById('search-bar').value.toLowerCase();
        renderGarage(CAR_DATABASE.filter(car => car.name.toLowerCase().includes(term)));
    }

    function handlePurchase(idx) {
        if (unlocked.includes(idx)) { selectedIdx = idx; } 
        else if (cash >= CAR_DATABASE[idx].price) { cash -= CAR_DATABASE[idx].price; unlocked.push(idx); selectedIdx = idx; }
        localStorage.setItem('lambo_cash', cash);
        localStorage.setItem('lambo_unlocked', JSON.stringify(unlocked));
        localStorage.setItem('lambo_selected', selectedIdx);
        renderGarage(CAR_DATABASE);
    }

    function drawCar(x, y, car, braking) {
        ctx.save(); ctx.translate(x, y);
        ctx.fillStyle = car.color; ctx.fillRect(-car.w/2, -car.h/2, car.w, car.h);
        ctx.fillStyle = "rgba(0,0,0,0.7)"; ctx.fillRect(-car.w/2.5, -car.h/4, car.w/1.25, car.h/3);
        ctx.fillStyle = braking ? "red" : "#700"; ctx.fillRect(-car.w/2, car.h/2-5, 10, 5); ctx.fillRect(car.w/2-10, car.h/2-5, 10, 5);
        ctx.restore();
    }

    function startGame() { active = true; speed = 0; enemies = []; document.getElementById('ui-menu').style.display = 'none'; requestAnimationFrame(update); }

    function update() {
        if (!active) return;
        if (isBraking) speed -= 0.6; else if (isAccelerating) speed += 0.1; else speed -= 0.05;
        speed = Math.max(0, Math.min(speed, 40));
        lineOffset = (lineOffset + speed) % 100;
        ctx.fillStyle = "#111"; ctx.fillRect(0,0,1280,720);
        ctx.fillStyle = "#222"; ctx.fillRect(280, 0, 720, 720);
        ctx.strokeStyle = "#444"; ctx.setLineDash([40,60]); ctx.lineDashOffset = -lineOffset;
        ctx.beginPath(); [422, 567, 712, 857].forEach(lx => { ctx.moveTo(lx,0); ctx.lineTo(lx,720); }); ctx.stroke();

        if (Math.random() < 0.015) enemies.push({ x: LANES[Math.floor(Math.random()*5)], y: -200, bSpeed: 10+Math.random()*10, model: CAR_DATABASE[Math.floor(Math.random()*CAR_DATABASE.length)] });

        for (let i = enemies.length - 1; i >= 0; i--) {
            let e = enemies[i]; e.y += (speed - e.bSpeed);
            drawCar(e.x, e.y, e.model, false);
            if (Math.abs(e.x - playerX) < 40 && Math.abs(e.y - playerY) < 80) { active = false; document.getElementById('ui-menu').style.display = 'flex'; }
            if (e.y > 1000 || e.y < -600) { if(e.y > 1000) cash += 25; enemies.splice(i, 1); }
        }
        playerX += (LANES[lane] - playerX) * 0.15;
        drawCar(playerX, playerY, CAR_DATABASE[selectedIdx], isBraking);
        document.getElementById('ui-stats').innerText = `CASH: $${cash} | ${Math.round(speed*5)} KPH`;
        requestAnimationFrame(update);
    }

    window.onkeydown = (e) => {
        if (e.key === "ArrowLeft" && lane > 0) lane--;
        if (e.key === "ArrowRight" && lane < 4) lane--;
        if (e.key === "ArrowUp") isAccelerating = true;
        if (e.key === " ") isBraking = true;
    };
    window.onkeyup = (e) => { if (e.key === "ArrowUp") isAccelerating = false; if (e.key === " ") isBraking = false; };
</script>
</body>
</html>

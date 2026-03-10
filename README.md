<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: Aero Edition</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #050505; overflow: hidden; border: 2px solid #444; }
        canvas { width: 100%; height: 100%; display: block; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 100; backdrop-filter: blur(10px); }
        .stats { position: absolute; top: 20px; left: 20px; font-size: 18px; color: #f1c40f; z-index: 10; font-weight: bold; }
        .btn { background: #f1c40f; color: #000; border: none; padding: 12px 20px; font-weight: bold; cursor: pointer; margin: 5px; border-radius: 4px; }
        #garage-menu { display: none; }
        .garage-container { width: 85%; height: 75%; display: flex; flex-direction: column; background: #111; border: 1px solid #333; padding: 20px; border-radius: 8px; }
        #search-bar { width: 100%; padding: 12px; margin-bottom: 15px; background: #222; border: 1px solid #444; color: #fff; }
        .scroll-area { flex-grow: 1; overflow-y: auto; }
        .car-card { display: flex; justify-content: space-between; align-items: center; background: #1a1a1a; margin-bottom: 8px; padding: 15px; border-radius: 10px; }
    </style>
</head>
<body>

<div id="wrapper">
    <div class="stats" id="ui-stats">CASH: $0 | 0 KPH</div>
    <div id="ui-menu" class="ui">
        <h1 style="letter-spacing:10px; color:#f1c40f;">NITRO RACER</h1>
        <button class="btn" onclick="startGame()">START RACE</button>
        <button class="btn" style="background:#444; color:#fff;" onclick="openGarage()">GARAGE</button>
    </div>
    <div id="garage-menu" class="ui">
        <div class="garage-container">
            <h2>GARAGE</h2>
            <input type="text" id="search-bar" placeholder="Search Nissan, GT-R..." oninput="filterGarage()">
            <div class="scroll-area" id="garage-list"></div>
            <button class="btn" style="background:#e74c3c; color:#fff; width: 100%; margin-top: 10px;" onclick="closeGarage()">CLOSE</button>
        </div>
    </div>
    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d');
    canvas.width = 1280; canvas.height = 720;

    const CAR_DATABASE = [
        { name: "Starter Sedan", color: "#888", price: 0, w: 52, h: 95, r: 15 },
        { name: "Nissan Skyline R34", color: "#2e86de", price: 8000, w: 54, h: 100, r: 12 },
        { name: "Nissan GT-R R35", color: "#57606f", price: 15000, w: 62, h: 96, r: 20 },
        { name: "Supra MK4", color: "#d35400", price: 12000, w: 56, h: 98, r: 25 },
        { name: "Aventador", color: "#f1c40f", price: 45000, w: 65, h: 105, r: 8 }
    ];

    let active = false, lane = 2, playerX = 640, playerY = 600, speed = 0;
    let isBraking = false, isAccelerating = false, lineOffset = 0, enemies = [];
    let cash = parseInt(localStorage.getItem('lambo_cash')) || 0;
    let unlocked = JSON.parse(localStorage.getItem('lambo_unlocked')) || [0];
    let selectedIdx = parseInt(localStorage.getItem('lambo_selected')) || 0;
    const LANES = [350, 495, 640, 785, 930];

    // FIX: Input protection
    let canChangeLane = true;

    function openGarage() { document.getElementById('ui-menu').style.display='none'; document.getElementById('garage-menu').style.display='flex'; renderGarage(CAR_DATABASE); }
    function closeGarage() { document.getElementById('garage-menu').style.display='none'; document.getElementById('ui-menu').style.display='flex'; }

    function renderGarage(listToRender) {
        const list = document.getElementById('garage-list');
        list.innerHTML = '';
        listToRender.forEach((car) => {
            const idx = CAR_DATABASE.findIndex(c => c.name === car.name);
            const isOwned = unlocked.includes(idx);
            list.innerHTML += `<div class="car-card"><b>${car.name}</b><button class="btn" style="background:${selectedIdx==idx?'#3498db':(isOwned?'#f1c40f':'#444')}" onclick="handlePurchase(${idx})">${selectedIdx==idx?'SELECTED':(isOwned?'SELECT':'$'+car.price)}</button></div>`;
        });
    }

    function handlePurchase(idx) {
        if (unlocked.includes(idx)) { selectedIdx = idx; } 
        else if (cash >= CAR_DATABASE[idx].price) { cash -= CAR_DATABASE[idx].price; unlocked.push(idx); selectedIdx = idx; }
        localStorage.setItem('lambo_cash', cash); localStorage.setItem('lambo_unlocked', JSON.stringify(unlocked)); localStorage.setItem('lambo_selected', selectedIdx);
        renderGarage(CAR_DATABASE);
    }

    function filterGarage() {
        const term = document.getElementById('search-bar').value.toLowerCase();
        renderGarage(CAR_DATABASE.filter(car => car.name.toLowerCase().includes(term)));
    }

    // NEW: Rounded Drawing Function
    function drawRoundedCar(x, y, car, braking) {
        ctx.save();
        ctx.translate(x, y);
        
        // Body shadow
        ctx.fillStyle = "rgba(0,0,0,0.3)";
        ctx.beginPath();
        ctx.roundRect(-car.w/2 + 5, -car.h/2 + 5, car.w, car.h, car.r);
        ctx.fill();

        // Main Body
        ctx.fillStyle = car.color;
        ctx.beginPath();
        ctx.roundRect(-car.w/2, -car.h/2, car.w, car.h, car.r);
        ctx.fill();

        // Windshield (Rounded)
        ctx.fillStyle = "rgba(0,0,0,0.7)";
        ctx.beginPath();
        ctx.roundRect(-car.w/2.5, -car.h/4, car.w/1.25, car.h/3, 5);
        ctx.fill();

        // Headlights
        ctx.fillStyle = "#fff";
        ctx.beginPath(); ctx.arc(-car.w/2.5, -car.h/2 + 10, 4, 0, Math.PI*2); ctx.fill();
        ctx.beginPath(); ctx.arc(car.w/2.5, -car.h/2 + 10, 4, 0, Math.PI*2); ctx.fill();

        // Taillights
        ctx.fillStyle = braking ? "#ff0000" : "#700";
        if(braking) ctx.shadowBlur = 10, ctx.shadowColor = "red";
        ctx.fillRect(-car.w/2, car.h/2-8, 12, 6);
        ctx.fillRect(car.w/2-12, car.h/2-8, 12, 6);

        ctx.restore();
    }

    function startGame() { active = true; speed = 0; enemies = []; lane = 2; playerX = LANES[2]; document.getElementById('ui-menu').style.display='none'; requestAnimationFrame(update); }

    function update() {
        if (!active) return;
        if (isBraking) speed -= 0.7; else if (isAccelerating) speed += 0.12; else speed -= 0.04;
        speed = Math.max(0, Math.min(speed, 45));
        lineOffset = (lineOffset + speed) % 100;

        ctx.fillStyle = "#111"; ctx.fillRect(0,0,1280,720);
        ctx.fillStyle = "#222"; ctx.fillRect(280, 0, 720, 720);
        ctx.strokeStyle = "#444"; ctx.setLineDash([40,60]); ctx.lineDashOffset = -lineOffset;
        ctx.beginPath(); [422, 567, 712, 857].forEach(lx => { ctx.moveTo(lx,0); ctx.lineTo(lx,720); }); ctx.stroke();

        if (Math.random() < 0.012) enemies.push({ x: LANES[Math.floor(Math.random()*5)], y: -200, bSpeed: 10+Math.random()*15, model: CAR_DATABASE[Math.floor(Math.random()*CAR_DATABASE.length)] });

        for (let i = enemies.length - 1; i >= 0; i--) {
            let e = enemies[i]; e.y += (speed - e.bSpeed);
            drawRoundedCar(e.x, e.y, e.model, false);
            if (Math.abs(e.x - playerX) < 40 && Math.abs(e.y - playerY) < 80) { active = false; document.getElementById('ui-menu').style.display='flex'; }
            if (e.y > 1000 || e.y < -600) { if(e.y > 1000) cash += 25; enemies.splice(i, 1); }
        }

        // SMOOTH STEERING: Fixed drift
        let targetX = LANES[lane];
        playerX += (targetX - playerX) * 0.2;

        drawRoundedCar(playerX, playerY, CAR_DATABASE[selectedIdx], isBraking);
        document.getElementById('ui-stats').innerText = `CASH: $${cash} | ${Math.round(speed*5)} KPH`;
        requestAnimationFrame(update);
    }

    window.onkeydown = (e) => {
        if (e.repeat) return;
        if (e.key === "ArrowLeft" && lane > 0) lane--;
        if (e.key === "ArrowRight" && lane < 4) lane++;
        if (e.key === "ArrowUp") isAccelerating = true;
        if (e.key === " ") isBraking = true;
    };
    window.onkeyup = (e) => {
        if (e.key === "ArrowUp") isAccelerating = false;
        if (e.key === " ") isBraking = false;
    };
</script>
</body>
</html>  

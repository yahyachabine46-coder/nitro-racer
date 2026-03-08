<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: Micro 100</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: 'Segoe UI', sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #050505; border: 3px solid #222; overflow: hidden; }
        canvas { width: 100%; height: 100%; display: block; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 100; }
        #garage-menu { padding: 20px; display: none; }
        .scroll-area { width: 80%; height: 60%; overflow-y: auto; background: rgba(255,255,255,0.05); padding: 20px; border-radius: 10px; margin: 20px 0; border: 1px solid #333; }
        .car-row { display: flex; justify-content: space-between; align-items: center; background: #1a1a1a; margin-bottom: 10px; padding: 15px; border-radius: 5px; border-left: 5px solid #f1c40f; }
        .btn { background: #f1c40f; color: #000; border: none; padding: 10px 20px; font-size: 16px; font-weight: bold; cursor: pointer; border-radius: 5px; transition: 0.2s; text-transform: uppercase; }
        .btn:hover { background: #fff; transform: scale(1.05); }
        .btn.owned { background: #2ecc71; color: #fff; }
        .btn.active { background: #3498db; color: #fff; border: 2px solid #fff; }
        .stats { position: absolute; top: 20px; left: 20px; font-size: 24px; color: #f1c40f; font-weight: bold; pointer-events: none; z-index: 10; text-shadow: 2px 2px #000; }
        h1 { margin: 10px 0; text-shadow: 0 0 20px #f1c40f; }
    </style>
</head>
<body>

<div id="wrapper">
    <div class="stats" id="ui-stats">CASH: $0 | LVL: 1</div>
    <div id="ui-menu" class="ui">
        <h1 style="font-size: 60px; color: #f1c40f;">NITRO RACER</h1>
        <p style="letter-spacing: 5px; margin-bottom: 20px;">MICRO 5-LANE HIGHWAY</p>
        <button class="btn" style="padding: 20px 60px; font-size: 24px;" onclick="startGame()">RACE</button>
        <button class="btn" style="margin-top: 10px;" onclick="openGarage()">GARAGE</button>
    </div>
    <div id="garage-menu" class="ui">
        <h1>THE GARAGE</h1>
        <p id="garage-cash" style="font-size: 24px;">CASH: $0</p>
        <div class="scroll-area" id="garage-list"></div>
        <button class="btn" style="background: #e74c3c; color: #fff;" onclick="closeGarage()">BACK</button>
    </div>
    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d');
    canvas.width = 1600; canvas.height = 900;

    // --- CAR GENERATOR (100 CARS) ---
    const carNames = ["Countach", "Aventador", "Veneno", "Diablo", "Murciélago", "Gallardo", "Huracán", "Sian", "Terzo", "Reventón", "Centenario", "Miura", "Urus", "Sesto", "Egoista", "Jalpa", "Jarama", "Urraco", "Espada", "Islero"];
    const CAR_TYPES = [];
    for(let i=0; i<100; i++) {
        CAR_TYPES.push({
            id: i,
            name: carNames[i % carNames.length] + (i > 19 ? " v." + i : ""),
            color: `hsl(${(i * 137.5) % 360}, 75%, 50%)`,
            price: i === 0 ? 0 : (i * 150) + 200
        });
    }

    let active = false, lane = 2, playerX = 800, playerY = 780;
    let speed = 0, isNitro = false, lineOffset = 0;
    let enemies = [], level = 1, finishLineY = -5000;
    
    // Adjusted LANES for smaller cars
    const LANES = [450, 625, 800, 975, 1150];

    let cash = parseInt(localStorage.getItem('lambo_cash')) || 0;
    let unlocked = JSON.parse(localStorage.getItem('lambo_unlocked_ids')) || [0];
    let selectedCarIndex = 0;

    function updateCashUI() {
        document.getElementById('ui-stats').innerText = `CASH: $${cash} | LVL: ${level}`;
        document.getElementById('garage-cash').innerText = `CASH: $${cash}`;
        localStorage.setItem('lambo_cash', cash);
        renderGarage();
    }

    function renderGarage() {
        const list = document.getElementById('garage-list');
        list.innerHTML = '';
        CAR_TYPES.forEach((car, i) => {
            const isUnlocked = unlocked.includes(i);
            const isActive = selectedCarIndex === i;
            const div = document.createElement('div');
            div.className = 'car-row';
            div.innerHTML = `
                <div style="display:flex; align-items:center;">
                    <div style="width:20px; height:20px; background:${car.color}; margin-right:15px; border-radius:2px; border:1px solid #fff"></div>
                    <div><strong>${car.name}</strong><br><small>${isUnlocked ? "Owned" : "$"+car.price}</small></div>
                </div>
                <button class="btn ${isActive ? 'active' : (isUnlocked ? 'owned' : '')}" onclick="handleCarPurchase(${i})">
                    ${isActive ? 'SELECTED' : (isUnlocked ? 'SELECT' : 'BUY')}
                </button>`;
            list.appendChild(div);
        });
    }

    function handleCarPurchase(i) {
        if (unlocked.includes(i)) { selectedCarIndex = i; } 
        else if (cash >= CAR_TYPES[i].price) {
            cash -= CAR_TYPES[i].price; unlocked.push(i); selectedCarIndex = i;
            localStorage.setItem('lambo_unlocked_ids', JSON.stringify(unlocked));
        }
        updateCashUI();
    }

    function openGarage() { document.getElementById('ui-menu').style.display='none'; document.getElementById('garage-menu').style.display='flex'; updateCashUI(); }
    function closeGarage() { document.getElementById('garage-menu').style.display='none'; document.getElementById('ui-menu').style.display='flex'; }

    // SMALLER CAR DRAWING
    function drawLambo(x, y, color, isPlayer, nitro) {
        ctx.save(); ctx.translate(x, y);
        const s = 0.6; // Scale factor
        if (isPlayer && nitro) {
            ctx.fillStyle = "#00f2ff"; ctx.shadowBlur = 15;
            ctx.fillRect(-15, 45, 10, 30+Math.random()*30); ctx.fillRect(5, 45, 10, 30+Math.random()*30);
            ctx.shadowBlur = 0;
        }
        ctx.fillStyle = color;
        ctx.beginPath(); 
        ctx.moveTo(-30*s, 80*s); ctx.lineTo(-25*s, -90*s); 
        ctx.lineTo(25*s, -90*s); ctx.lineTo(30*s, 80*s);
        ctx.closePath(); ctx.fill();
        ctx.fillStyle = "#000"; ctx.fillRect(-18*s, -40*s, 36*s, 50*s);
        ctx.fillStyle = "#ff0000"; ctx.fillRect(-25*s, 75*s, 50*s, 5*s);
        ctx.restore();
    }

    function startGame() {
        active = true; speed = 18; lane = 2; playerX = LANES[2];
        enemies = []; level = 1; finishLineY = -5000;
        document.getElementById('ui-menu').style.display = 'none';
        update();
    }

    function update() {
        if (!active) return;
        let currentSpeed = isNitro ? speed * 1.8 : speed;
        finishLineY += currentSpeed;
        lineOffset = (lineOffset + currentSpeed) % 100;

        ctx.fillStyle = "#050505"; ctx.fillRect(0,0,1600,900);
        ctx.fillStyle = "#111"; ctx.fillRect(350, 0, 900, 900); // Tighter road for smaller cars

        ctx.strokeStyle = "rgba(255,255,255,0.3)"; ctx.setLineDash([30, 50]); ctx.lineDashOffset = -lineOffset;
        [537, 712, 887, 1062].forEach(x => { ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, 900); ctx.stroke(); });

        if (finishLineY > -200 && finishLineY < 1200) {
            ctx.fillStyle = "#fff";
            for(let i=0; i<9; i++) for(let j=0; j<2; j++) if((i+j)%2==0) ctx.fillRect(350+(i*100), finishLineY+(j*50), 100, 50);
        }

        if (finishLineY > 900) { level++; cash += 1000; finishLineY = -15000; updateCashUI(); }

        playerX += (LANES[lane] - playerX) * 0.2;
        speed += 0.002;

        if (Math.random() < 0.06) enemies.push({ x: LANES[Math.floor(Math.random()*5)], y: -200 });

        for (let i = enemies.length-1; i>=0; i--) {
            let e = enemies[i]; e.y += currentSpeed * 0.7;
            drawLambo(e.x, e.y, "#e74c3c", false, false);
            // Smaller collision hitboxes
            if (Math.abs(e.x - playerX) < 50 && Math.abs(e.y - playerY) < 80) {
                active = false; document.getElementById('ui-menu').style.display = 'flex';
            }
            if (e.y > 1000) { enemies.splice(i,1); cash += 10; updateCashUI(); }
        }

        drawLambo(playerX, playerY, CAR_TYPES[selectedCarIndex].color, true, isNitro);
        requestAnimationFrame(update);
    }

    window.onkeydown = (e) => {
        if ((e.key === "ArrowLeft" || e.key === "a") && lane > 0) lane--;
        if ((e.key === "ArrowRight" || e.key === "d") && lane < 4) lane++;
        if (e.key === "Shift") isNitro = true;
    };
    window.onkeyup = (e) => { if (e.key === "Shift") isNitro = false; };
    
    updateCashUI();
</script>
</body>
</html>

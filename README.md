<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: 100 Car Garage</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: 'Segoe UI', sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #050505; border: 3px solid #222; overflow: hidden; }
        canvas { width: 100%; height: 100%; display: block; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 100; }
        
        /* Scrollable Garage Styling */
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
        <p style="letter-spacing: 5px; margin-bottom: 20px;">100 CAR COLLECTION</p>
        <button class="btn" style="padding: 20px 60px; font-size: 24px;" onclick="startGame()">RACE</button>
        <button class="btn" style="margin-top: 10px;" onclick="openGarage()">GARAGE</button>
    </div>

    <div id="garage-menu" class="ui">
        <h1>THE GARAGE</h1>
        <p id="garage-cash" style="font-size: 24px;">CASH: $0</p>
        <div class="scroll-area" id="garage-list">
            </div>
        <button class="btn" style="background: #e74c3c; color: #fff;" onclick="closeGarage()">BACK TO MENU</button>
    </div>

    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d');
    
    // --- GENERATE 100 CARS ---
    const carNames = ["Countach", "Aventador", "Veneno", "Diablo", "Murciélago", "Gallardo", "Huracán", "Sian", "Terzo", "Reventón", "Centenario", "Miura", "Urus", "Sesto Elemento", "Egoista", "Jalpa", "Jarama", "Urraco", "Espada", "Islero"];
    const CAR_TYPES = [];
    for(let i=0; i<100; i++) {
        let baseName = carNames[i % carNames.length];
        let suffix = i > 19 ? " Mark " + (Math.floor(i/10)) : "";
        CAR_TYPES.push({
            id: i,
            name: baseName + suffix,
            color: `hsl(${(i * 137.5) % 360}, 70%, 50%)`, // Unique color for every car
            price: i === 0 ? 0 : Math.floor(i * 150) + 200
        });
    }

    // GAME STATE
    let active = false, lane = 2, playerX = 800, playerY = 750;
    let speed = 0, isNitro = false, lineOffset = 0;
    let enemies = [], level = 1, finishLineY = -5000;
    const LANES = [250, 525, 800, 1075, 1350];

    // STORAGE
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
                    <div style="width:30px; height:30px; background:${car.color}; margin-right:15px; border-radius:3px; border:2px solid #fff"></div>
                    <div>
                        <strong style="font-size:18px">${car.name}</strong><br>
                        <small style="color:#aaa">${isUnlocked ? "Owned" : "$"+car.price}</small>
                    </div>
                </div>
                <button class="btn ${isActive ? 'active' : (isUnlocked ? 'owned' : '')}" onclick="handleCarPurchase(${i})">
                    ${isActive ? 'SELECTED' : (isUnlocked ? 'SELECT' : 'BUY')}
                </button>
            `;
            list.appendChild(div);
        });
    }

    function handleCarPurchase(i) {
        if (unlocked.includes(i)) {
            selectedCarIndex = i;
        } else if (cash >= CAR_TYPES[i].price) {
            cash -= CAR_TYPES[i].price;
            unlocked.push(i);
            selectedCarIndex = i;
            localStorage.setItem('lambo_unlocked_ids', JSON.stringify(unlocked));
        }
        updateCashUI();
    }

    function openGarage() { document.getElementById('ui-menu').style.display = 'none'; document.getElementById('garage-menu').style.display = 'flex'; updateCashUI(); }
    function closeGarage() { document.getElementById('garage-menu').style.display = 'none'; document.getElementById('ui-menu').style.display = 'flex'; }

    function drawLambo(x, y, color, type, isPlayer, nitro) {
        ctx.save(); ctx.translate(x, y);
        if (isPlayer && nitro) {
            ctx.fillStyle = "#00f2ff"; ctx.shadowBlur = 20; ctx.fillRect(-35, 85, 20, 50+Math.random()*40); ctx.fillRect(15, 85, 20, 50+Math.random()*40); ctx.shadowBlur = 0;
        }
        ctx.fillStyle = color;
        ctx.beginPath(); ctx.moveTo(-55, 85); ctx.lineTo(-45, -90); ctx.lineTo(45, -90); ctx.lineTo(55, 85);
        ctx.closePath(); ctx.fill();
        ctx.fillStyle = "#000"; ctx.fillRect(-30, -35, 60, 45);
        ctx.fillStyle = "#ff0000"; ctx.fillRect(-50, 80, 100, 5);
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
        ctx.fillStyle = "#111"; ctx.fillRect(150, 0, 1300, 900);

        // Lines
        ctx.strokeStyle = "rgba(255,255,255,0.3)"; ctx.setLineDash([40, 60]); ctx.lineDashOffset = -lineOffset;
        [387, 662, 937, 1212].forEach(x => { ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, 900); ctx.stroke(); });

        // Finish Line
        if (finishLineY > -200 && finishLineY < 1200) {
            ctx.fillStyle = "#fff";
            for(let i=0; i<13; i++) for(let j=0; j<2; j++) if((i+j)%2==0) ctx.fillRect(150+(i*100), finishLineY+(j*50), 100, 50);
        }

        if (finishLineY > 900) { level++; cash += 1000; finishLineY = -15000; updateCashUI(); }

        playerX += (LANES[lane] - playerX) * 0.15;
        speed += 0.002;

        if (Math.random() < 0.05) enemies.push({ x: LANES[Math.floor(Math.random()*5)], y: -200 });

        for (let i = enemies.length-1; i>=0; i--) {
            let e = enemies[i]; e.y += currentSpeed * 0.7;
            drawLambo(e.x, e.y, "#e74c3c", 'Other', false, false);
            if (Math.abs(e.x - playerX) < 100 && Math.abs(e.y - playerY) < 150) {
                active = false; document.getElementById('ui-menu').style.display = 'flex';
            }
            if (e.y > 1000) { enemies.splice(i,1); cash += 10; updateCashUI(); }
        }

        drawLambo(playerX, playerY, CAR_TYPES[selectedCarIndex].color, CAR_TYPES[selectedCarIndex].name, true, isNitro);
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

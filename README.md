<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: Open Road</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: 'Segoe UI', sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #0a0a0a; border: 2px solid #333; overflow: hidden; }
        canvas { width: 100%; height: 100%; display: block; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.85); z-index: 100; backdrop-filter: blur(5px); }
        #garage-menu { padding: 20px; display: none; }
        .scroll-area { width: 85%; height: 65%; overflow-y: auto; background: rgba(0,0,0,0.5); padding: 15px; border-radius: 10px; margin: 20px 0; border: 1px solid #444; }
        .car-row { display: flex; justify-content: space-between; align-items: center; background: #151515; margin-bottom: 8px; padding: 12px 20px; border-radius: 4px; transition: 0.3s; }
        .car-row:hover { background: #222; }
        .btn { background: #f1c40f; color: #000; border: none; padding: 12px 24px; font-size: 16px; font-weight: bold; cursor: pointer; border-radius: 3px; transition: 0.2s; text-transform: uppercase; letter-spacing: 1px; }
        .btn:hover { background: #fff; transform: translateY(-2px); box-shadow: 0 5px 15px rgba(241, 196, 15, 0.4); }
        .btn.active { background: #3498db; color: #fff; border: 1px solid #fff; }
        .stats { position: absolute; top: 25px; left: 25px; font-size: 20px; color: #f1c40f; font-weight: 300; pointer-events: none; z-index: 10; letter-spacing: 2px; }
        h1 { font-style: italic; text-transform: uppercase; letter-spacing: 10px; margin: 0; }
    </style>
</head>
<body>

<div id="wrapper">
    <div class="stats" id="ui-stats">CASH: $0 | LVL: 1 | MAX MPH: 0</div>
    <div id="ui-menu" class="ui">
        <h1>Nitro Racer</h1>
        <p style="color: #666; margin-top: 5px;">OPEN HIGHWAY EDITION</p>
        <button class="btn" style="margin-top: 30px; width: 200px;" onclick="startGame()">Race</button>
        <button class="btn" style="margin-top: 10px; width: 200px; background: #333; color: #ccc;" onclick="openGarage()">Garage</button>
    </div>
    <div id="garage-menu" class="ui">
        <h1 style="font-size: 24px; letter-spacing: 5px;">Showroom</h1>
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

    const carNames = ["Aventador", "Huracán", "Veneno", "Sian", "Countach", "Murciélago", "Gallardo", "Reventón", "Centenario", "Miura"];
    const CAR_TYPES = [];
    for(let i=0; i<100; i++) {
        CAR_TYPES.push({
            id: i,
            name: carNames[i % carNames.length] + (i > 9 ? " Spec-"+i : ""),
            color: `hsl(${(i * 45) % 360}, 80%, 45%)`,
            price: i === 0 ? 0 : (i * 200) + 300
        });
    }

    let active = false, lane = 2, playerX = 800, playerY = 780;
    let speed = 0, isNitro = false, isBraking = false, isAccelerating = false, lineOffset = 0, shake = 0;
    let enemies = [], level = 1, finishLineY = -8000;
    const LANES = [450, 625, 800, 975, 1150];

    let cash = parseInt(localStorage.getItem('lambo_cash')) || 0;
    let unlocked = JSON.parse(localStorage.getItem('lambo_unlocked_ids')) || [0];
    let selectedCarIndex = 0;

    function updateCashUI() {
        const maxDisplay = isNitro ? (50 + (level * 3)) : (35 + (level * 2));
        document.getElementById('ui-stats').innerText = `CASH: $${cash} | LVL: ${level} | TOP: ${maxDisplay*5} MPH`;
        document.getElementById('garage-cash').innerText = `TOTAL FUNDS: $${cash}`;
        localStorage.setItem('lambo_cash', cash);
        renderGarage();
    }

    function renderGarage() {
        const list = document.getElementById('garage-list');
        list.innerHTML = '';
        CAR_TYPES.forEach((car, i) => {
            const isUnlocked = unlocked.includes(i);
            const div = document.createElement('div');
            div.className = 'car-row';
            div.innerHTML = `
                <div style="display:flex; align-items:center;">
                    <div style="width:15px; height:15px; background:${car.color}; margin-right:15px; border-radius:50%;"></div>
                    <strong>${car.name}</strong>
                </div>
                <button class="btn ${selectedCarIndex === i ? 'active' : ''}" onclick="handleCarPurchase(${i})">
                    ${selectedCarIndex === i ? 'Selected' : (isUnlocked ? 'Select' : '$'+car.price)}
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

    function drawRealisticCar(x, y, color, isPlayer, nitro, braking) {
        ctx.save();
        ctx.translate(x, y);
        ctx.fillStyle = "rgba(0,0,0,0.4)";
        ctx.beginPath(); ctx.ellipse(0, 5, 35, 65, 0, 0, Math.PI*2); ctx.fill();

        if (isPlayer && nitro && !braking && speed > 10) {
            const grad = ctx.createLinearGradient(0, 50, 0, 120);
            grad.addColorStop(0, '#fff'); grad.addColorStop(0.2, '#00d4ff'); grad.addColorStop(1, 'transparent');
            ctx.fillStyle = grad;
            ctx.fillRect(-22, 50, 12, 40+Math.random()*40); ctx.fillRect(10, 50, 12, 40+Math.random()*40);
        }

        ctx.fillStyle = color;
        ctx.beginPath();
        ctx.moveTo(-28, 55); ctx.bezierCurveTo(-32, 20, -32, -20, -25, -60);
        ctx.bezierCurveTo(-15, -75, 15, -75, 25, -60); ctx.bezierCurveTo(32, -20, 32, 20, 28, 55);
        ctx.closePath(); ctx.fill();

        const glassGrad = ctx.createLinearGradient(0, -30, 0, 10);
        glassGrad.addColorStop(0, '#111'); glassGrad.addColorStop(1, '#333');
        ctx.fillStyle = glassGrad;
        ctx.beginPath(); ctx.moveTo(-20, -25); ctx.lineTo(20, -25); ctx.lineTo(24, 10); ctx.lineTo(-24, 10); ctx.closePath(); ctx.fill();

        ctx.fillStyle = braking ? "#ff3300" : "#900";
        if (braking) { ctx.shadowBlur = 20; ctx.shadowColor = "#ff0000"; }
        ctx.fillRect(-26, 50, 15, 6); ctx.fillRect(11, 50, 15, 6);
        ctx.shadowBlur = 0;
        ctx.restore();
    }

    function startGame() {
        active = true; speed = 0; lane = 2; playerX = LANES[2];
        enemies = []; level = 1; finishLineY = -8000;
        document.getElementById('ui-menu').style.display = 'none';
        updateCashUI();
        update();
    }

    function update() {
        if (!active) return;
        
        const baseCap = 35 + (level * 2);
        const nitroCap = 50 + (level * 3);
        const currentMax = isNitro ? nitroCap : baseCap;

        if (isBraking) {
            speed -= 0.8;
        } else if (isAccelerating) {
            speed += 0.12;
            if (isNitro) speed += 0.25;
        } else {
            speed -= 0.04;
        }
        
        if (speed > currentMax) speed -= 0.15;
        if (speed < 0) speed = 0;

        shake = (isNitro && isAccelerating && speed > 25) ? Math.random() * (speed/8) - (speed/16) : 0;
        let currentSpeed = speed;
        finishLineY += currentSpeed;
        lineOffset = (lineOffset + currentSpeed) % 100;

        ctx.save();
        ctx.translate(shake, shake);

        ctx.fillStyle = "#151515"; ctx.fillRect(0,0,1600,900);
        ctx.fillStyle = "#1a1a1a"; ctx.fillRect(350, 0, 900, 900);

        ctx.strokeStyle = "rgba(255,255,255,0.08)"; ctx.setLineDash([40, 60]); ctx.lineDashOffset = -lineOffset;
        [537, 712, 887, 1062].forEach(x => { ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, 900); ctx.stroke(); });

        if (finishLineY > -200 && finishLineY < 1200) {
            ctx.fillStyle = "#fff";
            for(let i=0; i<9; i++) for(let j=0; j<2; j++) if((i+j)%2==0) ctx.fillRect(350+(i*100), finishLineY+(j*50), 100, 50);
        }

        if (finishLineY > 900) { 
            level++; cash += 1500; 
            finishLineY = -8000 - (level * 1500);
            updateCashUI(); 
        }

        playerX += (LANES[lane] - playerX) * 0.15;

        // LESS TRAFFIC LOGIC: Reduced spawn chance significantly
        if (speed > 5 && Math.random() < 0.02 + (level * 0.002)) {
            enemies.push({ x: LANES[Math.floor(Math.random()*5)], y: -200, color: `hsl(${Math.random()*360}, 10%, 35%)` });
        }

        for (let i = enemies.length-1; i>=0; i--) {
            let e = enemies[i]; 
            e.y += (currentSpeed * 0.6);
            drawRealisticCar(e.x, e.y, e.color, false, false, false);
            
            if (Math.abs(e.x - playerX) < 55 && Math.abs(e.y - playerY) < 100) {
                active = false; document.getElementById('ui-menu').style.display = 'flex';
            }
            if (e.y > 1000) { enemies.splice(i,1); cash += 10; updateCashUI(); }
        }

        drawRealisticCar(playerX, playerY, CAR_TYPES[selectedCarIndex].color, true, isNitro, isBraking);
        
        ctx.restore();
        requestAnimationFrame(update);
    }

    window.onkeydown = (e) => {
        if ((e.key === "ArrowLeft" || e.key === "a") && lane > 0) lane--;
        if ((e.key === "ArrowRight" || e.key === "d") && lane < 4) lane++;
        if (e.key === "ArrowUp" || e.key === "w") isAccelerating = true;
        if (e.key === "Shift") isNitro = true;
        if (e.key === " ") isBraking = true;
    };
    window.onkeyup = (e) => { 
        if (e.key === "ArrowUp" || e.key === "w") isAccelerating = false;
        if (e.key === "Shift") isNitro = false;
        if (e.key === " ") isBraking = false;
    };
    
    updateCashUI();
</script>
</body>
</html>

<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: The Ultimate Mix</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: 'Segoe UI', sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #020202; overflow: hidden; border: 2px solid #f1c40f; box-shadow: 0 0 20px #f1c40f55; }
        canvas { width: 100%; height: 100%; display: block; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 100; backdrop-filter: blur(10px); }
        .stats { position: absolute; top: 20px; left: 20px; font-size: 18px; color: #f1c40f; z-index: 10; font-weight: bold; }
        #health-container { width: 200px; height: 12px; background: #222; position: absolute; top: 55px; left: 20px; border: 1px solid #444; border-radius: 6px; overflow: hidden; }
        #health-fill { width: 100%; height: 100%; background: #2ecc71; transition: 0.2s; }
        .btn { background: #f1c40f; color: #000; border: none; padding: 12px 25px; font-weight: bold; cursor: pointer; margin: 10px; border-radius: 4px; text-transform: uppercase; transition: 0.2s; }
        .btn:hover { background: #fff; transform: scale(1.05); }
        .garage-win { width: 85%; height: 80%; background: #111; border: 1px solid #f1c40f; border-radius: 12px; padding: 20px; display: flex; flex-direction: column; }
        #garage-list { flex-grow: 1; overflow-y: auto; margin-top: 15px; }
        .car-row { display: flex; justify-content: space-between; padding: 10px; border-bottom: 1px solid #222; align-items: center; }
    </style>
</head>
<body>

<div id="wrapper">
    <div class="stats" id="ui-stats">CASH: $41,205 | 0 KPH</div>
    <div id="health-container"><div id="health-fill"></div></div>
    
    <div id="main-menu" class="ui">
        <h1 style="color:#f1c40f; letter-spacing: 10px; font-size: 50px; margin: 0;">NITRO MIX</h1>
        <p style="color: #666;">300+ LUXURY SHAPES • NIGHT PURSUIT</p>
        <button class="btn" onclick="startGame()">Start Engine</button>
        <button class="btn" style="background:#333; color:#fff;" onclick="toggleGarage(true)">Showroom</button>
    </div>

    <div id="garage-menu" class="ui" style="display:none;">
        <div class="garage-win">
            <input type="text" id="search" placeholder="Search 300+ models..." oninput="renderGarage()" style="width:100%; padding:12px; background:#000; border:1px solid #444; color:#fff;">
            <div id="garage-list"></div>
            <button class="btn" onclick="toggleGarage(false)">Back to Main</button>
        </div>
    </div>
    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d');
    canvas.width = 1280; canvas.height = 720;

    // Database & Logic
    const BRANDS = ["Lambo", "Ferrari", "Bugatti", "Bentley", "Porsche", "Rolls", "Mclaren", "Aston"];
    const SHAPES = ["wedge", "oval", "teardrop"];
    const COLORS = ["#f1c40f", "#e74c3c", "#3498db", "#9b59b6", "#2ecc71", "#00d4ff", "#ff00ff"];
    const CAR_DB = [];
    for(let i=0; i<305; i++) {
        CAR_DB.push({
            name: `${BRANDS[i%BRANDS.length]} Elite #${i+1}`,
            color: COLORS[i%COLORS.length],
            shape: SHAPES[i%SHAPES.length],
            price: i * 1500,
            id: i
        });
    }

    let active = false, lane = 2, playerX = 640, playerY = 600, speed = 0;
    let isBraking = false, isAccelerating = false, health = 100, shake = 0;
    let items = [], cash = 41205, selectedIdx = 0, lineOffset = 0;
    const LANES = [350, 495, 640, 785, 930];

    function toggleGarage(show) {
        document.getElementById('main-menu').style.display = show ? 'none' : 'flex';
        document.getElementById('garage-menu').style.display = show ? 'flex' : 'none';
        if(show) renderGarage();
    }

    function renderGarage() {
        const query = document.getElementById('search').value.toLowerCase();
        const list = document.getElementById('garage-list');
        list.innerHTML = CAR_DB.filter(c => c.name.toLowerCase().includes(query)).map(c => `
            <div class="car-row">
                <span><b>${c.name}</b> (${c.shape})</span>
                <button class="btn" style="background:${selectedIdx==c.id?'#3498db':'#222'}; color:#fff" onclick="selectedIdx=${c.id}">
                    ${selectedIdx==c.id ? 'SELECTED' : '$'+c.price}
                </button>
            </div>
        `).join('');
    }

    function drawObject(x, y, car, type, braking) {
        ctx.save();
        ctx.translate(x, y);

        if(type === 'player') {
            let tilt = (LANES[lane] - x) * 0.05;
            ctx.rotate(tilt * Math.PI / 180);
        }

        // Underglow
        ctx.shadowBlur = 20;
        ctx.shadowColor = (type === 'police' && Math.floor(Date.now()/100)%2) ? "blue" : car.color;
        
        // Headlights
        ctx.fillStyle = "rgba(255,255,255,0.2)";
        ctx.beginPath(); ctx.moveTo(-20, -40); ctx.lineTo(-40, -150); ctx.lineTo(40, -150); ctx.lineTo(20, -40); ctx.fill();

        // Shape Rendering
        ctx.fillStyle = (type === 'police') ? "#111" : car.color;
        if (car.shape === "wedge") {
            ctx.beginPath(); ctx.moveTo(-28, 48); ctx.lineTo(28, 48); ctx.lineTo(22, -48); ctx.lineTo(-22, -48); ctx.closePath(); ctx.fill();
        } else if (car.shape === "oval") {
            ctx.beginPath(); ctx.ellipse(0, 0, 30, 50, 0, 0, Math.PI*2); ctx.fill();
        } else {
            ctx.beginPath(); ctx.moveTo(0, -50); ctx.bezierCurveTo(35, -20, 35, 40, 0, 50); ctx.bezierCurveTo(-35, 40, -35, -20, 0, -50); ctx.fill();
        }

        // Taillights
        ctx.fillStyle = braking ? "#ff0000" : "#500";
        if(braking) ctx.shadowColor = "red", ctx.shadowBlur = 15;
        ctx.fillRect(-20, 40, 10, 5); ctx.fillRect(10, 40, 10, 5);

        ctx.restore();
    }

    function startGame() { 
        active = true; speed = 0; health = 100; items = []; 
        document.getElementById('main-menu').style.display = 'none';
        requestAnimationFrame(update); 
    }

    function update() {
        if (!active) return;
        if (isBraking) speed -= 0.8; else if (isAccelerating) speed += 0.2; else speed -= 0.05;
        speed = Math.max(0, Math.min(speed, 60));
        lineOffset = (lineOffset + speed) % 100;
        if (shake > 0) shake -= 0.1;

        ctx.save();
        ctx.translate(Math.random()*shake*8, Math.random()*shake*8);
        
        // Background
        ctx.fillStyle = "#020202"; ctx.fillRect(0,0,1280,720);
        ctx.fillStyle = "#080808"; ctx.fillRect(280, 0, 720, 720);
        ctx.strokeStyle = "#1a1a1a"; ctx.setLineDash([40,60]); ctx.lineDashOffset = -lineOffset;
        ctx.beginPath(); [422, 567, 712, 857].forEach(lx => { ctx.moveTo(lx,0); ctx.lineTo(lx,720); }); ctx.stroke();

        // Spawn logic (Cars, Cops, Repair)
        if (Math.random() < 0.015) {
            let roll = Math.random();
            let type = roll < 0.05 ? 'repair' : (roll < 0.2 && speed > 40 ? 'police' : 'traffic');
            items.push({
                x: LANES[Math.floor(Math.random()*5)],
                y: type === 'police' ? 800 : -200,
                type: type,
                bSpeed: type === 'police' ? speed + 5 : 15 + Math.random()*20,
                model: CAR_DB[Math.floor(Math.random()*CAR_DB.length)]
            });
        }

        for (let i = items.length - 1; i >= 0; i--) {
            let o = items[i]; o.y += (speed - o.bSpeed);
            
            if(o.type === 'repair') {
                ctx.fillStyle = "#2ecc71"; ctx.font = "30px Arial"; ctx.fillText("🔧", o.x-15, o.y+10);
            } else {
                drawObject(o.x, o.y, o.model, o.type, false);
            }

            // Hit Detection
            if (Math.abs(o.x - playerX) < 45 && Math.abs(o.y - playerY) < 80) {
                if(o.type === 'repair') { health = Math.min(100, health+30); items.splice(i, 1); }
                else { shake = 2; health -= 15; speed *= 0.5; items.splice(i, 1); }
            }
            if (o.y > 1100 || o.y < -900) items.splice(i, 1);
        }

        // Movement Fix (No leak)
        playerX += (LANES[lane] - playerX) * 0.25;
        drawObject(playerX, playerY, CAR_DB[selectedIdx], 'player', isBraking);
        ctx.restore();

        // UI
        document.getElementById('health-fill').style.width = health + "%";
        document.getElementById('health-fill').style.background = health < 30 ? "#e74c3c" : "#2ecc71";
        document.getElementById('ui-stats').innerText = `CASH: $${cash.toLocaleString()} | ${Math.round(speed*7.5)} KPH`;

        if (health <= 0) { active = false; document.getElementById('main-menu').style.display='flex'; }
        requestAnimationFrame(update);
    }

    // Controls (WASD + Arrows)
    window.onkeydown = (e) => {
        const k = e.key.toLowerCase();
        if ((k === "a" || k === "arrowleft") && lane > 0) lane--;
        if ((k === "d" || k === "arrowright") && lane < 4) lane++;
        if (k === "w" || k === "arrowup") isAccelerating = true;
        if (k === "s" || k === "arrowdown" || k === " ") isBraking = true;
    };
    window.onkeyup = (e) => {
        const k = e.key.toLowerCase();
        if (k === "w" || k === "arrowup") isAccelerating = false;
        if (k === "s" || k === "arrowdown" || k === " ") isBraking = false;
    };
</script>
</body>
</html>

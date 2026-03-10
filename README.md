<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: The Master Build</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: 'Segoe UI', sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #020202; overflow: hidden; border: 2px solid #f1c40f; box-shadow: 0 0 30px rgba(241, 196, 15, 0.2); }
        canvas { width: 100%; height: 100%; display: block; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 100; backdrop-filter: blur(15px); }
        .stats { position: absolute; top: 20px; left: 20px; font-size: 18px; color: #f1c40f; z-index: 10; font-weight: bold; text-shadow: 0 0 10px #f1c40f; }
        #health-bar { width: 200px; height: 10px; background: #333; position: absolute; top: 50px; left: 20px; border: 1px solid #fff; border-radius: 5px; overflow: hidden; }
        #health-fill { width: 100%; height: 100%; background: #2ecc71; transition: 0.2s; }
        #status-alert { position: absolute; top: 70px; left: 20px; color: #e74c3c; font-weight: bold; display: none; }
        .btn { background: #f1c40f; color: #000; border: none; padding: 12px 25px; font-weight: bold; cursor: pointer; margin: 10px; border-radius: 4px; text-transform: uppercase; transition: 0.2s; }
        .btn:hover { background: #fff; box-shadow: 0 0 15px #fff; }
        .garage-win { width: 80%; height: 80%; background: #0a0a0a; border: 1px solid #f1c40f; padding: 20px; border-radius: 10px; display: flex; flex-direction: column; }
        #garage-scroll { flex-grow: 1; overflow-y: auto; margin: 15px 0; }
        .car-item { display: flex; justify-content: space-between; padding: 10px; border-bottom: 1px solid #222; align-items: center; }
    </style>
</head>
<body>

<div id="wrapper">
    <div class="stats" id="ui-stats">CASH: $41,205 | 0 KPH</div>
    <div id="health-bar"><div id="health-fill"></div></div>
    <div id="status-alert">⚠️ TIRE BLOWN: STEERING IMPAIRED</div>
    
    <div id="ui-menu" class="ui">
        <h1 style="letter-spacing:10px; color:#f1c40f; margin-bottom: 0;">NITRO MASTER</h1>
        <p style="color: #666; margin-bottom: 20px;">VIP PURSUIT EDITION</p>
        <button class="btn" onclick="startGame()">Ignition</button>
        <button class="btn" style="background:#222; color:#ccc;" onclick="openGarage()">Showroom</button>
    </div>

    <div id="garage-menu" class="ui" style="display:none;">
        <div class="garage-win">
            <input type="text" id="g-search" placeholder="Search 300+ Cars..." oninput="renderGarage()" style="width:100%; padding:10px; background:#111; border:1px solid #f1c40f; color:#fff;">
            <div id="garage-scroll"></div>
            <button class="btn" onclick="closeGarage()">Exit</button>
        </div>
    </div>
    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d');
    canvas.width = 1280; canvas.height = 720;

    // --- DATA ---
    const SHAPES = ["wedge", "oval", "teardrop"];
    const BRANDS = ["Lambo", "GT-R", "Ferrari", "Bugatti", "Bentley", "Porsche", "Rolls", "Mclaren"];
    const COLORS = ["#f1c40f", "#e74c3c", "#3498db", "#9b59b6", "#2ecc71", "#00d4ff", "#ff00ff"];
    const CAR_DB = [];
    for(let i=0; i<305; i++) {
        CAR_DB.push({ name: `${BRANDS[i%BRANDS.length]} v${i+1}`, color: COLORS[i%COLORS.length], shape: SHAPES[i%SHAPES.length], price: i*1200, id: i });
    }

    // --- STATE ---
    let active = false, lane = 2, playerX = 640, playerY = 600, speed = 0;
    let isBraking = false, isAccelerating = false, health = 100, shake = 0;
    let items = [], cash = 41205, selectedIdx = 0, lineOffset = 0;
    let steerSpeed = 0.25, flatTire = false;
    const LANES = [350, 495, 640, 785, 930];

    function openGarage() { document.getElementById('ui-menu').style.display='none'; document.getElementById('garage-menu').style.display='flex'; renderGarage(); }
    function closeGarage() { document.getElementById('garage-menu').style.display='none'; document.getElementById('ui-menu').style.display='flex'; }
    function renderGarage() {
        const term = document.getElementById('g-search').value.toLowerCase();
        document.getElementById('garage-scroll').innerHTML = CAR_DB.filter(c => c.name.toLowerCase().includes(term)).map(c => `
            <div class="car-item">
                <span><b>${c.name}</b> (${c.shape})</span>
                <button class="btn" style="background:${selectedIdx==c.id?'#3498db':'#333'}; color:#fff" onclick="selectedIdx=${c.id}; renderGarage();">
                    ${selectedIdx==c.id ? 'ACTIVE' : '$'+c.price}
                </button>
            </div>
        `).join('');
    }

    function drawBody(x, y, car, type, braking) {
        ctx.save();
        ctx.translate(x, y);

        // Neon Underglow
        ctx.shadowBlur = 20;
        ctx.shadowColor = (type === 'police' && Math.floor(Date.now()/100)%2) ? (Math.random()>0.5?"red":"blue") : car.color;

        // Headlights
        ctx.fillStyle = "rgba(255,255,255,0.15)";
        ctx.beginPath(); ctx.moveTo(-20,-40); ctx.lineTo(-45,-180); ctx.lineTo(45,-180); ctx.lineTo(20,-40); ctx.fill();

        // Chassis
        ctx.fillStyle = (type === 'police') ? "#0a0a0a" : car.color;
        if (car.shape === "wedge") {
            ctx.beginPath(); ctx.moveTo(-28,45); ctx.lineTo(28,45); ctx.lineTo(22,-45); ctx.lineTo(-22,-45); ctx.closePath(); ctx.fill();
        } else if (car.shape === "oval") {
            ctx.beginPath(); ctx.ellipse(0,0,30,50,0,0,Math.PI*2); ctx.fill();
        } else {
            ctx.beginPath(); ctx.moveTo(0,-50); ctx.bezierCurveTo(38,-20, 38,40, 0,50); ctx.bezierCurveTo(-38,40, -38,-20, 0,-50); ctx.fill();
        }

        // Brakelights
        if (braking) { ctx.shadowColor = "red"; ctx.shadowBlur = 25; ctx.fillStyle = "red"; ctx.fillRect(-25, 38, 12, 6); ctx.fillRect(13, 38, 12, 6); }
        
        // Police Lights
        if (type === 'police') {
            ctx.fillStyle = Math.floor(Date.now()/100)%2 ? "red" : "blue";
            ctx.fillRect(-18, -5, 36, 10);
        }
        
        // Sparks for Flat Tire
        if (type === 'player' && flatTire && Math.random()>0.7) {
            ctx.fillStyle = "orange"; ctx.fillRect(-35 + Math.random()*70, 40, 4, 4);
        }

        ctx.restore();
    }

    function startGame() { 
        active = true; speed = 0; health = 100; items = []; flatTire = false; steerSpeed = 0.25;
        document.getElementById('ui-menu').style.display = 'none';
        document.getElementById('status-alert').style.display = 'none';
        requestAnimationFrame(update); 
    }

    function update() {
        if (!active) return;

        if (isBraking) speed -= 0.8; else if (isAccelerating) speed += 0.22; else speed -= 0.05;
        speed = Math.max(0, Math.min(speed, 65));
        lineOffset = (lineOffset + speed) % 100;
        if (shake > 0) shake -= 0.1;

        ctx.save();
        ctx.translate(Math.random()*shake*10, Math.random()*shake*10);
        ctx.fillStyle = "#020202"; ctx.fillRect(0,0,1280,720);
        ctx.fillStyle = "#080808"; ctx.fillRect(280, 0, 720, 720); // Road
        
        // Lines
        ctx.strokeStyle = "#1a1a1a"; ctx.setLineDash([40,60]); ctx.lineDashOffset = -lineOffset;
        ctx.beginPath(); [422, 567, 712, 857].forEach(lx => { ctx.moveTo(lx,0); ctx.lineTo(lx,720); }); ctx.stroke();

        // Spawn
        if (Math.random() < 0.015) {
            let roll = Math.random();
            let type = roll < 0.06 ? 'repair' : (roll < 0.22 && speed > 40 ? 'police' : 'traffic');
            items.push({
                x: LANES[Math.floor(Math.random()*5)],
                y: type === 'police' ? 850 : -250,
                type: type,
                bSpeed: type === 'police' ? speed + 12 : 12 + Math.random()*20,
                isBraking: false,
                car: type === 'police' ? {shape:'wedge', color:'#111'} : CAR_DB[Math.floor(Math.random()*CAR_DB.length)]
            });
        }

        for (let i = items.length - 1; i >= 0; i--) {
            let o = items[i];
            
            // AI Behavior
            if (o.type === 'police') {
                if (o.y < playerY && o.y > playerY - 400 && o.x === LANES[lane]) {
                    if (Math.random() < 0.01 && !o.isBraking) items.push({ x: o.x, y: o.y + 60, type: 'spike', bSpeed: 0 });
                    if (Math.random() < 0.02) o.isBraking = true;
                }
                if (o.isBraking) o.bSpeed *= 0.93;
            }
            
            o.y += (speed - o.bSpeed);

            // Render
            if (o.type === 'spike') {
                ctx.fillStyle = "#444"; ctx.fillRect(o.x-60, o.y-5, 120, 10);
                ctx.fillStyle = "#f1c40f"; for(let s=0; s<6; s++) ctx.fillRect(o.x-50+(s*20), o.y-5, 5, 10);
            } else if (o.type === 'repair') {
                ctx.shadowBlur=15; ctx.shadowColor="#2ecc71"; ctx.fillStyle="#2ecc71"; ctx.font="35px Arial"; ctx.fillText("🔧", o.x-15, o.y+10);
            } else {
                drawBody(o.x, o.y, o.car, o.type, o.isBraking);
            }

            // Collisions
            if (Math.abs(o.x - playerX) < 50 && Math.abs(o.y - playerY) < 65) {
                if (o.type === 'spike') { flatTire = true; steerSpeed = 0.06; items.splice(i, 1); document.getElementById('status-alert').style.display='block'; }
                else if (o.type === 'repair') { health = Math.min(100, health+40); flatTire = false; steerSpeed = 0.25; items.splice(i, 1); document.getElementById('status-alert').style.display='none'; }
                else { shake = 3; health -= 15; speed *= 0.4; items.splice(i, 1); }
            }
            if (o.y > 1100 || o.y < -1100) items.splice(i, 1);
        }

        // Final Steering
        playerX += (LANES[lane] - playerX) * steerSpeed;
        drawBody(playerX, playerY, CAR_DB[selectedIdx], 'player', isBraking);
        ctx.restore();

        // UI Update
        document.getElementById('health-fill').style.width = health + "%";
        document.getElementById('health-fill').style.background = health < 30 ? "#e74c3c" : "#2ecc71";
        document.getElementById('ui-stats').innerText = `CASH: $${cash.toLocaleString()} | ${Math.round(speed*7.5)} KPH`;

        if (health <= 0) { active = false; document.getElementById('ui-menu').style.display='flex'; }
        requestAnimationFrame(update);
    }

    // Input Handling
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

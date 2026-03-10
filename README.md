<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: Pursuit & Physics</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: 'Segoe UI', sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #020202; overflow: hidden; border: 2px solid #f1c40f; }
        canvas { width: 100%; height: 100%; display: block; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 100; backdrop-filter: blur(15px); }
        .stats { position: absolute; top: 20px; left: 20px; font-size: 18px; color: #f1c40f; z-index: 10; font-weight: bold; }
        #health-bar { width: 200px; height: 10px; background: #333; position: absolute; top: 50px; left: 20px; border: 1px solid #fff; }
        #health-fill { width: 100%; height: 100%; background: #e74c3c; transition: 0.3s; }
        .btn { background: #f1c40f; color: #000; border: none; padding: 12px 20px; font-weight: bold; cursor: pointer; margin: 5px; border-radius: 4px; }
    </style>
</head>
<body>

<div id="wrapper">
    <div class="stats" id="ui-stats">CASH: $0 | 0 KPH</div>
    <div id="health-bar"><div id="health-fill"></div></div>
    
    <div id="ui-menu" class="ui">
        <h1 style="letter-spacing:10px; color:#f1c40f; text-shadow: 0 0 15px red;">PURSUIT MODE</h1>
        <button class="btn" onclick="startGame()">EVADE</button>
        <button class="btn" style="background:#222; color:#fff;" onclick="openGarage()">GARAGE</button>
    </div>

    <div id="garage-menu" class="ui" style="display:none">
        <div style="width: 80%; background:#111; padding:20px; border-radius:10px;">
            <h2>GARAGE (WASD TO NAVIGATE)</h2>
            <div id="garage-list" style="height:300px; overflow-y:auto;"></div>
            <button class="btn" onclick="closeGarage()">BACK</button>
        </div>
    </div>
    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d');
    canvas.width = 1280; canvas.height = 720;

    const brands = ["Lambo", "GT-R", "Skyline", "Ferrari", "Bugatti", "Police"];
    const colors = ["#f1c40f", "#3498db", "#e74c3c", "#9b59b6", "#2ecc71"];
    const CAR_DATABASE = [];
    for(let i=0; i<305; i++) {
        CAR_DATABASE.push({ name: brands[i % 5] + " Spec-" + i, color: colors[i % 5], price: i*500, w: 58, h: 95 });
    }

    let active = false, lane = 2, playerX = 640, playerY = 600, speed = 0;
    let isBraking = false, isAccelerating = false, health = 100, shake = 0;
    let enemies = [], cash = 5000, selectedIdx = 0, lineOffset = 0;
    const LANES = [350, 495, 640, 785, 930];

    function openGarage() { document.getElementById('ui-menu').style.display='none'; document.getElementById('garage-menu').style.display='flex'; renderGarage(); }
    function closeGarage() { document.getElementById('garage-menu').style.display='none'; document.getElementById('ui-menu').style.display='flex'; }
    function renderGarage() {
        const list = document.getElementById('garage-list');
        list.innerHTML = CAR_DATABASE.slice(0, 20).map((c, i) => `
            <div style="padding:10px; border-bottom:1px solid #333; display:flex; justify-content:space-between;">
                <span>${c.name}</span>
                <button onclick="selectedIdx=${i}; closeGarage();">SELECT</button>
            </div>
        `).join('');
    }

    function drawVehicle(x, y, car, isPolice, braking) {
        ctx.save();
        ctx.translate(x, y);
        
        // Neon Glow
        ctx.shadowBlur = 15;
        ctx.shadowColor = isPolice ? (Math.floor(Date.now()/100)%2 ? "red" : "blue") : car.color;
        
        // Body
        ctx.fillStyle = isPolice ? "#111" : car.color;
        ctx.beginPath(); ctx.roundRect(-car.w/2, -car.h/2, car.w, car.h, 15); ctx.fill();
        
        // Police Lights
        if(isPolice) {
            ctx.fillStyle = Math.floor(Date.now()/100)%2 ? "red" : "blue";
            ctx.fillRect(-car.w/2, -10, car.w, 10);
        }

        // Taillights
        ctx.fillStyle = braking ? "red" : "#400";
        ctx.fillRect(-car.w/2+5, car.h/2-8, 10, 5); ctx.fillRect(car.w/2-15, car.h/2-8, 10, 5);
        ctx.restore();
    }

    function startGame() { 
        active = true; speed = 0; health = 100; enemies = []; 
        document.getElementById('ui-menu').style.display='none'; 
        requestAnimationFrame(update); 
    }

    function update() {
        if (!active) return;

        // Physics
        if (isBraking) speed -= 0.8; else if (isAccelerating) speed += 0.2; else speed -= 0.05;
        speed = Math.max(0, Math.min(speed, 65));
        lineOffset = (lineOffset + speed) % 100;

        // Screen Shake
        if (shake > 0) shake -= 0.1;
        ctx.save();
        ctx.translate(Math.random()*shake*10, Math.random()*shake*10);

        // Background
        ctx.fillStyle = "#050505"; ctx.fillRect(-50,-50,1380,820);
        ctx.fillStyle = "#111"; ctx.fillRect(280, 0, 720, 720);
        ctx.strokeStyle = "#222"; ctx.setLineDash([40,60]); ctx.lineDashOffset = -lineOffset;
        ctx.beginPath(); [422, 567, 712, 857].forEach(lx => { ctx.moveTo(lx,0); ctx.lineTo(lx,720); }); ctx.stroke();

        // Spawn logic
        if (Math.random() < 0.02) {
            let isCop = speed > 40 && Math.random() < 0.3;
            enemies.push({ 
                x: LANES[Math.floor(Math.random()*5)], 
                y: isCop ? 800 : -200, 
                isCop: isCop,
                bSpeed: isCop ? speed + 5 : 15 + Math.random()*20, 
                model: CAR_DATABASE[Math.floor(Math.random()*5)] 
            });
        }

        for (let i = enemies.length - 1; i >= 0; i--) {
            let e = enemies[i];
            e.y += e.isCop ? (speed - e.bSpeed) : (speed - e.bSpeed);
            drawVehicle(e.x, e.y, e.model, e.isCop, false);

            // Crash Physics
            if (Math.abs(e.x - playerX) < 50 && Math.abs(e.y - playerY) < 90) {
                shake = 2;
                health -= 10;
                speed *= 0.5;
                enemies.splice(i, 1);
                if (health <= 0) {
                    active = false;
                    document.getElementById('ui-menu').style.display='flex';
                }
            }
            if (e.y > 1100 || e.y < -900) { if(e.y > 1100) cash += 50; enemies.splice(i, 1); }
        }

        playerX += (LANES[lane] - playerX) * 0.25;
        drawVehicle(playerX, playerY, CAR_DATABASE[selectedIdx], false, isBraking);
        
        ctx.restore();
        
        document.getElementById('health-fill').style.width = health + "%";
        document.getElementById('ui-stats').innerText = `CASH: $${cash} | ${Math.round(speed*7)} KPH`;
        requestAnimationFrame(update);
    }

    window.onkeydown = (e) => {
        const key = e.key.toLowerCase();
        if ((key === "a" || key === "arrowleft") && lane > 0) lane--;
        if ((key === "d" || key === "arrowright") && lane < 4) lane++;
        if (key === "w" || key === "arrowup") isAccelerating = true;
        if (key === "s" || key === " " || key === "arrowdown") isBraking = true;
    };
    window.onkeyup = (e) => {
        const key = e.key.toLowerCase();
        if (key === "w" || key === "arrowup") isAccelerating = false;
        if (key === "s" || key === " " || key === "arrowdown") isBraking = false;
    };
</script>
</body>
</html>

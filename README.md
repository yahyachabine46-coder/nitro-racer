<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: Precision Steering</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #0a0a0a; overflow: hidden; border: 2px solid #444; }
        canvas { width: 100%; height: 100%; display: block; image-rendering: optimizeSpeed; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 100; backdrop-filter: blur(5px); }
        #garage-menu { display: none; }
        .stats { position: absolute; top: 20px; left: 20px; font-size: 18px; color: #f1c40f; z-index: 10; font-weight: bold; }
        .btn { background: #f1c40f; color: #000; border: none; padding: 12px 25px; font-weight: bold; cursor: pointer; margin: 8px; min-width: 150px; }
        .btn:hover { background: #fff; }
        .scroll-area { width: 80%; height: 50%; overflow-y: auto; background: #111; margin: 15px 0; }
        .row { display: flex; justify-content: space-between; padding: 10px 20px; border-bottom: 1px solid #222; }
    </style>
</head>
<body>

<div id="wrapper">
    <div class="stats" id="ui-stats">CASH: $0 | LVL: 1</div>
    
    <div id="ui-menu" class="ui">
        <h1 style="letter-spacing:8px; color:#f1c40f;">NITRO RACER</h1>
        <p style="color:#888;">STEERING RECALIBRATED</p>
        <button class="btn" onclick="startGame()">RACE</button>
        <button class="btn" style="background:#333; color:#ccc;" onclick="openGarage()">GARAGE</button>
    </div>

    <div id="garage-menu" class="ui">
        <h1>GARAGE</h1>
        <div class="scroll-area" id="garage-list"></div>
        <button class="btn" onclick="closeGarage()">BACK</button>
    </div>

    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d', { alpha: false });
    canvas.width = 1280; canvas.height = 720;

    const TRAFFIC_MODELS = [
        { type: 'coupe', w: 42, h: 80, color: '#5a5a5a' },
        { type: 'sedan', w: 48, h: 90, color: '#444' },
        { type: 'suv', w: 58, h: 100, color: '#3a3a3a' },
        { type: 'long_truck', w: 54, h: 220, color: '#222' }, // Extra long truck
        { type: 'van', w: 56, h: 110, color: '#666' }
    ];

    const PLAYER_CARS = [
        { name: "Superlight", color: "#f1c40f" },
        { name: "GT Red", color: "#e74c3c" },
        { name: "Midnight", color: "#2c3e50" }
    ];

    let active = false, lane = 2, playerX = 640, playerY = 600;
    let speed = 0, isBraking = false, isAccelerating = false, lineOffset = 0;
    let enemies = [], level = 1, cash = parseInt(localStorage.getItem('lambo_cash')) || 0;
    let selectedCar = 0;
    const LANES = [350, 495, 640, 785, 930];

    function openGarage() { document.getElementById('ui-menu').style.display='none'; document.getElementById('garage-menu').style.display='flex'; updateGarage(); }
    function closeGarage() { document.getElementById('garage-menu').style.display='none'; document.getElementById('ui-menu').style.display='flex'; }

    function updateGarage() {
        const list = document.getElementById('garage-list');
        list.innerHTML = '';
        PLAYER_CARS.forEach((c, i) => {
            list.innerHTML += `<div class="row"><span>${c.name}</span><button class="btn" onclick="selectedCar=${i};closeGarage();">${selectedCar==i?'SELECTED':'SELECT'}</button></div>`;
        });
    }

    function drawVehicle(x, y, model, color, braking) {
        ctx.save();
        ctx.translate(x, y);

        // Body
        ctx.fillStyle = color;
        ctx.fillRect(-model.w/2, -model.h/2, model.w, model.h);

        // Windshield
        ctx.fillStyle = "rgba(0,0,0,0.7)";
        ctx.fillRect(-model.w/2.5, -model.h/3, model.w/1.25, model.h/4);

        // Headlights
        ctx.fillStyle = "#fff";
        ctx.fillRect(-model.w/2, -model.h/2, 8, 3);
        ctx.fillRect(model.w/2 - 8, -model.h/2, 8, 3);

        // Taillights
        ctx.fillStyle = braking ? "#ff0000" : "#800";
        ctx.fillRect(-model.w/2, model.h/2 - 4, 12, 4);
        ctx.fillRect(model.w/2 - 12, model.h/2 - 4, 12, 4);

        ctx.restore();
    }

    function startGame() {
        active = true; speed = 0; enemies = []; lane = 2; playerX = LANES[2];
        document.getElementById('ui-menu').style.display = 'none';
        requestAnimationFrame(update);
    }

    function update() {
        if (!active) return;

        // Corrected Physics
        if (isBraking) speed -= 0.7;
        else if (isAccelerating) speed += 0.09;
        else speed -= 0.03;
        speed = Math.max(0, Math.min(speed, 35 + level));
        lineOffset = (lineOffset + speed) % 100;

        // Background / Road
        ctx.fillStyle = "#1a1a1a"; ctx.fillRect(0, 0, 1280, 720);
        ctx.fillStyle = "#333"; ctx.fillRect(280, 0, 720, 720);
        ctx.strokeStyle = "#555"; ctx.setLineDash([40, 60]); ctx.lineDashOffset = -lineOffset;
        ctx.beginPath(); [422, 567, 712, 857].forEach(lx => { ctx.moveTo(lx, 0); ctx.lineTo(lx, 720); }); ctx.stroke();

        // Spawn logic
        if (Math.random() < 0.012) {
            const template = TRAFFIC_MODELS[Math.floor(Math.random()*TRAFFIC_MODELS.length)];
            enemies.push({ 
                x: LANES[Math.floor(Math.random()*5)], 
                y: -300, 
                bSpeed: 10 + Math.random()*8,
                model: template
            });
        }

        for (let i = enemies.length - 1; i >= 0; i--) {
            let e = enemies[i];
            e.y += (speed - e.bSpeed);
            drawVehicle(e.x, e.y, e.model, e.model.color, false);

            // Tight collision check
            if (Math.abs(e.x - playerX) < (e.model.w/2 + 20) && Math.abs(e.y - playerY) < (e.model.h/2 + 40)) {
                active = false;
                document.getElementById('ui-menu').style.display = 'flex';
                localStorage.setItem('lambo_cash', cash);
            }
            if (e.y > 1000 || e.y < -600) enemies.splice(i, 1);
            if (e.y > 1000) cash += 20;
        }

        // FIXED STEERING: Direct interpolation to target lane with no drift
        const targetX = LANES[lane];
        const drift = targetX - playerX;
        if (Math.abs(drift) > 1) {
            playerX += drift * 0.15;
        } else {
            playerX = targetX; // Snap to center
        }

        drawVehicle(playerX, playerY, {w:46, h:90}, PLAYER_CARS[selectedCar].color, isBraking);

        document.getElementById('ui-stats').innerText = `CASH: $${cash} | LVL: ${level} | ${Math.round(speed*5)} KPH`;
        requestAnimationFrame(update);
    }

    // Input with debounce to prevent lane-skipping
    window.onkeydown = (e) => {
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

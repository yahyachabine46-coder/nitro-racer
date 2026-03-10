<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: Spike Strips</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: 'Segoe UI', sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #020202; overflow: hidden; border: 2px solid #e74c3c; }
        canvas { width: 100%; height: 100%; display: block; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.9); z-index: 100; backdrop-filter: blur(10px); }
        .stats { position: absolute; top: 20px; left: 20px; font-size: 18px; color: #f1c40f; z-index: 10; font-weight: bold; }
        #health-container { width: 200px; height: 12px; background: #222; position: absolute; top: 55px; left: 20px; border: 1px solid #444; border-radius: 6px; overflow: hidden; }
        #health-fill { width: 100%; height: 100%; background: #2ecc71; transition: 0.1s; }
        #status-msg { position: absolute; top: 80px; left: 20px; color: #e74c3c; font-weight: bold; display: none; text-shadow: 0 0 5px #000; }
        .btn { background: #e74c3c; color: #fff; border: none; padding: 12px 25px; font-weight: bold; cursor: pointer; border-radius: 4px; text-transform: uppercase; }
    </style>
</head>
<body>

<div id="wrapper">
    <div class="stats" id="ui-stats">CASH: $41,205 | 0 KPH</div>
    <div id="health-container"><div id="health-fill"></div></div>
    <div id="status-msg">TIRE BLOWN - STEERING IMPAIRED</div>
    
    <div id="main-menu" class="ui">
        <h1 style="color:#e74c3c; letter-spacing: 10px; font-size: 50px; margin: 0;">SPIKE STRIP PURSUIT</h1>
        <button class="btn" onclick="startGame()">Start Evading</button>
    </div>
    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d');
    canvas.width = 1280; canvas.height = 720;

    const LANES = [350, 495, 640, 785, 930];
    let active = false, lane = 2, playerX = 640, playerY = 600, speed = 0;
    let isBraking = false, isAccelerating = false, health = 100, shake = 0;
    let items = [], cash = 41205, lineOffset = 0;
    let steeringSpeed = 0.25, tireBlown = false;

    function drawObject(x, y, color, type, braking) {
        ctx.save();
        ctx.translate(x, y);

        if (type === 'spike') {
            ctx.fillStyle = "#333";
            ctx.fillRect(-60, -5, 120, 10);
            ctx.fillStyle = "#f1c40f";
            for(let i=0; i<6; i++) ctx.fillRect(-50 + (i*20), -5, 5, 10);
        } else if (type === 'repair') {
            ctx.fillStyle = "#2ecc71"; ctx.font = "30px Arial"; ctx.fillText("🔧", -15, 15);
        } else {
            // Car Body
            ctx.shadowBlur = 15;
            ctx.shadowColor = (type === 'police' && Math.floor(Date.now()/100)%2) ? "blue" : color;
            ctx.fillStyle = (type === 'police') ? "#111" : color;
            ctx.beginPath(); ctx.roundRect(-28, -48, 56, 96, 15); ctx.fill();
            
            if (braking) {
                ctx.shadowColor = "red"; ctx.shadowBlur = 20;
                ctx.fillStyle = "#ff0000"; ctx.fillRect(-25, 38, 12, 6); ctx.fillRect(13, 38, 12, 6);
            }
            if (type === 'police') {
                ctx.fillStyle = Math.floor(Date.now()/100)%2 ? "red" : "blue";
                ctx.fillRect(-15, -5, 30, 8);
            }
            if (type === 'player' && tireBlown) {
                ctx.fillStyle = "rgba(255,165,0,0.5)";
                if(Math.random() > 0.5) ctx.fillRect(-35, 30, 10, 10); // Spark FX
            }
        }
        ctx.restore();
    }

    function startGame() { 
        active = true; speed = 0; health = 100; items = []; tireBlown = false; steeringSpeed = 0.25;
        document.getElementById('status-msg').style.display = 'none';
        document.getElementById('main-menu').style.display = 'none';
        requestAnimationFrame(update); 
    }

    function update() {
        if (!active) return;
        if (isBraking) speed -= 0.8; else if (isAccelerating) speed += 0.2; else speed -= 0.05;
        speed = Math.max(0, Math.min(speed, 65));
        lineOffset = (lineOffset + speed) % 100;
        if (shake > 0) shake -= 0.1;

        ctx.save();
        ctx.translate(Math.random()*shake*10, Math.random()*shake*10);
        ctx.fillStyle = "#050505"; ctx.fillRect(0,0,1280,720);
        ctx.fillStyle = "#111"; ctx.fillRect(280, 0, 720, 720);
        
        ctx.strokeStyle = "#222"; ctx.setLineDash([40,60]); ctx.lineDashOffset = -lineOffset;
        ctx.beginPath(); [422, 567, 712, 857].forEach(lx => { ctx.moveTo(lx,0); ctx.lineTo(lx,720); }); ctx.stroke();

        // Spawn logic
        if (Math.random() < 0.015) {
            let roll = Math.random();
            let type = roll < 0.05 ? 'repair' : (roll < 0.2 && speed > 35 ? 'police' : 'traffic');
            items.push({
                x: LANES[Math.floor(Math.random()*5)],
                y: type === 'police' ? 800 : -200,
                type: type,
                bSpeed: type === 'police' ? speed + 12 : 15 + Math.random()*20,
                isBrakeChecking: false,
                color: type === 'police' ? "#111" : "#3498db"
            });
        }

        for (let i = items.length - 1; i >= 0; i--) {
            let o = items[i];
            
            if (o.type === 'police') {
                if (o.y < playerY && o.y > playerY - 400 && o.x === LANES[lane]) {
                    if (Math.random() < 0.01 && !o.isBrakeChecking) {
                        items.push({ x: o.x, y: o.y + 60, type: 'spike', bSpeed: 0 }); // Drop spikes
                    }
                    if (Math.random() < 0.02) o.isBrakeChecking = true;
                }
                if (o.isBrakeChecking) o.bSpeed *= 0.93;
            }
            
            o.y += (speed - o.bSpeed);
            drawObject(o.x, o.y, o.color, o.type, o.isBrakeChecking);

            // Collisions
            if (Math.abs(o.x - playerX) < 50 && Math.abs(o.y - playerY) < 60) {
                if (o.type === 'spike') {
                    tireBlown = true; steeringSpeed = 0.06;
                    document.getElementById('status-msg').style.display = 'block';
                    items.splice(i, 1);
                } else if (o.type === 'repair') {
                    health = Math.min(100, health+30); tireBlown = false; steeringSpeed = 0.25;
                    document.getElementById('status-msg').style.display = 'none';
                    items.splice(i, 1);
                } else {
                    shake = 3; health -= 15; speed *= 0.4; items.splice(i, 1);
                }
            }
            if (o.y > 1100 || o.y < -900) items.splice(i, 1);
        }

        playerX += (LANES[lane] - playerX) * steeringSpeed;
        drawObject(playerX, playerY, "#f1c40f", "player", isBraking);
        ctx.restore();

        document.getElementById('health-fill').style.width = health + "%";
        document.getElementById('ui-stats').innerText = `CASH: $${cash} | ${Math.round(speed*7.5)} KPH`;

        if (health <= 0) { active = false; document.getElementById('main-menu').style.display='flex'; }
        requestAnimationFrame(update);
    }

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

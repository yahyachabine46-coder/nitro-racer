<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer: Unique Shapes</title>
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; color: #fff; font-family: sans-serif; }
        #wrapper { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 95vw; height: 53.4vw; max-height: 100vh; max-width: 177vh; background: #050505; border: 2px solid #f1c40f; overflow: hidden; }
        canvas { width: 100%; height: 100%; display: block; }
        .stats { position: absolute; top: 20px; left: 20px; font-size: 18px; color: #f1c40f; z-index: 10; font-weight: bold; }
        .ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.8); z-index: 100; }
        .btn { background: #f1c40f; color: #000; border: none; padding: 15px 30px; font-weight: bold; cursor: pointer; border-radius: 5px; text-transform: uppercase; }
    </style>
</head>
<body>

<div id="wrapper">
    <div class="stats" id="ui-stats">CASH: $41205 | 0 KPH</div>
    <div id="ui-menu" class="ui">
        <h1 style="color:#f1c40f; letter-spacing: 5px;">SHAPE SHIFTER RACING</h1>
        <button class="btn" onclick="startGame()">Drive Now</button>
    </div>
    <canvas id="stage"></canvas>
</div>

<script>
    const canvas = document.getElementById('stage');
    const ctx = canvas.getContext('2d');
    canvas.width = 1280; canvas.height = 720;

    // Archetypes for different car shapes
    const SHAPES = ["wedge", "oval", "teardrop"];
    const COLORS = ["#f1c40f", "#e74c3c", "#3498db", "#9b59b6", "#2ecc71"];

    let active = false, lane = 2, playerX = 640, playerY = 600, speed = 0;
    let isBraking = false, isAccelerating = false, lineOffset = 0, enemies = [];
    let cash = 41205; 
    const LANES = [350, 495, 640, 785, 930];

    function drawCustomCar(x, y, color, shape, isPlayer) {
        ctx.save();
        ctx.translate(x, y);
        
        // Steering Tilt (Visual only)
        if (isPlayer) {
            let tilt = (LANES[lane] - x) * 0.05;
            ctx.rotate(tilt * Math.PI / 180);
        }

        ctx.fillStyle = color;
        ctx.shadowBlur = 15;
        ctx.shadowColor = color;

        if (shape === "wedge") {
            // Sharp Sport Shape
            ctx.beginPath();
            ctx.moveTo(-25, 45); ctx.lineTo(25, 45);
            ctx.lineTo(20, -45); ctx.lineTo(-20, -45);
            ctx.closePath(); ctx.fill();
        } else if (shape === "oval") {
            // Rounded Luxury Shape
            ctx.beginPath();
            ctx.ellipse(0, 0, 28, 48, 0, 0, Math.PI * 2);
            ctx.fill();
        } else {
            // Teardrop Aero Shape
            ctx.beginPath();
            ctx.moveTo(0, -50);
            ctx.bezierCurveTo(30, -20, 30, 40, 0, 50);
            ctx.bezierCurveTo(-30, 40, -30, -20, 0, -50);
            ctx.fill();
        }

        // Cockpit Window
        ctx.fillStyle = "rgba(0,0,0,0.6)";
        ctx.roundRect(-15, -10, 30, 25, 5); ctx.fill();
        
        ctx.restore();
    }

    function startGame() { active = true; speed = 0; enemies = []; lane = 2; playerX = LANES[2]; document.getElementById('ui-menu').style.display='none'; requestAnimationFrame(update); }

    function update() {
        if (!active) return;
        if (isBraking) speed -= 0.6; else if (isAccelerating) speed += 0.15; else speed -= 0.05;
        speed = Math.max(0, Math.min(speed, 55));
        lineOffset = (lineOffset + speed) % 100;

        ctx.fillStyle = "#111"; ctx.fillRect(0,0,1280,720);
        ctx.fillStyle = "#181818"; ctx.fillRect(280, 0, 720, 720);
        
        // Lane Lines
        ctx.strokeStyle = "#333"; ctx.setLineDash([40,60]); ctx.lineDashOffset = -lineOffset;
        ctx.beginPath(); [422, 567, 712, 857].forEach(lx => { ctx.moveTo(lx,0); ctx.lineTo(lx,720); }); ctx.stroke();

        // LOWERED Traffic Spawn (0.01 instead of 0.02)
        if (Math.random() < 0.01) {
            enemies.push({ 
                x: LANES[Math.floor(Math.random()*5)], 
                y: -200, 
                bSpeed: 10 + Math.random()*20, 
                color: COLORS[Math.floor(Math.random()*COLORS.length)],
                shape: SHAPES[Math.floor(Math.random()*SHAPES.length)]
            });
        }

        for (let i = enemies.length - 1; i >= 0; i--) {
            let e = enemies[i]; e.y += (speed - e.bSpeed);
            drawCustomCar(e.x, e.y, e.color, e.shape, false);
            
            // Collision
            if (Math.abs(e.x - playerX) < 40 && Math.abs(e.y - playerY) < 80) { 
                active = false; document.getElementById('ui-menu').style.display='flex'; 
            }
            if (e.y > 1000 || e.y < -800) enemies.splice(i, 1);
        }

        // FIXED STEERING: No leak, precise snapping
        playerX += (LANES[lane] - playerX) * 0.2;

        drawCustomCar(playerX, playerY, "#f1c40f", "teardrop", true);
        document.getElementById('ui-stats').innerText = `CASH: $${cash} | ${Math.round(speed*7)} KPH`;
        requestAnimationFrame(update);
    }

    window.onkeydown = (e) => {
        const key = e.key.toLowerCase();
        if ((key === "a" || key === "arrowleft") && lane > 0) lane--;
        if ((key === "d" || key === "arrowright") && lane < 4) lane++;
        if (key === "w" || key === "arrowup") isAccelerating = true;
        if (key === "s" || key === "arrowdown" || key === " ") isBraking = true;
    };
    window.onkeyup = (e) => {
        const key = e.key.toLowerCase();
        if (key === "w" || key === "arrowup") isAccelerating = false;
        if (key === "s" || key === "arrowdown" || key === " ") isBraking = false;
    };
</script>
</body>
</html>

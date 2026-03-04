<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Nitro Racer - Fixed</title>
    <style>
        body { background: #000; color: #fff; font-family: sans-serif; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; overflow: hidden; }
        #game-container { position: relative; width: 360px; height: 600px; border: 2px solid #333; border-radius: 20px; background: #050505; }
        canvas { display: block; border-radius: 18px; cursor: crosshair; }
        #overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.9); display: flex; flex-direction: column; align-items: center; justify-content: center; z-index: 10; border-radius: 18px; }
        .garage { background: #151515; padding: 20px; border-radius: 15px; width: 80%; margin-bottom: 20px; border: 1px solid #222; }
        h1 { color: #00f2ff; text-shadow: 0 0 10px #00f2ff; margin: 0 0 20px 0; font-size: 24px; }
        label { display: block; font-size: 10px; color: #888; margin-top: 10px; text-transform: uppercase; }
        input, select { width: 100%; margin-top: 5px; background: #222; border: 1px solid #444; color: #fff; padding: 5px; }
        #startBtn { background: #ff00ff; color: #fff; border: none; padding: 15px 40px; border-radius: 50px; font-weight: bold; font-size: 18px; cursor: pointer; box-shadow: 0 0 20px #ff00ff; margin-top: 10px; }
        #startBtn:hover { transform: scale(1.05); }
    </style>
</head>
<body>

<div id="game-container">
    <canvas id="raceCanvas" width="360" height="600"></canvas>
    
    <div id="overlay">
        <h1>NITRO GARAGE</h1>
        <div class="garage">
            <label>Paint Job</label>
            <input type="color" id="carColor" value="#00aaff">
            
            <label>Body Style</label>
            <select id="decalStyle">
                <option value="none">Standard</option>
                <option value="carbon">Carbon Fiber</option>
                <option value="stripes">Racing Stripes</option>
            </select>

            <label>Underglow Intensity</label>
            <input type="range" id="glowPower" min="0" max="40" value="20">
        </div>
        <button id="startBtn">START ENGINE</button>
        <p style="font-size: 10px; color: #444; margin-top: 15px;">ARROWS OR CLICK SIDES TO STEER</p>
    </div>
</div>

<script>
    const canvas = document.getElementById('raceCanvas');
    const ctx = canvas.getContext('2d');
    const overlay = document.getElementById('overlay');
    const startBtn = document.getElementById('startBtn');
    
    // Config
    const LANES = [70, 180, 290];
    let player = { lane: 1, x: 180, color: '#00aaff' };
    let enemies = [];
    let score = 0;
    let gameActive = false;
    let speed = 5;
    let frame = 0;

    function drawCar(x, y, color, isPlayer) {
        ctx.save();
        if (isPlayer) {
            ctx.shadowBlur = document.getElementById('glowPower').value;
            ctx.shadowColor = color;
        }
        
        // Body
        ctx.fillStyle = color;
        ctx.beginPath();
        ctx.roundRect(x - 25, y, 50, 90, 10);
        ctx.fill();
        ctx.shadowBlur = 0;

        // Windows
        ctx.fillStyle = "rgba(0,0,0,0.6)";
        ctx.fillRect(x - 18, y + 15, 36, 25);
        
        // Decals
        if (isPlayer && document.getElementById('decalStyle').value === 'stripes') {
            ctx.fillStyle = "rgba(255,255,255,0.3)";
            ctx.fillRect(x - 12, y, 6, 90); ctx.fillRect(x + 6, y, 6, 90);
        }

        ctx.restore();
    }

    function update() {
        if (!gameActive) return;
        frame++;

        // Spawn enemies
        if (frame % 60 === 0) {
            enemies.push({ x: LANES[Math.floor(Math.random() * 3)], y: -100 });
        }

        enemies.forEach((en, i) => {
            en.y += speed;
            // Collision Check
            if (en.x === LANES[player.lane] && en.y + 80 > 480 && en.y < 570) {
                gameActive = false;
                overlay.style.display = 'flex';
                document.querySelector('h1').innerText = "WASTED!";
                startBtn.innerText = "RETRY";
            }
            if (en.y > 650) {
                enemies.splice(i, 1);
                score++;
                if (score % 5 === 0) speed += 0.5;
            }
        });

        draw();
        requestAnimationFrame(update);
    }

    function draw() {
        ctx.fillStyle = '#050505';
        ctx.fillRect(0, 0, 360, 600);

        // Lines
        ctx.strokeStyle = '#222';
        ctx.setLineDash([20, 20]);
        [125, 235].forEach(lx => {
            ctx.beginPath(); ctx.moveTo(lx, 0); ctx.lineTo(lx, 600); ctx.stroke();
        });

        // Player & Enemies
        drawCar(LANES[player.lane], 480, document.getElementById('carColor').value, true);
        enemies.forEach(en => drawCar(en.x, en.y, '#555', false));

        // Score
        ctx.setLineDash([]);
        ctx.fillStyle = '#00f2ff';
        ctx.font = 'bold 16px Arial';
        ctx.fillText("SCORE: " + score, 20, 40);
    }

    // Input
    window.addEventListener('keydown', e => {
        if (e.key === "ArrowLeft" && player.lane > 0) player.lane--;
        if (e.key === "ArrowRight" && player.lane < 2) player.lane++;
    });

    canvas.addEventListener('click', e => {
        const rect = canvas.getBoundingClientRect();
        const x = e.clientX - rect.left;
        if (x < 180 && player.lane > 0) player.lane--;
        else if (x > 180 && player.lane < 2) player.lane++;
    });

    startBtn.addEventListener('click', () => {
        gameActive = true;
        score = 0;
        speed = 5;
        enemies = [];
        player.lane = 1;
        overlay.style.display = 'none';
        update();
    });

    // Run initial frame
    draw();
</script>
</body>
</html>

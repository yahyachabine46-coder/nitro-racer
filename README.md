<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Nitro Racer Wide</title>
    <style>
        body { background: #000; margin: 0; overflow: hidden; font-family: sans-serif; color: #fff; }
        #ui { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; background: rgba(0,0,0,0.8); z-index: 10; }
        canvas { display: block; width: 100vw; height: auto; aspect-ratio: 16/9; background: #111; }
        button { background: #00f2ff; color: #000; border: none; padding: 15px 40px; font-size: 20px; font-weight: bold; cursor: pointer; border-radius: 5px; box-shadow: 0 0 20px #00f2ff; }
        .stats { position: absolute; top: 20px; left: 20px; z-index: 5; font-size: 24px; color: #00f2ff; pointer-events: none; }
    </style>
</head>
<body>

<div class="stats" id="stats">STAGE 1 | 0.0s</div>
<div id="ui">
    <h1 style="font-size: 48px; text-shadow: 0 0 20px #00f2ff;">NITRO RACER WIDE</h1>
    <button onclick="startGame()">START RACE</button>
    <p style="margin-top: 20px; color: #888;">USE ARROWS OR CLICK SIDES TO STEER</p>
</div>
<canvas id="g"></canvas>

<script>
    const canvas = document.getElementById('g');
    const ctx = canvas.getContext('2d');
    const ui = document.getElementById('ui');
    const statBox = document.getElementById('stats');

    // Set internal resolution
    canvas.width = 1600;
    canvas.height = 900;

    let active = false;
    let score = 0;
    let lane = 2;
    let playerX = 800;
    let velocity = 5;
    let startTime = 0;
    let enemies = [];
    const LANES = [200, 500, 800, 1100, 1400];

    function drawCar(x, y, color) {
        ctx.fillStyle = color;
        // Shadow/Glow
        ctx.shadowBlur = 15;
        ctx.shadowColor = color;
        // Body
        ctx.fillRect(x - 50, y - 80, 100, 160);
        // Cockpit
        ctx.fillStyle = "rgba(0,0,0,0.7)";
        ctx.fillRect(x - 35, y - 20, 70, 50);
        ctx.shadowBlur = 0;
    }

    function startGame() {
        active = true;
        score = 0;
        velocity = 8;
        enemies = [];
        startTime = Date.now();
        ui.style.display = 'none';
        loop();
    }

    function loop() {
        if (!active) return;

        // 1. Clear & Background
        ctx.fillStyle = "#050505";
        ctx.fillRect(0, 0, 1600, 900);

        // 2. Road & Lines
        ctx.fillStyle = "#111";
        ctx.fillRect(100, 0, 1400, 900);
        ctx.strokeStyle = "#333";
        ctx.setLineDash([30, 30]);
        for(let i = 1; i < 5; i++) {
            ctx.beginPath();
            ctx.moveTo(100 + (i * 280), 0);
            ctx.lineTo(100 + (i * 280), 900);
            ctx.stroke();
        }

        // 3. Update Stats
        let time = ((Date.now() - startTime) / 1000).toFixed(1);
        statBox.innerText = `SCORE: ${score} | ${time}s`;

        // 4. Player Physics
        let targetX = LANES[lane];
        playerX += (targetX - playerX) * 0.15;
        velocity += 0.005;
        if (velocity > 25) velocity = 25;

        // 5. Enemies
        if (Math.random() < 0.03) {
            enemies.push({ x: LANES[Math.floor(Math.random() * 5)], y: -200 });
        }

        for (let i = enemies.length - 1; i >= 0; i--) {
            let e = enemies[i];
            e.y += velocity;
            drawCar(e.x, e.y, "#ff3300");

            // Collision
            if (Math.abs(e.x - playerX) < 90 && Math.abs(e.y - 750) < 140) {
                active = false;
                ui.style.display = 'flex';
                alert("CRASHED! Score: " + score);
            }

            if (e.y > 1000) {
                enemies.splice(i, 1);
                score++;
            }
        }

        // 6. Draw Player
        drawCar(playerX, 750, "#00f2ff");

        requestAnimationFrame(loop);
    }

    // Input
    window.addEventListener('keydown', e => {
        if (e.key === "ArrowLeft" && lane > 0) lane--;
        if (e.key === "ArrowRight" && lane < 4) lane--; // Typo fix: lane++
        if (e.key === "ArrowRight" && lane < 4) lane++; 
    });

    canvas.addEventListener('mousedown', e => {
        const rect = canvas.getBoundingClientRect();
        const x = (e.clientX - rect.left) / rect.width;
        if (x < 0.5 && lane > 0) lane--;
        else if (x > 0.5 && lane < 4) lane++;
    });
</script>
</body>
</html>

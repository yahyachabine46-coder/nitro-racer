<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    
    <title>Nitro Racer Pro | High-Speed Arcade</title>
    <meta name="description" content="Play Nitro Racer Pro. A high-speed neon car game. Dodge traffic and break records.">
    
    <style>
        :root { --neon-blue: #00f3ff; --neon-pink: #ff00ff; --bg: #0a0a0c; }
        body { 
            margin: 0; background: var(--bg); color: white; 
            font-family: 'Segoe UI', sans-serif; display: flex; 
            flex-direction: column; align-items: center; overflow: hidden; 
        }
        
        #game-container { 
            position: relative; width: 360px; height: 640px; 
            margin-top: 20px; border: 4px solid #222; border-radius: 15px;
            box-shadow: 0 0 50px rgba(0,0,0,0.8); overflow: hidden;
        }
        
        canvas { display: block; background: #111; width: 100%; height: 100%; }

        .ui-overlay {
            position: absolute; inset: 0; background: rgba(0,0,0,0.8);
            display: flex; flex-direction: column; justify-content: center;
            align-items: center; z-index: 100;
        }

        .hud {
            position: absolute; top: 15px; left: 15px; right: 15px;
            display: flex; justify-content: space-between; pointer-events: none;
            font-family: monospace; font-size: 1.2rem; color: var(--neon-blue);
            text-shadow: 0 0 10px var(--neon-blue);
        }

        button {
            padding: 15px 40px; font-size: 1.2rem; font-weight: bold;
            background: var(--neon-pink); color: white; border: none;
            border-radius: 50px; cursor: pointer; box-shadow: 0 0 20px var(--neon-pink);
            transition: 0.2s; text-transform: uppercase;
        }
        button:hover { transform: scale(1.1); }

        .controls-hint { margin-top: 20px; color: #666; font-size: 0.8rem; }
        
        .mobile-btn-container {
            display: flex; gap: 40px; margin-top: 20px;
        }
        .touch-zone {
            width: 80px; height: 80px; border: 2px solid var(--neon-blue);
            border-radius: 50%; display: flex; align-items: center; 
            justify-content: center; font-size: 2rem; color: var(--neon-blue);
            user-select: none; touch-action: none;
        }
    </style>
</head>
<body>

<div id="game-container">
    <div class="hud">
        <div>SCORE: <span id="score">0</span></div>
        <div>BEST: <span id="best">0</span></div>
    </div>

    <div id="menu" class="ui-overlay">
        <h1 style="color:var(--neon-blue); letter-spacing:5px;">NITRO RACER</h1>
        <button onclick="startGame()">Start Engine</button>
        <div class="controls-hint">ARROWS or TOUCH to drive</div>
    </div>

    <canvas id="gameCanvas"></canvas>
</div>

<div class="mobile-btn-container">
    <div class="touch-zone" id="lTouch">◀</div>
    <div class="touch-zone" id="rTouch">▶</div>
</div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    canvas.width = 360;
    canvas.height = 640;

    let active = false;
    let score = 0;
    let speed = 7;
    let highScore = localStorage.getItem('nitro_top') || 0;
    document.getElementById('best').innerText = highScore;

    const player = { x: 155, y: 520, w: 50, h: 95, color: '#00f3ff' };
    let enemies = [];
    let roadOffset = 0;

    // Movement state
    let moveLeft = false;
    let moveRight = false;

    // Keyboard
    window.onkeydown = (e) => {
        if(e.code === 'ArrowLeft') moveLeft = true;
        if(e.code === 'ArrowRight') moveRight = true;
    };
    window.onkeyup = (e) => {
        if(e.code === 'ArrowLeft') moveLeft = false;
        if(e.code === 'ArrowRight') moveRight = false;
    };

    // Mobile Touch
    const lT = document.getElementById('lTouch');
    const rT = document.getElementById('rTouch');
    lT.ontouchstart = (e) => { e.preventDefault(); moveLeft = true; };
    lT.ontouchend = () => moveLeft = false;
    rT.ontouchstart = (e) => { e.preventDefault(); moveRight = true; };
    rT.ontouchend = () => moveRight = false;

    function drawCar(car, color, isPlayer) {
        ctx.save();
        // Body Glow
        ctx.shadowBlur = 15;
        ctx.shadowColor = color;
        
        // Main Chassis
        ctx.fillStyle = color;
        ctx.beginPath();
        // Modern rounded shape
        const r = 10;
        ctx.moveTo(car.x + r, car.y);
        ctx.lineTo(car.x + car.w - r, car.y);
        ctx.quadraticCurveTo(car.x + car.w, car.y, car.x + car.w, car.y + r);
        ctx.lineTo(car.x + car.w, car.y + car.h - r);
        ctx.quadraticCurveTo(car.x + car.w, car.y + car.h, car.x + car.w - r, car.y + car.h);
        ctx.lineTo(car.x + r, car.y + car.h);
        ctx.quadraticCurveTo(car.x, car.y + car.h, car.x, car.y + car.h - r);
        ctx.lineTo(car.x, car.y + r);
        ctx.quadraticCurveTo(car.x, car.y, car.x + r, car.y);
        ctx.fill();

        // Cockpit (Windows)
        ctx.shadowBlur = 0;
        ctx.fillStyle = "#050505";
        ctx.fillRect(car.x + 8, car.y + 25, car.w - 16, 30);
        
        // Windshield shine
        ctx.fillStyle = "rgba(255,255,255,0.1)";
        ctx.fillRect(car.x + 10, car.y + 27, car.w - 20, 5);

        // Headlights / Taillights
        ctx.fillStyle = isPlayer ? "#fff" : "#ff0044";
        const lightY = isPlayer ? car.y : car.y + car.h - 5;
        ctx.fillRect(car.x + 5, lightY, 10, 5);
        ctx.fillRect(car.x + car.w - 15, lightY, 10, 5);
        
        ctx.restore();
    }

    function startGame() {
        document.getElementById('menu').style.display = 'none';
        active = true;
        score = 0;
        speed = 7;
        enemies = [];
        player.x = 155;
        update();
    }

    function update() {
        if(!active) return;

        if(moveLeft && player.x > 10) player.x -= 9;
        if(moveRight && player.x < canvas.width - player.w - 10) player.x += 9;

        roadOffset = (roadOffset + speed) % 100;

        for(let i = enemies.length - 1; i >= 0; i--) {
            enemies[i].y += speed;

            // Collision Detection
            if (player.x < enemies[i].x + enemies[i].w &&
                player.x + player.w > enemies[i].x &&
                player.y < enemies[i].y + enemies[i].h &&
                player.y + player.h > enemies[i].y) {
                gameOver();
            }

            if(enemies[i].y > canvas.height) {
                enemies.splice(i, 1);
                score++;
                document.getElementById('score').innerText = score;
                if(score % 10 === 0) speed += 0.4;
            }
        }

        if(Math.random() < 0.02) {
            const lane = Math.floor(Math.random() * 3);
            enemies.push({ x: lane * 120 + 35, y: -100, w: 50, h: 95, color: '#ff00ff' });
        }

        render();
        requestAnimationFrame(update);
    }

    function render() {
        ctx.fillStyle = '#0a0a0c';
        ctx.fillRect(0, 0, canvas.width, canvas.height);

        // Road markings
        ctx.setLineDash([40, 60]);
        ctx.lineDashOffset = -roadOffset;
        ctx.strokeStyle = '#222';
        ctx.lineWidth = 4;
        ctx.beginPath();
        ctx.moveTo(120, 0); ctx.lineTo(120, 640);
        ctx.moveTo(240, 0); ctx.lineTo(240, 640);
        ctx.stroke();

        drawCar(player, player.color, true);
        enemies.forEach(en => drawCar(en, en.color, false));
    }

    function gameOver() {
        active = false;
        if(score > highScore) {
            highScore = score;
            localStorage.setItem('nitro_top', highScore);
        }
        document.getElementById('best').innerText = highScore;
        document.getElementById('menu').style.display = 'flex';
        document.querySelector('h1').innerText = "CRASHED!";
    }

    render();
</script>

</body>
</html>

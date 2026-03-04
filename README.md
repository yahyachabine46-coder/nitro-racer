<!DOCTYPE html>
<html>
<head>
    <title>Nitro Racer: Day & Night</title>
    <style>
        body {
            background-color: #111;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            font-family: 'Segoe UI', sans-serif;
            overflow: hidden;
        }

        #game-wrapper {
            position: relative;
            width: 350px;
            height: 550px;
            border-radius: 20px;
            box-shadow: 0 0 50px rgba(0,0,0,0.8);
            overflow: hidden;
            border: 4px solid #222;
        }

        canvas { display: block; }

        #menu {
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0, 0, 0, 0.8);
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            z-index: 10;
            transition: 0.5s;
        }

        h1 {
            color: #00f2ff;
            font-size: 32px;
            margin: 0;
            text-shadow: 0 0 15px #00f2ff;
        }

        button {
            background: #ff00ff;
            color: white;
            border: none;
            padding: 15px 40px;
            font-size: 18px;
            font-weight: bold;
            border-radius: 50px;
            cursor: pointer;
            box-shadow: 0 0 20px #ff00ff;
            margin-top: 30px;
        }
    </style>
</head>
<body>

<div id="game-wrapper">
    <canvas id="raceCanvas" width="350" height="550"></canvas>
    <div id="menu">
        <h1 id="title">NITRO RACER</h1>
        <button id="startBtn">START ENGINE</button>
    </div>
</div>

<script>
    const canvas = document.getElementById('raceCanvas');
    const ctx = canvas.getContext('2d');
    const menu = document.getElementById('menu');
    const startBtn = document.getElementById('startBtn');

    const LANES = [65, 175, 285];
    let playerLane = 1;
    let score = 0;
    let enemies = [];
    let gameRunning = false;
    let speed = 6;
    let timeTick = 0; // Controls day/night cycle

    function getCycleColors() {
        // cycle moves from 0 to 2000 and loops
        // 0-1000 is Day, 1000-2000 is Night
        const phase = (Math.sin(timeTick / 300) + 1) / 2; // 0 (Day) to 1 (Night)
        return {
            bg: `rgb(${10 - 10 * phase}, ${20 - 20 * phase}, ${40 - 40 * phase})`,
            road: `rgb(${40 - 30 * phase}, ${40 - 30 * phase}, ${50 - 35 * phase})`,
            glow: 5 + (25 * phase),
            isNight: phase > 0.6
        };
    }

    function drawCar(x, y, color, glowColor, isPlayer) {
        const cycle = getCycleColors();
        ctx.save();
        
        // Glow effect
        ctx.shadowBlur = cycle.glow;
        ctx.shadowColor = glowColor;
        ctx.fillStyle = color;
        
        // Body
        ctx.beginPath();
        ctx.roundRect(x - 25, y, 50, 85, 8);
        ctx.fill();

        // Windows
        ctx.shadowBlur = 0;
        ctx.fillStyle = 'rgba(0,0,0,0.7)';
        ctx.fillRect(x - 18, y + 15, 36, 20);

        // Lights
        if (cycle.isNight) {
            // Headlights
            ctx.fillStyle = "rgba(255, 255, 200, 0.8)";
            ctx.beginPath();
            ctx.moveTo(x - 20, y);
            ctx.lineTo(x - 40, y - 40);
            ctx.lineTo(x - 0, y - 40);
            ctx.fill();
            ctx.beginPath();
            ctx.moveTo(x + 20, y);
            ctx.lineTo(x + 40, y - 40);
            ctx.lineTo(x + 0, y - 40);
            ctx.fill();
        }

        // Tail Lights
        ctx.fillStyle = '#ff0000';
        ctx.fillRect(x - 20, y + 78, 8, 4);
        ctx.fillRect(x + 12, y + 78, 8, 4);
        
        ctx.restore();
    }

    function update() {
        if (!gameRunning) return;

        timeTick++;
        const colors = getCycleColors();

        // 1. Clear Background (Sky/Space)
        ctx.fillStyle = colors.bg;
        ctx.fillRect(0, 0, canvas.width, canvas.height);

        // 2. Draw Road
        ctx.fillStyle = colors.road;
        ctx.fillRect(20, 0, 310, 550);

        // 3. Lane Markers
        ctx.strokeStyle = colors.isNight ? '#444' : '#666';
        ctx.setLineDash([30, 40]);
        ctx.lineDashOffset = -(timeTick * speed) % 70;
        [120, 230].forEach(x => {
            ctx.beginPath();
            ctx.moveTo(x, 0); ctx.lineTo(x, 550);
            ctx.stroke();
        });

        // 4. Player
        const targetX = LANES[playerLane];
        drawCar(targetX, 430, '#ff00ff', '#ff00ff', true);

        // 5. Enemies
        if (timeTick % 50 === 0) {
            enemies.push({ x: LANES[Math.floor(Math.random() * 3)], y: -100 });
        }

        enemies.forEach((enemy, index) => {
            enemy.y += speed;
            drawCar(enemy.x, enemy.y, '#00ccff', '#00ccff', false);

            if (enemy.x === targetX && enemy.y + 80 > 430 && enemy.y < 430 + 85) {
                gameRunning = false;
                menu.style.display = 'flex';
                document.getElementById('title').innerText = "WASTED";
            }

            if (enemy.y > 600) {
                enemies.splice(index, 1);
                score++;
                speed += 0.05;
            }
        });

        // 6. UI
        ctx.setLineDash([]);
        ctx.fillStyle = colors.isNight ? '#00f2ff' : '#000';
        ctx.font = 'bold 18px Arial';
        ctx.fillText(`SCORE: ${score}`, 30, 40);
        ctx.fillText(colors.isNight ? "NIGHT" : "DAY", 270, 40);

        requestAnimationFrame(update);
    }

    window.addEventListener('keydown', (e) => {
        if (e.key === "ArrowLeft" && playerLane > 0) playerLane--;
        if (e.key === "ArrowRight" && playerLane < 2) playerLane++;
    });

    startBtn.addEventListener('click', () => {
        score = 0; speed = 6; enemies = []; timeTick = 0;
        gameRunning = true;
        menu.style.display = 'none';
        update();
    });
</script>
</body>
</html>

<!DOCTYPE html>
<html>
<head>
    <title>Nitro Racer: Garage Edition</title>
    <style>
        body {
            background-color: #000;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            font-family: 'Segoe UI', sans-serif;
            color: white;
            overflow: hidden;
        }

        #game-wrapper {
            position: relative;
            width: 360px;
            height: 600px;
            border-radius: 20px;
            box-shadow: 0 0 50px rgba(0,0,0,0.8);
            overflow: hidden;
            border: 4px solid #222;
            background: #000;
        }

        canvas { display: block; }

        #menu {
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0, 0, 0, 0.9);
            display: flex;
            flex-direction: column;
            align-items: center;
            z-index: 10;
            padding-top: 40px;
        }

        .garage-box {
            background: #1a1a1a;
            padding: 20px;
            border-radius: 15px;
            width: 80%;
            margin-top: 20px;
            border: 1px solid #333;
            text-align: center;
        }

        h1 { color: #00f2ff; margin: 0; text-shadow: 0 0 10px #00f2ff; font-size: 28px; }

        label { display: block; margin: 10px 0 5px; font-size: 12px; color: #aaa; text-transform: uppercase; }

        input[type="color"] {
            width: 100%; height: 40px; border: none; cursor: pointer; background: none;
        }

        select {
            width: 100%; padding: 8px; background: #333; color: white; border: none; border-radius: 5px;
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
            margin-top: 25px;
            transition: 0.2s;
        }

        button:hover { transform: scale(1.05); }
    </style>
</head>
<body>

<div id="game-wrapper">
    <canvas id="raceCanvas" width="360" height="600"></canvas>
    
    <div id="menu">
        <h1>NITRO GARAGE</h1>
        
        <div class="garage-box">
            <label>Paint Job</label>
            <input type="color" id="carColor" value="#ff00ff">
            
            <label>Body Style</label>
            <select id="decalStyle">
                <option value="none">Standard</option>
                <option value="stripes">Racing Stripes</option>
                <option value="carbon">Carbon Fiber</option>
            </select>

            <label>Underglow Intensity</label>
            <input type="range" id="glowPower" min="0" max="40" value="20">
        </div>

        <button id="startBtn">START ENGINE</button>
    </div>
</div>

<script>
    const canvas = document.getElementById('raceCanvas');
    const ctx = canvas.getContext('2d');
    const menu = document.getElementById('menu');
    const startBtn = document.getElementById('startBtn');
    
    // Customization Inputs
    const colorInput = document.getElementById('carColor');
    const decalInput = document.getElementById('decalStyle');
    const glowInput = document.getElementById('glowPower');

    const LANES = [70, 180, 290];
    let playerLane = 1;
    let score = 0;
    let enemies = [];
    let gameRunning = false;
    let speed = 6;
    let timeTick = 0;

    function drawCar(x, y, color, isPlayer) {
        ctx.save();
        
        // Underglow
        const glowStr = isPlayer ? glowInput.value : 10;
        ctx.shadowBlur = glowStr;
        ctx.shadowColor = color;
        
        // Body
        ctx.fillStyle = color;
        ctx.beginPath();
        ctx.roundRect(x - 25, y, 50, 85, 8);
        ctx.fill();
        ctx.shadowBlur = 0;

        // Decals for Player
        if (isPlayer) {
            if (decalInput.value === 'stripes') {
                ctx.fillStyle = 'rgba(255,255,255,0.3)';
                ctx.fillRect(x - 15, y, 10, 85);
                ctx.fillRect(x + 5, y, 10, 85);
            } else if (decalInput.value === 'carbon') {
                ctx.fillStyle = 'rgba(0,0,0,0.2)';
                for(let i=0; i<85; i+=4) ctx.fillRect(x-25, y+i, 50, 1);
            }
        }

        // Windows
        ctx.fillStyle = 'rgba(0,0,0,0.7)';
        ctx.fillRect(x - 18, y + 15, 36, 20);

        // Lights
        ctx.fillStyle = '#ff0000';
        ctx.fillRect(x - 20, y + 78, 8, 4);
        ctx.fillRect(x + 12, y + 78, 8, 4);
        
        ctx.restore();

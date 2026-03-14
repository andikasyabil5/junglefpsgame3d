<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Jungle FPS - Fixed Movement</title>
    
    <script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>
    <script src="https://unpkg.com/aframe-environment-component@1.3.3/dist/aframe-environment-component.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/nipplejs/0.10.1/nipplejs.min.js"></script>

    <style>
        body { margin: 0; overflow: hidden; background-color: #000; font-family: sans-serif; user-select: none; }
        #ui-layer { position: absolute; top: 0; left: 0; width: 100vw; height: 100vh; pointer-events: none; z-index: 10; }
        
        .screen { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,20,0,0.9); 
                  display: flex; flex-direction: column; align-items: center; justify-content: center; pointer-events: auto; z-index: 20; color: white; }
        .hidden { display: none !important; }
        
        #hud { width: 100%; height: 100%; display: none; position: relative; }
        #minimap-container { position: absolute; top: 15px; left: 15px; width: 120px; height: 120px; 
                             background: rgba(0,0,0,0.5); border: 2px solid #4ade80; border-radius: 50%; overflow: hidden; }
        #minimap { width: 100%; height: 100%; }
        
        #stats { position: absolute; top: 15px; right: 15px; text-align: right; }
        #health-bar-container { width: 150px; height: 15px; background: #333; border: 1px solid #fff; margin-top: 5px; }
        #health-bar { width: 100%; height: 100%; background: #22c55e; transition: width 0.3s; }
        
        #crosshair { position: absolute; top: 50%; left: 50%; width: 20px; height: 20px; 
                     margin: -10px 0 0 -10px; border: 2px solid rgba(255,255,255,0.5); border-radius: 50%; }
        
        #joystick-zone { position: absolute; bottom: 40px; left: 40px; width: 120px; height: 120px; pointer-events: auto; }
        #shoot-button { position: absolute; bottom: 40px; right: 40px; width: 80px; height: 80px; 
                        background: rgba(255,0,0,0.5); border-radius: 50%; border: 2px solid white; pointer-events: auto; }
        
        #vignette { position: absolute; top: 0; left: 0; width: 100%; height: 100%; 
                    box-shadow: inset 0 0 100px rgba(255,0,0,0); transition: box-shadow 0.1s; pointer-events: none; }
        
        button { padding: 15px 30px; font-size: 1.2rem; background: #22c55e; color: white; border: none; cursor: pointer; border-radius: 5px; }
    </style>
</head>
<body>

    <div id="ui-layer">
        <div id="vignette"></div>

        <div id="start-screen" class="screen">
            <h1>JUNGLE FPS</h1>
            <p>Move: Left Stick | Shoot: Right Button</p>
            <button onclick="startGame()">START GAME</button>
        </div>

        <div id="game-over-screen" class="screen hidden">
            <h1>GAME OVER</h1>
            <p id="final-score">Score: 0</p>
            <button onclick="location.reload()">RETRY</button>
        </div>

        <div id="hud">
            <div id="minimap-container"><canvas id="minimap" width="120" height="120"></canvas></div>
            <div id="stats">
                <div id="score">Score: 0</div>
                <div id="health-bar-container"><div id="health-bar"></div></div>
            </div>
            <div id="crosshair"></div>
            <div id="joystick-zone"></div>
            <div id="shoot-button"></div>
        </div>
    </div>

    <a-scene loading-screen="enabled: false" renderer="antialias: true">
        <a-entity environment="preset: forest; fog: 0; dressingAmount: 300;"></a-entity>

        <a-entity id="player" position="0 0 0">
            <a-entity id="camera" camera fov="90" look-controls position="0 1.6 0">
                <a-entity id="raycaster" raycaster="objects: .enemy; far: 50"></a-entity>
                
                <a-entity id="gun" position="0.3 -0.3 -0.5">
                    <a-box width="0.1" height="0.15" depth="0.5" color="#222"></a-box>
                    <a-entity id="muzzle-flash" position="0 0 -0.3" visible="false">
                        <a-sphere radius="0.1" color="yellow"></a-sphere>
                    </a-entity>
                </a-entity>
            </a-camera>
        </a-entity>

        <a-entity id="enemy-container"></a-entity>
        
        <a-box position="-5 1 -10" width="2" height="2" depth="2" color="#4b3621"></a-box>
        <a-box position="8 1.5 -15" width="3" height="3" depth="3" color="#4b3621"></a-box>
    </a-scene>

    <script>
        let score = 0, health = 100, gameState = 'start', enemies = [];
        let moveVector = { x: 0, y: 0 };
        
        const player = document.querySelector('#player');
        const camera = document.querySelector('#camera');
        const enemyContainer = document.querySelector('#enemy-container');

        // --- FIXED MOVEMENT COMPONENT ---
        AFRAME.registerComponent('joystick-controller', {
            tick: function() {
                if (gameState !== 'playing' || (moveVector.x === 0 && moveVector.y === 0)) return;
                
                const rotation = camera.object3D.rotation.y;
                const speed = 0.12;
                
                // Vector math to move relative to camera direction
                let dx = moveVector.x * Math.cos(rotation) - moveVector.y * Math.sin(rotation);
                let dz = moveVector.x * Math.sin(rotation) + moveVector.y * Math.cos(rotation);
                
                player.object3D.position.x += dx * speed;
                player.object3D.position.z += dz * speed;
            }
        });
        player.setAttribute('joystick-controller', '');

        // --- ENEMY LOGIC ---
        function spawnEnemy() {
            if (gameState !== 'playing') return;
            const enemy = document.createElement('a-entity');
            const angle = Math.random() * Math.PI * 2;
            const dist = 15 + Math.random() * 5;
            const x = player.object3D.position.x + Math.cos(angle) * dist;
            const z = player.object3D.position.z + Math.sin(angle) * dist;

            enemy.setAttribute('geometry', 'primitive: box; width: 0.8; height: 1.8; depth: 0.8');
            enemy.setAttribute('material', 'color: red');
            enemy.setAttribute('position', {x: x, y: 0.9, z: z});
            enemy.setAttribute('class', 'enemy');
            
            enemy.tick = () => {
                const dir = new THREE.Vector3().copy(player.object3D.position).sub(enemy.object3D.position).normalize();
                enemy.object3D.position.addScaledVector(dir, 0.03); // Slow speed for Easy Mode
                if (enemy.object3D.position.distanceTo(player.object3D.position) < 1.5) {
                    damagePlayer(0.5); // Low damage
                }
            };
            
            enemyContainer.appendChild(enemy);
            enemies.push(enemy);
            setTimeout(spawnEnemy, 4000);
        }

        // --- CORE SYSTEMS ---
        function startGame() {
            gameState = 'playing';
            document.getElementById('start-screen').classList.add('hidden');
            document.getElementById('hud').style.display = 'block';
            spawnEnemy();
            animate();
        }

        function damagePlayer(amt) {
            health -= amt;
            document.getElementById('health-bar').style.width = health + '%';
            document.getElementById('vignette').style.boxShadow = 'inset 0 0 100px rgba(255,0,0,0.5)';
            setTimeout(() => { document.getElementById('vignette').style.boxShadow = 'none'; }, 100);
            if (health <= 0) endGame();
        }

        function shoot() {
            if (gameState !== 'playing') return;
            
            // Muzzle flash & Recoil
            const flash = document.getElementById('muzzle-flash');
            flash.setAttribute('visible', 'true');
            setTimeout(() => flash.setAttribute('visible', 'false'), 50);

            const ray = document.querySelector('#raycaster').components.raycaster;
            const hits = ray.intersections;
            if (hits.length > 0) {
                const target = hits[0].object.el;
                target.parentNode.removeChild(target);
                enemies = enemies.filter(e => e !== target);
                score += 10;
                document.getElementById('score').innerText = "Score: " + score;
            }
        }

        function endGame() {
            gameState = 'over';
            document.getElementById('hud').style.display = 'none';
            document.getElementById('game-over-screen').classList.remove('hidden');
            document.getElementById('final-score').innerText = "Score: " + score;
        }

        function animate() {
            if (gameState === 'playing') {
                enemies.forEach(e => e.tick());
                drawMinimap();
            }
            requestAnimationFrame(animate);
        }

        // --- INPUTS ---
        const manager = nipplejs.create({ zone: document.getElementById('joystick-zone'), color: 'white' });
        manager.on('move', (evt, data) => {
            moveVector.x = Math.cos(data.angle.radian) * (data.distance / 50);
            moveVector.y = -Math.sin(data.angle.radian) * (data.distance / 50);
        });
        manager.on('end', () => { moveVector = { x: 0, y: 0 }; });
        
        document.getElementById('shoot-button').addEventListener('touchstart', (e) => { e.preventDefault(); shoot(); });
        document.getElementById('shoot-button').addEventListener('mousedown', shoot);

        // --- MINIMAP ---
        const canvas = document.getElementById('minimap');
        const ctx = canvas.getContext('2d');
        function drawMinimap() {
            ctx.clearRect(0, 0, 120, 120);
            ctx.fillStyle = '#4ade80';
            ctx.beginPath(); ctx.arc(60, 60, 3, 0, 7); ctx.fill(); // Player
            
            ctx.fillStyle = 'red';
            enemies.forEach(e => {
                const relX = (e.object3D.position.x - player.object3D.position.x) * 4 + 60;
                const relZ = (e.object3D.position.z - player.object3D.position.z) * 4 + 60;
                ctx.beginPath(); ctx.arc(relX, relZ, 2, 0, 7); ctx.fill();
            });
        }
    </script>
</body>
</html>

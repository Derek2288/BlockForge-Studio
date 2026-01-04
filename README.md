<!DOCTYPE html>
<html lang="uk">
<head>
    <meta charset="UTF-8">
    <title>3D Sandbox: Fixed Spawning</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <style>
        body { margin: 0; overflow: hidden; background: #000; touch-action: none; font-family: sans-serif; }
        .screen { position: fixed; top: 0; left: 0; width: 100%; height: 100%; display: flex; flex-direction: column; align-items: center; justify-content: center; z-index: 1000; color: white; }
        #loading-screen { background: #1a1a1a; z-index: 2000; }
        #menu-screen { background: linear-gradient(to bottom, #87ceeb, #27ae60); display: none; }
        #settings-screen, #achievements-screen { background: rgba(0,0,0,0.95); display: none; text-align: center; z-index: 1500; }
        .loader-bar { width: 200px; height: 10px; background: #333; border-radius: 5px; overflow: hidden; margin: 20px; }
        #loader-fill { width: 0%; height: 100%; background: #e74c3c; transition: width 0.2s; }
        .menu-btn { width: 220px; padding: 12px; margin: 8px; background: white; border: none; border-radius: 30px; font-weight: bold; color: black; cursor: pointer; pointer-events: auto; }
        #crosshair { position: fixed; top: 50%; left: 50%; width: 6px; height: 6px; background: white; border-radius: 50%; margin: -3px 0 0 -3px; z-index: 5; pointer-events: none; border: 1px solid black; display: none; }
        #ui { position: absolute; width: 100%; height: 100%; pointer-events: none; z-index: 10; display: none; }
        .game-btn { position: absolute; background: rgba(0,0,0,0.6); border: 1px solid rgba(255,255,255,0.4); color: #fff; border-radius: 50%; pointer-events: auto; display: flex; align-items: center; justify-content: center; user-select: none; }
        #joy-base { bottom: 40px; left: 40px; width: 100px; height: 100px; }
        #joy-knob { width: 40px; height: 40px; background: #fff; border-radius: 50%; position: absolute; left: 30px; top: 30px; }
        #action-btn { bottom: 40px; right: 40px; width: 90px; height: 90px; background: #e74c3c; font-weight: bold; }
        #jump-btn { bottom: 140px; right: 55px; width: 60px; height: 60px; }
        #inv-toggle { top: 20px; left: 20px; width: 55px; height: 55px; border-radius: 15px; font-size: 24px; }
        #exit-btn { top: 20px; right: 20px; width: 55px; height: 55px; border-radius: 15px; font-size: 24px; background: #c0392b; }
        #inventory-screen { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.98); display: none; flex-direction: column; align-items: center; justify-content: center; z-index: 3000; }
        #inventory-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 15px; }
        .slot { width: 85px; height: 85px; border: 2px solid #555; background: #222; color: white; display: flex; flex-direction: column; align-items: center; justify-content: center; border-radius: 12px; font-size: 10px; cursor: pointer; pointer-events: auto; }
        .slot.active { border-color: #f1c40f; background: #333; }
        .ach-item { background: #222; padding: 10px; margin: 5px; width: 280px; border-radius: 10px; border: 1px solid #444; }
    </style>
</head>
<body>

<div id="loading-screen" class="screen">
    <h2>3D SANDBOX</h2>
    <div class="loader-bar"><div id="loader-fill"></div></div>
    <p>Підготовка до спавну...</p>
</div>

<div id="menu-screen" class="screen">
    <h1>3D SANDBOX</h1>
    <h3 id="level-display">РІВЕНЬ: 1</h3>
    <button class="menu-btn" onclick="startGame()">ГРАТИ</button>
    <button class="menu-btn" onclick="openAch()">ДОСЯГНЕННЯ</button>
    <button class="menu-btn" onclick="openSettings()">НАЛАШТУВАННЯ</button>
</div>

<div id="achievements-screen" class="screen">
    <h2>ДОСЯГНЕННЯ</h2>
    <div id="ach-list" style="margin-bottom: 20px;"></div>
    <button class="menu-btn" onclick="closeAch()">НАЗАД</button>
</div>

<div id="settings-screen" class="screen">
    <h2>НАЛАШТУВАННЯ</h2>
    <p>Тіні: <input type="checkbox" id="set-shadows" checked></p>
    <p>Кров: <input type="checkbox" id="set-blood" checked></p>
    <button class="menu-btn" onclick="closeSettings()">ЗБЕРЕГТИ</button>
</div>

<div id="crosshair"></div>

<div id="ui">
    <div id="inv-toggle" class="game-btn" onclick="toggleInventory()">🎒</div>
    <div id="exit-btn" class="game-btn" onclick="exitToMenu()">🏠</div>
    <div id="joy-base" class="game-btn"><div id="joy-knob"></div></div>
    <div id="action-btn" class="game-btn">АТАКА</div>
    <div id="jump-btn" class="game-btn">JUMP</div>
</div>

<div id="inventory-screen">
    <h2 style="color: white;">ІНВЕНТАР</h2>
    <div id="inventory-grid">
        <div class="slot active" onclick="selectItem(0, 'hand')">👊 РУКИ</div>
        <div class="slot" onclick="selectItem(1, 'axe')">🪓 СОКИРА</div>
        <div class="slot" onclick="selectItem(2, 'pistol')">🔫 GLOCK</div>
        <div class="slot" onclick="selectItem(3, 'block')">📦 БЛОК</div>
        <div class="slot" onclick="selectItem(4, 'dummy')">🧍 DUMMY</div>
    </div>
    <button class="menu-btn" style="margin-top:20px;" onclick="toggleInventory()">ЗАКРИТИ</button>
</div>

<script>
    let stats = JSON.parse(localStorage.getItem('sandbox_stats')) || { level: 1, kills: 0, jumps: 0, completed: [] };
    let config = { shadows: true, blood: true };
    let scene, camera, renderer, sun, toolGroup;
    let moveX = 0, moveZ = 0, lat = 0, lon = 0, velocityY = 0, selected = 'hand';
    let isGameRunning = false;
    const blocks = [], dummies = [], particles = [];
    const raycaster = new THREE.Raycaster();

    // Завантаження
    let progress = 0;
    const loadInt = setInterval(() => {
        progress += 10;
        document.getElementById('loader-fill').style.width = progress + "%";
        if(progress >= 100) { clearInterval(loadInt); initGame(); document.getElementById('loading-screen').style.display = 'none'; document.getElementById('menu-screen').style.display = 'flex'; }
    }, 20);

    function initGame() {
        if (!window.THREE || scene) return;
        scene = new THREE.Scene(); scene.background = new THREE.Color(0x87ceeb);
        camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.shadowMap.enabled = true;
        document.body.appendChild(renderer.domElement);

        sun = new THREE.DirectionalLight(0xffffff, 1.2); sun.position.set(20, 40, 20); sun.castShadow = true;
        scene.add(sun, new THREE.AmbientLight(0xffffff, 0.5));

        const ground = new THREE.Mesh(new THREE.PlaneGeometry(200, 200), new THREE.MeshPhongMaterial({color: 0x27ae60}));
        ground.rotation.x = -Math.PI/2; ground.receiveShadow = true; scene.add(ground);
        blocks.push(ground);

        toolGroup = new THREE.Group();
        const hand = new THREE.Mesh(new THREE.BoxGeometry(0.2, 0.2, 0.5), new THREE.MeshLambertMaterial({color: 0xffdbac}));
        const axe = new THREE.Group();
        const aH = new THREE.Mesh(new THREE.BoxGeometry(0.06, 0.6, 0.06), new THREE.MeshLambertMaterial({color: 0x3d2b1f}));
        const aB = new THREE.Mesh(new THREE.BoxGeometry(0.3, 0.2, 0.1), new THREE.MeshLambertMaterial({color: 0x95a5a6}));
        aB.position.set(0.12, 0.2, 0); axe.add(aH, aB); axe.visible = false;
        const pistol = new THREE.Mesh(new THREE.BoxGeometry(0.1, 0.2, 0.4), new THREE.MeshLambertMaterial({color: 0x222222}));
        pistol.visible = false;
        toolGroup.add(hand, axe, pistol); scene.add(toolGroup);

        for(let i=0; i<15; i++) createTree((Math.random()-0.5)*70, (Math.random()-0.5)*70);
        animate();
    }

    function createTree(x, z) {
        const trunk = new THREE.Mesh(new THREE.CylinderGeometry(0.2, 0.2, 2.5), new THREE.MeshPhongMaterial({color: 0x8B4513}));
        trunk.position.set(x, 1.25, z); trunk.castShadow = trunk.receiveShadow = true;
        scene.add(trunk); blocks.push(trunk);
        const leaves = new THREE.Mesh(new THREE.SphereGeometry(1.5, 8, 8), new THREE.MeshPhongMaterial({color: 0x228B22}));
        leaves.position.set(x, 3, z); leaves.castShadow = true; scene.add(leaves); blocks.push(leaves);
    }

    function createDummy(pos) {
        const g = new THREE.Group();
        const body = new THREE.Mesh(new THREE.BoxGeometry(0.5, 0.8, 0.3), new THREE.MeshLambertMaterial({color: 0x2980b9}));
        body.position.y = 0.9; body.castShadow = true; g.add(body);
        const head = new THREE.Mesh(new THREE.BoxGeometry(0.3, 0.3, 0.3), new THREE.MeshLambertMaterial({color: 0xffdbac}));
        head.position.y = 1.5; head.castShadow = true; g.add(head);
        const eye = new THREE.Mesh(new THREE.BoxGeometry(0.05, 0.05, 0.05), new THREE.MeshBasicMaterial({color: 0}));
        eye.position.set(-0.07, 0.05, 0.16); head.add(eye);
        const eye2 = eye.clone(); eye2.position.x = 0.07; head.add(eye2);
        const arm = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.6, 0.15), new THREE.MeshLambertMaterial({color: 0xffdbac}));
        arm.position.set(0.35, 0.9, 0); g.add(arm);
        const arm2 = arm.clone(); arm2.position.x = -0.35; g.add(arm2);
        const leg = new THREE.Mesh(new THREE.BoxGeometry(0.2, 0.5, 0.2), new THREE.MeshLambertMaterial({color: 0x333333}));
        leg.position.set(0.15, 0.25, 0); g.add(leg);
        const leg2 = leg.clone(); leg2.position.x = -0.15; g.add(leg2);

        g.position.set(pos.x, 0, pos.z); g.userData = { hp: 5, dead: false };
        scene.add(g); dummies.push(g);
    }

    function attack() {
        toolGroup.position.z -= 0.4; setTimeout(() => toolGroup.position.z += 0.4, 100);
        
        if(selected === 'dummy') {
            // ФІКС СПАВНУ: Манекен з'являється перед гравцем
            const spawnPos = new THREE.Vector3(0, 0, -3).applyQuaternion(camera.quaternion).add(camera.position);
            spawnPos.y = 0; // На землю
            createDummy(spawnPos);
            return;
        }

        raycaster.setFromCamera(new THREE.Vector2(0,0), camera);
        const hits = raycaster.intersectObjects([...blocks, ...dummies.flatMap(d => d.children)]);
        
        if(hits.length > 0 && hits[0].distance < 6) {
            const obj = hits[0].object;
            const d = dummies.find(dm => dm.children.includes(obj));
            if(d && !d.userData.dead) {
                d.userData.hp -= (selected==='pistol'?5:selected==='axe'?3:1);
                if(config.blood) spawnBlood(hits[0].point);
                if(d.userData.hp <= 0) {
                    d.userData.dead = true; d.rotation.x = Math.PI/2; d.position.y = 0.15;
                    if(config.blood) {
                        const puddle = new THREE.Mesh(new THREE.CircleGeometry(0.7, 12), new THREE.MeshBasicMaterial({color: 0x880000, transparent: true, opacity: 0.7}));
                        puddle.rotation.x = -Math.PI/2; puddle.position.set(d.position.x, 0.01, d.position.z); scene.add(puddle);
                    }
                    stats.kills++; saveStats();
                }
            } else if(selected === 'block') {
                const b = new THREE.Mesh(new THREE.BoxGeometry(1,1,1), new THREE.MeshPhongMaterial({color: 0x888888}));
                b.position.copy(hits[0].point.add(hits[0].face.normal.multiplyScalar(0.5)));
                b.castShadow = b.receiveShadow = true; scene.add(b); blocks.push(b);
            }
        }
    }

    function spawnBlood(pos) {
        for(let i=0; i<6; i++) {
            const p = new THREE.Mesh(new THREE.SphereGeometry(0.05), new THREE.MeshBasicMaterial({color: 0xaa0000}));
            p.position.copy(pos); p.userData = {v: new THREE.Vector3((Math.random()-0.5)*0.1, 0.1, (Math.random()-0.5)*0.1), life: 1};
            scene.add(p); particles.push(p);
        }
    }

    window.openSettings = () => { document.getElementById('menu-screen').style.display='none'; document.getElementById('settings-screen').style.display='flex'; };
    window.closeSettings = () => { config.shadows = document.getElementById('set-shadows').checked; config.blood = document.getElementById('set-blood').checked; document.getElementById('settings-screen').style.display='none'; document.getElementById('menu-screen').style.display='flex'; };
    window.openAch = () => {
        document.getElementById('menu-screen').style.display='none';
        const list = document.getElementById('ach-list'); list.innerHTML = `<div class="ach-item">Вбивств: ${stats.kills}</div><div class="ach-item">Стрибків: ${stats.jumps}</div>`;
        document.getElementById('achievements-screen').style.display='flex';
    };
    window.closeAch = () => { document.getElementById('achievements-screen').style.display='none'; document.getElementById('menu-screen').style.display='flex'; };
    window.startGame = () => { document.getElementById('menu-screen').style.display='none'; document.getElementById('ui').style.display='block'; document.getElementById('crosshair').style.display='block'; isGameRunning=true; };
    window.exitToMenu = () => { isGameRunning=false; document.getElementById('ui').style.display='none'; document.getElementById('crosshair').style.display='none'; document.getElementById('menu-screen').style.display='flex'; };
    window.toggleInventory = () => { const inv = document.getElementById('inventory-screen'); inv.style.display = inv.style.display === 'flex' ? 'none' : 'flex'; };
    window.selectItem = (idx, type) => {
        selected = type; document.querySelectorAll('.slot').forEach((s,i) => s.classList.toggle('active', i===idx));
        toolGroup.children[0].visible = (type==='hand'); toolGroup.children[1].visible = (type==='axe'); toolGroup.children[2].visible = (type==='pistol');
        toggleInventory();
    };

    function saveStats() { localStorage.setItem('sandbox_stats', JSON.stringify(stats)); document.getElementById('level-display').innerText = "РІВЕНЬ: " + stats.level; }

    let joyId = null, lastLon, lastLat;
    window.addEventListener('touchstart', e => {
        if(!isGameRunning) return;
        const t = e.changedTouches[0];
        if(t.target.id === 'action-btn') { attack(); return; }
        if(t.target.id === 'jump-btn') { if(camera.position.y <= 1.71) { velocityY = 0.22; stats.jumps++; } return; }
        const r = document.getElementById('joy-base').getBoundingClientRect();
        if(t.clientX > r.left && t.clientX < r.right) joyId = t.identifier;
        else { lastLon = t.clientX; lastLat = t.clientY; }
    });

    window.addEventListener('touchmove', e => {
        if(!isGameRunning) return;
        for(let t of e.changedTouches) {
            if(t.identifier === joyId) {
                const r = document.getElementById('joy-base').getBoundingClientRect();
                moveX = (t.clientX - (r.left+50))/50; moveZ = (t.clientY - (r.top+50))/50;
                document.getElementById('joy-knob').style.transform = `translate(${moveX*30}px, ${moveZ*30}px)`;
            } else {
                lon += (t.clientX - lastLon) * 0.3; lat -= (t.clientY - lastLat) * 0.3;
                lat = Math.max(-85, Math.min(85, lat)); lastLon = t.clientX; lastLat = t.clientY;
            }
        }
    });
    window.addEventListener('touchend', e => { for(let t of e.changedTouches) if(t.identifier === joyId) { joyId = null; moveX = 0; moveZ = 0; document.getElementById('joy-knob').style.transform = ''; } });

    function animate() {
        requestAnimationFrame(animate);
        if(!isGameRunning) return;
        camera.rotation.order = 'YXZ'; camera.rotation.y = THREE.MathUtils.degToRad(-lon); camera.rotation.x = THREE.MathUtils.degToRad(lat);
        const dir = new THREE.Vector3(moveX, 0, moveZ).applyAxisAngle(new THREE.Vector3(0,1,0), camera.rotation.y);
        camera.position.addScaledVector(dir, 0.15);
        velocityY -= 0.012; camera.position.y += velocityY; if(camera.position.y < 1.7) { camera.position.y = 1.7; velocityY = 0; }
        toolGroup.position.copy(camera.position); toolGroup.quaternion.copy(camera.quaternion);
        toolGroup.translateX(0.4); toolGroup.translateY(-0.3); toolGroup.translateZ(-0.6);
        particles.forEach((p, i) => { p.position.add(p.userData.v); p.userData.v.y -= 0.005; p.userData.life -= 0.02; if(p.userData.life <= 0) { scene.remove(p); particles.splice(i, 1); } });
        renderer.render(scene, camera);
    }
</script>
</body>
</html>


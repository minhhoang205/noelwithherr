
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI Xmas Gallery - Premium Edition</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Segoe UI', sans-serif; }
        canvas { display: block; }
        #ui-container {
            position: absolute; top: 20px; left: 20px;
            background: rgba(0, 0, 0, 0.85); padding: 20px; border-radius: 15px;
            color: white; border: 1px solid #ffcc00; z-index: 10; backdrop-filter: blur(10px);
        }
        .instruction { font-size: 13px; color: #ddd; margin-top: 10px; line-height: 1.6; }
        #video-container {
            position: absolute; bottom: 20px; right: 20px;
            width: 180px; height: 135px; border-radius: 12px;
            overflow: hidden; border: 2px solid #ffcc00; transform: scaleX(-1);
            background: #111;
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        input[type="color"] { vertical-align: middle; cursor: pointer; border: none; width: 40px; height: 40px; background: none; }
    </style>
</head>
<body>

<div id="ui-container">
    <h2 style="margin:0 0 5px 0; color: #ffcc00;">Xmas Magic AI</h2>
    <label>Màu Đèn: </label>
    <input type="color" id="particleColor" value="#00ff88">
    <div class="instruction">
        • <b>Xòe tay:</b> Mở album (Hạt đẩy ra xa)<br>
        • <b>Nắm tay:</b> Hiện cây thông & Ngôi sao<br>
        • <b>Phẩy tay:</b> Chuyển chính xác 1 ảnh<br>
        • <b>Giơ 1 ngón trỏ:</b> Phóng to ảnh trung tâm
    </div>
</div>

<div id="video-container">
    <video id="webcam" autoplay playsinline></video>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@tweenjs/tween.js@18.6.4/dist/tween.umd.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
    const myImages = [
        'https://sf-static.upanhlaylink.com/img/image_20251223158b8f4e69fccef93ee8225e9e404911.jpg', 
        'https://sf-static.upanhlaylink.com/img/image_20251223d0ccead908738c3199517532ed2d7415.jpg',
        'https://sf-static.upanhlaylink.com/img/image_20251223f7dca8b1456940e2c73650a2992b9abc.jpg',
        'https://sf-static.upanhlaylink.com/img/image_202512234e239f0422a49966c834104381edaff2.jpg',
        'https://sf-static.upanhlaylink.com/img/image_20251223935cfd3a9fd49834aaa64efd6b8ae948.jpg',
        'https://sf-static.upanhlaylink.com/img/image_20251223bee24b92ab7a2b65b8be013c52c370d8.jpg'
    ];

    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);
    camera.position.z = 10;

    // --- NGÔI SAO 5 CÁNH ---
    function createStarShape() {
        const shape = new THREE.Shape();
        const outerRad = 0.4, innerRad = 0.15;
        for (let i = 0; i < 10; i++) {
            const rad = i % 2 === 0 ? outerRad : innerRad;
            const angle = (Math.PI * 2 * i) / 10;
            if (i === 0) shape.moveTo(Math.cos(angle) * rad, Math.sin(angle) * rad);
            else shape.lineTo(Math.cos(angle) * rad, Math.sin(angle) * rad);
        }
        return new THREE.ExtrudeGeometry(shape, { depth: 0.1, bevelEnabled: false });
    }
    const star = new THREE.Mesh(createStarShape(), new THREE.MeshBasicMaterial({ color: 0xffcc00 }));
    star.position.y = 4.3;
    scene.add(star);

    // --- TRÁI CHÂU ---
    const ornaments = new THREE.Group();
    for (let i = 0; i < 30; i++) {
        const ball = new THREE.Mesh(new THREE.SphereGeometry(0.12, 12, 12), new THREE.MeshStandardMaterial({ emissive: 0xff0000 }));
        const y = Math.random() * 6 - 3;
        const r = (3.5 - (y + 3) / 2) * 1.1;
        const theta = Math.random() * Math.PI * 2;
        ball.position.set(Math.cos(theta) * r, y, Math.sin(theta) * r);
        ornaments.add(ball);
    }
    scene.add(ornaments);
    scene.add(new THREE.AmbientLight(0xffffff, 0.5));

    // --- TUYẾT RƠI ---
    const SNOW_COUNT = 500;
    const snowGeo = new THREE.BufferGeometry();
    const snowPos = new Float32Array(SNOW_COUNT * 3);
    for (let i = 0; i < SNOW_COUNT; i++) {
        snowPos[i*3] = (Math.random()-0.5)*20;
        snowPos[i*3+1] = Math.random()*20 - 10;
        snowPos[i*3+2] = (Math.random()-0.5)*20;
    }
    snowGeo.setAttribute('position', new THREE.BufferAttribute(snowPos, 3));
    const snowParticles = new THREE.Points(snowGeo, new THREE.PointsMaterial({ size: 0.05, color: 0xffffff, transparent: true, opacity: 0.8 }));
    scene.add(snowParticles);

    // --- HỆ THỐNG HẠT MEGA ---
    const PARTICLE_COUNT = 15000;
    const geometry = new THREE.BufferGeometry();
    const positions = new Float32Array(PARTICLE_COUNT * 3);
    const colors = new Float32Array(PARTICLE_COUNT * 3);
    const treePos = [], expandedPos = [];

    for (let i = 0; i < PARTICLE_COUNT; i++) {
        const h = 7.5, y = Math.random() * h - 3.5;
        const r = (h/2 - (y + 3.5) / 2) * 1.1;
        const theta = Math.random() * Math.PI * 2;
        treePos.push(Math.cos(theta) * r * Math.random(), y, Math.sin(theta) * r * Math.random());
        // Khi mở rộng, đẩy hạt ra rìa xa (r=12-15) để không che ảnh
        const exR = 12 + Math.random() * 3;
        const exTheta = Math.random() * Math.PI * 2;
        expandedPos.push(Math.cos(exTheta) * exR, (Math.random()-0.5)*15, Math.sin(exTheta) * exR);
    }
    geometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array(treePos), 3));
    geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    const particles = new THREE.Points(geometry, new THREE.PointsMaterial({ size: 0.05, vertexColors: true, transparent: true, blending: THREE.AdditiveBlending }));
    scene.add(particles);

    // --- ALBUM ẢNH ---
    const imageGroup = new THREE.Group();
    scene.add(imageGroup);
    const planeMeshes = [];
    const loader = new THREE.TextureLoader();
    loader.setCrossOrigin('anonymous');
    const stepAngle = (Math.PI * 2) / myImages.length;

    myImages.forEach((url, i) => {
        loader.load(url, (tex) => {
            const mesh = new THREE.Mesh(
                new THREE.PlaneGeometry(2.5, 3.2),
                new THREE.MeshBasicMaterial({ map: tex, transparent: true, opacity: 0, side: THREE.DoubleSide })
            );
            const angle = i * stepAngle;
            mesh.position.set(Math.cos(angle) * 6, 0, Math.sin(angle) * 6);
            mesh.lookAt(0,0,0);
            planeMeshes.push(mesh);
            imageGroup.add(mesh);
        });
    });

    // --- AI LOGIC (ONE HAND SNAP) ---
    let isExpanded = false;
    let targetRotation = 0;
    let lastHandX = 0;
    let canSwipe = true;

    function onResults(results) {
        if (!results.multiHandLandmarks || results.multiHandLandmarks.length === 0) return;
        const hand = results.multiHandLandmarks[0];
        const isFist = hand[8].y > hand[6].y && hand[12].y > hand[10].y;
        
        if (!isFist && !isExpanded) switchState(true);
        if (isFist && isExpanded) switchState(false);

        if (isExpanded) {
            const handX = hand[0].x;
            const deltaX = handX - lastHandX;
            if (Math.abs(deltaX) > 0.05 && canSwipe) {
                targetRotation += (deltaX > 0 ? -stepAngle : stepAngle);
                canSwipe = false;
                setTimeout(() => canSwipe = true, 600);
            }
            lastHandX = handX;

            const isPointing = hand[8].y < hand[6].y && hand[12].y > hand[10].y && hand[16].y > hand[14].y;
            if (isPointing) zoomCenter(); else resetZoom();
        }
    }

    function switchState(expand) {
        isExpanded = expand;
        const target = expand ? expandedPos : treePos;
        const posAttr = geometry.attributes.position;
        for (let i = 0; i < PARTICLE_COUNT; i++) {
            new TWEEN.Tween(posAttr.array).to({ [i*3]: target[i*3], [i*3+1]: target[i*3+1], [i*3+2]: target[i*3+2] }, 1500).easing(TWEEN.Easing.Quartic.Out).start();
        }
        planeMeshes.forEach(m => new TWEEN.Tween(m.material).to({ opacity: expand ? 1 : 0 }, 800).start());
        const starS = expand ? 0.001 : 1;
        new TWEEN.Tween(star.scale).to({x:starS, y:starS, z:starS}, 800).start();
        new TWEEN.Tween(ornaments.scale).to({x:starS, y:starS, z:starS}, 800).start();
    }

    function zoomCenter() {
        planeMeshes.forEach(m => {
            const wp = new THREE.Vector3();
            m.getWorldPosition(wp);
            if (wp.z > 5.5) new TWEEN.Tween(m.scale).to({x:2.5, y:2.5}, 300).start();
        });
    }

    function resetZoom() {
        planeMeshes.forEach(m => new TWEEN.Tween(m.scale).to({x:1, y:1}, 300).start());
    }

    // --- ANIMATION ---
    const colorPicker = document.getElementById('particleColor');
    function animate(time) {
        requestAnimationFrame(animate);
        TWEEN.update(time);

        imageGroup.rotation.y += (targetRotation - imageGroup.rotation.y) * 0.1;

        // Tuyết rơi
        const sPos = snowParticles.geometry.attributes.position.array;
        for(let i=0; i<SNOW_COUNT; i++) {
            sPos[i*3+1] -= 0.02;
            if(sPos[i*3+1] < -10) sPos[i*3+1] = 10;
        }
        snowParticles.geometry.attributes.position.needsUpdate = true;

        // LED & Ngôi sao
        const baseCol = new THREE.Color(colorPicker.value);
        for (let i = 0; i < PARTICLE_COUNT; i++) {
            const p = Math.sin(time * 0.003 + i*0.1) * 0.4 + 0.6;
            colors[i*3] = baseCol.r * p; colors[i*3+1] = baseCol.g * p; colors[i*3+2] = baseCol.b * p;
        }
        geometry.attributes.color.needsUpdate = true;
        geometry.attributes.position.needsUpdate = true;
        star.rotation.z += 0.05;
        star.material.color.setHSL(0.12, 1, 0.5 + Math.sin(time*0.01)*0.2);

        // Trái châu
        ornaments.children.forEach((b, i) => b.material.emissive.setHSL((time*0.001 + i/30)%1, 1, 0.5));

        if (!isExpanded) scene.rotation.y += 0.005;
        renderer.render(scene, camera);
    }

    const handsAI = new Hands({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` });
    handsAI.setOptions({ maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.6 });
    handsAI.onResults(onResults);
    new Camera(document.getElementById('webcam'), { onFrame: async () => await handsAI.send({image: document.getElementById('webcam')}) }).start();

    animate();
</script>
</body>
</html>

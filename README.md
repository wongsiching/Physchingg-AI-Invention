# Physchingg-AI-Invention
AI invention for teaching
[3D立體圖形切割器.html](https://github.com/user-attachments/files/23670125/3D.html)
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D 立體幾何切割模型 (Interactive Slicer)</title>
    <style>
        body { margin: 0; overflow: hidden; font-family: 'Segoe UI', sans-serif; background-color: #1a1a1a; color: white; }
        #info {
            position: absolute;
            top: 10px;
            left: 10px;
            background: rgba(0, 0, 0, 0.8);
            padding: 15px;
            border-radius: 8px;
            pointer-events: none; /* 讓滑鼠事件穿透到 canvas */
            border: 1px solid #444;
        }
        h1 { margin: 0 0 10px 0; font-size: 1.2rem; color: #4fd1c5; }
        p { margin: 5px 0; font-size: 0.9rem; color: #ccc; }
        .math-eq {
            font-family: 'Courier New', monospace;
            color: #f6ad55;
            background: #333;
            padding: 4px;
            border-radius: 4px;
            display: inline-block;
            margin-top: 5px;
        }
        #instructions {
            position: absolute;
            bottom: 20px;
            width: 100%;
            text-align: center;
            color: #888;
            font-size: 0.8rem;
            pointer-events: none;
        }
    </style>
    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
                "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
            }
        }
    </script>
</head>
<body>

    <div id="info">
        <h1>立體圖案切割器</h1>
        <p>目前形狀: <span id="shape-name" style="color:white; font-weight:bold;">圓柱體</span></p>
        <p>平面方程式 (Plane Equation):</p>
        <div id="equation" class="math-eq">0.00x + 1.00y + 0.00z = 0.00</div>
    </div>

    <div id="instructions">
        左鍵旋轉視角 | 右鍵平移 | 滾輪縮放 | 使用右側面板控制切割
    </div>

    <script type="module">
        import * as THREE from 'three';
        import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
        import { GUI } from 'three/addons/libs/lil-gui.module.min.js';

        // --- 1. 初始化場景 ---
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x222222);

        const camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 100);
        camera.position.set(15, 10, 15);

        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.localClippingEnabled = true; // 重要：啟用局部切割
        document.body.appendChild(renderer.domElement);

        // 控制器
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;

        // 燈光
        const ambientLight = new THREE.AmbientLight(0x404040);
        scene.add(ambientLight);
        const spotLight = new THREE.SpotLight(0xffffff, 500);
        spotLight.position.set(10, 20, 10);
        spotLight.angle = Math.PI / 4;
        spotLight.penumbra = 0.1;
        scene.add(spotLight);
        const dirLight = new THREE.DirectionalLight(0xffffff, 1);
        dirLight.position.set(-10, 10, -10);
        scene.add(dirLight);

        // --- 2. 數學核心：定義切割平面 ---
        // 初始平面：法向量 (0, -1, 0)，常數 0
        const clipPlane = new THREE.Plane(new THREE.Vector3(0, -1, 0), 0);
        
        // 視覺化平面 (PlaneHelper)
        const planeHelper = new THREE.PlaneHelper(clipPlane, 20, 0x444444);
        planeHelper.visible = true;
        scene.add(planeHelper);

        // --- 3. 建立幾何物體 (使用 Group 管理) ---
        const objectGroup = new THREE.Group();
        scene.add(objectGroup);

        // 用於渲染的材質
        // 1. 外部材質 (藍色，有光澤)
        const materialOutside = new THREE.MeshPhongMaterial({
            color: 0x00aaff,
            shininess: 100,
            side: THREE.FrontSide, // 只渲染正面
            clippingPlanes: [clipPlane], // 套用切割
            clipShadows: true
        });

        // 2. 內部截面材質 (紅色，模擬實心)
        // 技巧：渲染物體的「背面」，當正面被切掉時，我們看到的是背面的內側
        const materialInside = new THREE.MeshPhongMaterial({
            color: 0xff3333,
            side: THREE.BackSide, // 只渲染背面
            clippingPlanes: [clipPlane], // 套用切割
            clipShadows: true
        });

        let currentMeshOutside = null;
        let currentMeshInside = null;

        // 形狀生成器
        const geometries = {
            'Cylinder (圓柱)': new THREE.CylinderGeometry(4, 4, 12, 64),
            'Cone (圓錐)': new THREE.CylinderGeometry(0, 4, 12, 64),
            'Cube (正方體)': new THREE.BoxGeometry(7, 7, 7),
            'Sphere (球體)': new THREE.SphereGeometry(4.5, 64, 64),
            'Torus (圓環)': new THREE.TorusGeometry(4, 1.5, 32, 100)
        };

        function updateGeometry(type) {
            // 清除舊物體
            if (currentMeshOutside) {
                objectGroup.remove(currentMeshOutside);
                objectGroup.remove(currentMeshInside);
                currentMeshOutside.geometry.dispose();
                currentMeshInside.geometry.dispose();
            }

            const geometry = geometries[type].clone(); // 複製一份以免修改原始檔

            // 建立外部網格
            currentMeshOutside = new THREE.Mesh(geometry, materialOutside);
            // 建立內部網格 (截面)
            currentMeshInside = new THREE.Mesh(geometry, materialInside);

            objectGroup.add(currentMeshOutside);
            objectGroup.add(currentMeshInside);
            
            document.getElementById('shape-name').innerText = type.split(' ')[1];
        }

        // 初始載入圓柱
        updateGeometry('Cylinder (圓柱)');

        // --- 4. GUI 介面與互動邏輯 ---
        const gui = new GUI({ title: '控制面板' });
        
        const params = {
            shape: 'Cylinder (圓柱)',
            planeX: 0, // 旋轉角度 (度)
            planeY: 0,
            planeZ: 0, // 初始水平
            planeConstant: 0, // 位置
            showPlane: true,
            wireframe: false
        };

        // 形狀選單
        gui.add(params, 'shape', Object.keys(geometries)).name('選擇形狀').onChange(val => {
            updateGeometry(val);
        });

        // 平面控制資料夾
        const planeFolder = gui.addFolder('切割刀片 (平面)');
        
        function updatePlane() {
            // 將角度轉為弧度
            const euler = new THREE.Euler(
                THREE.MathUtils.degToRad(params.planeX),
                THREE.MathUtils.degToRad(params.planeY),
                THREE.MathUtils.degToRad(params.planeZ)
            );
            
            // 計算新的法向量
            const normal = new THREE.Vector3(0, -1, 0).applyEuler(euler).normalize();
            
            // 更新 Three.js 的切割平面
            clipPlane.normal.copy(normal);
            clipPlane.constant = params.planeConstant;
            
            // 更新 UI 上的方程式
            const a = normal.x.toFixed(2);
            const b = normal.y.toFixed(2);
            const c = normal.z.toFixed(2);
            const d = params.planeConstant.toFixed(2);
            document.getElementById('equation').innerText = `${a}x + ${b}y + ${c}z = ${d}`;
        }

        planeFolder.add(params, 'planeX', 0, 180).name('X 軸旋轉').onChange(updatePlane);
        planeFolder.add(params, 'planeY', 0, 180).name('Y 軸旋轉').onChange(updatePlane);
        planeFolder.add(params, 'planeZ', 0, 180).name('Z 軸旋轉').onChange(updatePlane);
        planeFolder.add(params, 'planeConstant', -10, 10).name('位置 (推移)').onChange(updatePlane);
        planeFolder.add(params, 'showPlane').name('顯示刀片').onChange(v => planeHelper.visible = v);
        
        gui.add(params, 'wireframe').name('線框模式').onChange(v => {
            materialOutside.wireframe = v;
        });

        // 初始化平面
        updatePlane();

        // --- 5. 動畫迴圈 ---
        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            renderer.render(scene, camera);
        }
        animate();

        // 視窗縮放處理
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

    </script>
</body>
</html>

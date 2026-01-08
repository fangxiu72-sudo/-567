# -567
小爱心
<!doctype html>
<html lang="zh-CN">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>粒子心形（Three.js）</title>
<style>
  html,body { height:100%; margin:0; background:#000; overflow:hidden; }
  #info {
    position: absolute;
    left: 12px;
    top: 12px;
    color: #ddd;
    font-family: Arial, Helvetica, sans-serif;
    font-size: 13px;
    z-index: 2;
    background: rgba(0,0,0,0.3);
    padding:8px 10px;
    border-radius:6px;
  }
  a { color: #9cf; }
</style>
</head>
<body>
<div id="info">移动鼠标旋转 · 鼠标滚轮缩放 · <a href="#" id="regen">重新生成</a></div>
<script src="https://unpkg.com/three@0.152.2/build/three.min.js"></script>
<script src="https://unpkg.com/three@0.152.2/examples/js/controls/OrbitControls.js"></script>
<script>
(() => {
  const scene = new THREE.Scene();

  // 相机与渲染器
  const camera = new THREE.PerspectiveCamera(45, innerWidth/innerHeight, 0.1, 2000);
  camera.position.set(0, 40, 140);

  const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false });
  renderer.setSize(innerWidth, innerHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
  renderer.setClearColor(0x02020a);
  document.body.appendChild(renderer.domElement);

  const controls = new THREE.OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.06;
  controls.minDistance = 40;
  controls.maxDistance = 400;

  // 光线（柔和填充感）
  scene.add(new THREE.AmbientLight(0xffffff, 0.6));
  const dir = new THREE.DirectionalLight(0xffffff, 0.6);
  dir.position.set(0.5, 1, 0.5).normalize();
  scene.add(dir);

  // 全局参数
  const heartCount = 4500;
  const fieldCount = 9000;
  let heartPoints, fieldPoints;

  // 心形参数化曲线 (平面)
  function heartXY(t) {
    // t in [0, 2PI]
    // 经典心形参数方程（缩放并翻转 y 方向，使看起来更自然）
    const x = 16 * Math.pow(Math.sin(t), 3);
    const y = 13 * Math.cos(t) - 5 * Math.cos(2*t) - 2 * Math.cos(3*t) - Math.cos(4*t);
    return [x, y];
  }

  // 生成心形粒子集合（3D 有厚度）
  function createHeartParticles() {
    const geometry = new THREE.BufferGeometry();
    const positions = new Float32Array(heartCount * 3);
    const colors = new Float32Array(heartCount * 3);
    const sizes = new Float32Array(heartCount);

    // 配色：中心偏粉，外围偏紫到蓝
    for (let i = 0; i < heartCount; i++) {
      // 在参数 t 上随机采样并在径向上加偏差以产生厚度
      const t = Math.random() * Math.PI * 2;
      const r = 0.6 + Math.pow(Math.random(), 2) * 1.6; // 从中心到边缘的径向尺度
      const [hx, hy] = heartXY(t);
      // 基于 hx,hy 计算位置并缩放
      const scale = 1.6;
      const x = hx * r * scale;
      const y = hy * r * scale * 0.9; // 纵向微调
      const z = (Math.random() - 0.5) * 10 * Math.pow(Math.random(), 2); // 给点云一个厚度分布

      const idx = i * 3;
      positions[idx] = x;
      positions[idx + 1] = y;
      positions[idx + 2] = z;

      // 颜色依据 y 和 r 混合
      const c = new THREE.Color();
      // 混合粉色系和紫色系
      const mix = Math.min(1, 0.2 + Math.abs(y) / 60 + (1 - r) * 0.6);
      c.setHSL(0.87 - mix * 0.12, 0.8 - mix * 0.2, 0.6 - mix * 0.2); // 粉->紫
      colors[idx] = c.r;
      colors[idx + 1] = c.g;
      colors[idx + 2] = c.b;

      sizes[i] = 2.0 + Math.random() * 3.0;
    }

    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    geometry.setAttribute('size', new THREE.BufferAttribute(sizes, 1));

    const material = new THREE.PointsMaterial({
      vertexColors: true,
      size: 2.6,
      sizeAttenuation: true,
      transparent: true,
      opacity: 0.95,
      depthWrite: false,
      blending: THREE.AdditiveBlending,
    });

    const points = new THREE.Points(geometry, material);
    points.userData = { basePositions: positions }; // 保存原始位置用于动画
    return points;
  }

  // 生成地面/周围散落的粒子
  function createFieldParticles() {
    const geometry = new THREE.BufferGeometry();
    const positions = new Float32Array(fieldCount * 3);
    const colors = new Float32Array(fieldCount * 3);
    const sizes = new Float32Array(fieldCount);

    for (let i = 0; i < fieldCount; i++) {
      const idx = i * 3;
      // 范围在一个较大的半径内分布，靠近地面
      const angle = Math.random() * Math.PI * 2;
      const radius = 30 + Math.pow(Math.random(), 2) * 220;
      const x = Math.cos(angle) * radius + (Math.random() - 0.5) * 6;
      const z = Math.sin(angle) * radius + (Math.random() - 0.5) * 6;
      const y = (Math.random() - 0.9) * 4 - 18 + Math.pow(Math.random(), 3) * 6; // 地面偏低

      positions[idx] = x;
      positions[idx + 1] = y;
      positions[idx + 2] = z;

      const col = new THREE.Color();
      // 冷色调（蓝白），并在远处更冷
      const hue = 0.57 + Math.random() * 0.06;
      const light = 0.6 + Math.random() * 0.4;
      col.setHSL(hue, 0.6, light);
      colors[idx] = col.r;
      colors[idx + 1] = col.g;
      colors[idx + 2] = col.b;

      sizes[i] = 0.6 + Math.random() * 1.4;
    }

    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    geometry.setAttribute('size', new THREE.BufferAttribute(sizes, 1));

    const material = new THREE.PointsMaterial({
      vertexColors: true,
      size: 1.8,
      sizeAttenuation: true,
      transparent: true,
      opacity: 0.9,
      depthWrite: false,
      blending: THREE.AdditiveBlending,
    });

    return new THREE.Points(geometry, material);
  }

  function regen() {
    if (heartPoints) scene.remove(heartPoints);
    if (fieldPoints) scene.remove(fieldPoints);

    heartPoints = createHeartParticles();
    fieldPoints = createFieldParticles();

    // 轻微旋转心形让效果更“立体”
    heartPoints.rotation.x = -0.08;
    heartPoints.rotation.z = 0.02;

    scene.add(fieldPoints);
    scene.add(heartPoints);
  }

  regen();

  // 动画循环（基于时间做微小抖动/上浮）
  const clock = new THREE.Clock();
  function animate() {
    requestAnimationFrame(animate);
    const t = clock.getElapsedTime();

    // 心形粒子做呼吸和轻微上浮 + 小幅旋转
    if (heartPoints) {
      const pos = heartPoints.geometry.attributes.position.array;
      for (let i = 0; i < heartCount; i++) {
        const idx = i * 3;
        // 原始位置
        const ox = pos[idx];
        const oy = pos[idx + 1];
        const oz = pos[idx + 2];
        // 使用简单噪声替代（trig）
        const n = Math.sin(t * 1.2 + i * 0.01) * 0.6 + Math.cos(t * 0.7 + i * 0.013) * 0.4;
        pos[idx + 2] = oz + n * 0.8; // z 方向脉动
      }
      heartPoints.geometry.attributes.position.needsUpdate = true;
      heartPoints.rotation.y += 0.0008;
      heartPoints.rotation.x += 0.00015;
    }

    // 地面粒子缓慢漂浮
    if (fieldPoints) {
      fieldPoints.rotation.y = Math.sin(t * 0.03) * 0.03;
      // optional subtle bobbing via uniforms not applied here for simplicity
    }

    controls.update();
    renderer.render(scene, camera);
  }
  animate();

  // 窗口大小响应
  window.addEventListener('resize', () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
  });

  // 重新生成按钮
  document.getElementById('regen').addEventListener('click', (e) => {
    e.preventDefault();
    regen();
  });

})(); 
</script>
</body>
</html>

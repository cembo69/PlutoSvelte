<script>
  import { onMount, onDestroy, createEventDispatcher } from 'svelte';
  import * as THREE from 'three';
  import { gsap } from 'gsap';
  // import cockpitVideo from '../assets/video.mp4';
  import Galaxy from './Galaxy.svelte';

  export let stations = [];
  export let isActive = false;

  const dispatch = createEventDispatcher();

  let container;
  let renderer, scene, camera;
  let animationId;
  let tubeMesh;
  let curve;
  let stationMeshes = [];
  let activeIndex = 0;
  
  // Kamera-Ziel für weiche Bewegungen
  let targetCameraPos = new THREE.Vector3();
  let targetLookAt = new THREE.Vector3();
  let currentLookAt = new THREE.Vector3();

  // Maus-Parallaxe
  let mouseX = 0;
  let mouseY = 0;

  // Debug Tools State
  // Video Settings (Fixed from Screenshot)
  let videoOpacity = 1;
  let videoScale = 1.6;
  let videoX = -130;
  let videoY = 170;
  let videoBlendMode = 'screen';
  
  // Text Debug State
  let textX = 440;
  let textY = -10;
  let textScale = 0.9;
  let textScaleY = 0.95;
  let textRotation = 0;
  let textRotationX = 0;
  let textWidth = 600;
  let headlineSize = 3;
  let textSize = 1.1;
  
  let showDebug = false;
  let showVideoDebug = false;
  let showTextDebug = false;

  onMount(() => {
    init();
    animate();
    window.addEventListener('mousemove', onMouseMove);
    window.addEventListener('resize', onResize);
    window.addEventListener('keydown', onKeyDown);

    return () => {
      cleanup();
      window.removeEventListener('mousemove', onMouseMove);
      window.removeEventListener('resize', onResize);
      window.removeEventListener('keydown', onKeyDown);
    };
  });

  function cleanup() {
    if (animationId) cancelAnimationFrame(animationId);
    if (renderer) renderer.dispose();
  }

  function init() {
    // Scene Setup
    scene = new THREE.Scene();
    // Nebel für Tiefe: Schwarz, startet bei 10, endet bei 60
    scene.fog = new THREE.FogExp2(0x000000, 0.02);

    camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 200);
    
    renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
    renderer.setSize(container.clientWidth, container.clientHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    // Tone Mapping für besseres Leuchten
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    container.appendChild(renderer.domElement);

    // --- 1. Der Pfad (Cinematic Curve) ---
    // Wir bauen eine geschwungene Kurve durch den Raum
    const points = [];
    const spread = 30;
    for (let i = 0; i < stations.length; i++) {
      // Spiralförmig / Wellenförmig
      const t = i / (stations.length - 1);
      const angle = t * Math.PI * 4; // 2 Umdrehungen
      const x = Math.cos(angle) * spread * (1 - t * 0.5); // Wird enger
      const y = (t - 0.5) * spread * 2; // Von unten nach oben
      const z = Math.sin(angle) * spread * (1 - t * 0.5) - (t * 50); // Tiefe
      points.push(new THREE.Vector3(x, y, z));
    }
    
    curve = new THREE.CatmullRomCurve3(points);
    
    // --- 2. Die Stationen (Bilder) ---
    const textureLoader = new THREE.TextureLoader();

    stations.forEach((station, i) => {
      const point = curve.getPointAt(i / (stations.length - 1));
      
      // Gruppe für die Station
      const stationGroup = new THREE.Group();
      stationGroup.position.copy(point);
      stationGroup.userData = { id: i, originalY: point.y };
      scene.add(stationGroup);
      stationMeshes.push(stationGroup);

      // Bild-Hologramm (Plane)
      if (station.image) {
        textureLoader.load(station.image, (tex) => {
          const aspect = tex.image.width / tex.image.height;
          const planeGeo = new THREE.PlaneGeometry(6 * aspect, 6);
          const planeMat = new THREE.MeshBasicMaterial({
            map: tex,
            transparent: true,
            opacity: 0, // Startet unsichtbar
            side: THREE.DoubleSide,
            blending: THREE.AdditiveBlending,
            depthWrite: false
          });
          const plane = new THREE.Mesh(planeGeo, planeMat);
          // Positioniere das Bild im Zentrum
          plane.position.set(0, 0, 0); 
          stationGroup.add(plane);
          stationGroup.userData.hologram = plane;
        });
      }
    });

    // Lichter
    const ambientLight = new THREE.AmbientLight(0x404040);
    scene.add(ambientLight);
    
    const pointLight = new THREE.PointLight(0xffffff, 2, 100);
    scene.add(pointLight);
    // Licht folgt der Kamera später

    // Startposition setzen
    updateCameraTarget(0);
    camera.position.copy(targetCameraPos);
    currentLookAt.copy(targetLookAt);
    camera.lookAt(currentLookAt);
  }

  function updateCameraTarget(index) {
    const t = index / (stations.length - 1);
    const point = curve.getPointAt(t);
    const tangent = curve.getTangentAt(t).normalize();
    
    // Kameraposition: Ein Stück "hinter" und "über" dem Punkt, basierend auf der Kurvenrichtung
    // Wir nutzen das Kreuzprodukt, um "seitlich" zu gehen
    const up = new THREE.Vector3(0, 1, 0);
    const side = new THREE.Vector3().crossVectors(tangent, up).normalize();
    
    // Zielposition berechnen:
    // -5 Einheiten zurück (entgegen Tangente)
    // +3 Einheiten zur Seite
    // +2 Einheiten hoch
    targetCameraPos.copy(point)
      .sub(tangent.multiplyScalar(8))
      .add(side.multiplyScalar(4))
      .add(up.multiplyScalar(2));
      
    targetLookAt.copy(point);

    // Animation der Kristalle/Hologramme
    stationMeshes.forEach((group, i) => {
      const isCurrent = i === index;
      
      // Hologramm ein/ausblenden
      if (group.userData.hologram) {
        gsap.to(group.userData.hologram.material, {
          opacity: isCurrent ? 0.9 : 0,
          duration: 1.5,
          ease: "power2.inOut"
        });
        // Leichtes Schweben des Hologramms
        gsap.to(group.userData.hologram.position, {
          y: isCurrent ? 0.5 : 0,
          duration: 2,
          yoyo: true,
          repeat: -1,
          ease: "sine.inOut"
        });
      }
    });
  }

  function animate() {
    animationId = requestAnimationFrame(animate);
    
    if (!isActive) return;

    // 1. Kamera Smooth Follow (Damping)
    // Wir addieren den Maus-Offset zur Zielposition
    const finalTargetPos = targetCameraPos.clone();
    finalTargetPos.x += (mouseX * 2);
    finalTargetPos.y += (mouseY * 2);

    camera.position.lerp(finalTargetPos, 0.03); // 0.03 = sehr weich
    currentLookAt.lerp(targetLookAt, 0.04);
    camera.lookAt(currentLookAt);

    // 2. Objekte animieren
    const time = Date.now() * 0.001;
    
    stationMeshes.forEach((group, i) => {
      // Schweben der ganzen Gruppe
      group.position.y = group.userData.originalY + Math.sin(time + i) * 0.5;

      // Hologramm immer zur Kamera drehen (Billboard)
      if (group.userData.hologram && i === activeIndex) {
        group.userData.hologram.lookAt(camera.position);
      }
    });

    renderer.render(scene, camera);
  }

  function onMouseMove(event) {
    // Normalisierte Mauskoordinaten (-1 bis +1)
    mouseX = (event.clientX / window.innerWidth) * 2 - 1;
    mouseY = -(event.clientY / window.innerHeight) * 2 + 1;
  }

  function onResize() {
    if (!container) return;
    camera.aspect = container.clientWidth / container.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.clientWidth, container.clientHeight);
  }

  function next() {
    if (activeIndex < stations.length - 1) {
      setActiveStation(activeIndex + 1);
    } else {
      dispatch('complete');
    }
  }

  function prev() {
    if (activeIndex > 0) {
      setActiveStation(activeIndex - 1);
    }
  }

  function setActiveStation(index) {
    activeIndex = index;
    updateCameraTarget(index);
  }

  function onKeyDown(e) {
    if (!isActive) return;
    if (e.key === 'ArrowRight' || e.key === ' ') next();
    if (e.key === 'ArrowLeft') prev();
  }
</script>

<div class="viz-container" bind:this={container}>
  <!-- Galaxy Background -->
  <div class="galaxy-layer">
    <Galaxy 
      density={0.3}
      baseScale={30.0}
      glowIntensity={0.5}
      speed={1.0}
      starSpeed={0.5}
      twinkleIntensity={0.3}
      mouseInteraction={false}
    />
  </div>

  <!-- Cockpit Video Overlay Removed -->


  <!-- Debug Panel -->
  {#if showDebug}
    <div class="debug-panel">
      <div class="debug-header">
        <h4>Debug Tools</h4>
        <button on:click={() => showDebug = false} class="close-debug">X</button>
      </div>

      <!-- Video Section -->
      <div class="debug-section">
        <button class="section-toggle" on:click={() => showVideoDebug = !showVideoDebug}>
          {showVideoDebug ? '▼' : '▶'} Cockpit Video
        </button>
        {#if showVideoDebug}
          <label>
            Opacity: {videoOpacity}
            <input type="range" min="0" max="1" step="0.05" bind:value={videoOpacity} />
          </label>
          <label>
            Scale: {videoScale}
            <input type="range" min="0.5" max="2" step="0.05" bind:value={videoScale} />
          </label>
          <label>
            Pos X: {videoX}px
            <input type="range" min="-500" max="500" step="10" bind:value={videoX} />
          </label>
          <label>
            Pos Y: {videoY}px
            <input type="range" min="-500" max="500" step="10" bind:value={videoY} />
          </label>
          <label>
            Blend Mode:
            <select bind:value={videoBlendMode}>
              <option value="screen">Screen</option>
              <option value="overlay">Overlay</option>
              <option value="multiply">Multiply</option>
              <option value="lighten">Lighten</option>
              <option value="normal">Normal</option>
            </select>
          </label>
        {/if}
      </div>

      <!-- Text Section -->
      <div class="debug-section">
        <button class="section-toggle" on:click={() => showTextDebug = !showTextDebug}>
          {showTextDebug ? '▼' : '▶'} Text Overlay
        </button>
        {#if showTextDebug}
          <label>
            Pos X: {textX}px
            <input type="range" min="-500" max="500" step="10" bind:value={textX} />
          </label>
          <label>
            Pos Y: {textY}px
            <input type="range" min="-500" max="500" step="10" bind:value={textY} />
          </label>
          <label>
            Scale: {textScale}
            <input type="range" min="0.5" max="2" step="0.05" bind:value={textScale} />
          </label>
          <label>
            Scale Y (Stretch): {textScaleY}
            <input type="range" min="0.5" max="2" step="0.05" bind:value={textScaleY} />
          </label>
          <label>
            Rotation: {textRotation}°
            <input type="range" min="-45" max="45" step="1" bind:value={textRotation} />
          </label>
          <label>
            Rotation X (Tilt): {textRotationX}°
            <input type="range" min="-60" max="60" step="1" bind:value={textRotationX} />
          </label>
          <label>
            Width: {textWidth}px
            <input type="range" min="300" max="1200" step="10" bind:value={textWidth} />
          </label>
          <label>
            Headline Size: {headlineSize}rem
            <input type="range" min="1" max="6" step="0.1" bind:value={headlineSize} />
          </label>
          <label>
            Text Size: {textSize}rem
            <input type="range" min="0.5" max="3" step="0.1" bind:value={textSize} />
          </label>
        {/if}
      </div>
    </div>
  {/if}

  <!-- UI Overlay -->
  <div class="ui-overlay">
    
    <!-- Große Jahreszahl im Hintergrund -->
    <div class="year-display">
      {stations[activeIndex].headline.split(':')[0]}
    </div>

    <!-- Content Box -->
    <div class="content-box" style="
      transform: translate({textX}px, {textY}px) scale({textScale}, {textScale * textScaleY}) rotate({textRotation}deg) rotateX({textRotationX}deg);
      max-width: {textWidth}px;
    ">
      <div class="line-connector"></div>
      <h3 style="font-size: {headlineSize}rem">{stations[activeIndex].headline}</h3>
      <p style="font-size: {textSize}rem">{stations[activeIndex].text}</p>
      
      <div class="controls">
        <button class="nav-btn" on:click={prev} disabled={activeIndex === 0}>←</button>
        <div class="progress-bar">
          <div class="progress-fill" style="width: {(activeIndex / (stations.length - 1)) * 100}%"></div>
        </div>
        <button class="nav-btn next-btn" on:click={next}>
          {activeIndex === stations.length - 1 ? 'WEITERREISE ⇩' : '→'}
        </button>
      </div>
    </div>
  </div>
</div>

<style>
  .viz-container {
    width: 100%;
    height: 100%;
    position: absolute;
    top: 0;
    left: 0;
    overflow: hidden;
    background: transparent; /* Transparent, damit Galaxy sichtbar bleibt */
  }

  .galaxy-layer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 0; /* Ganz hinten */
    pointer-events: none;
  }

  canvas {
    display: block;
    position: absolute;
    top: 0;
    left: 0;
    z-index: 1; /* 3D Szene über Galaxy */
  }

  .cockpit-overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    z-index: 5; /* Über Canvas, unter UI */
    pointer-events: none;
    /* Styles werden jetzt inline gesetzt */
  }

  .debug-panel {
    position: absolute;
    top: 20px;
    right: 20px;
    background: rgba(0, 0, 0, 0.9);
    padding: 15px;
    border-radius: 8px;
    border: 1px solid #444;
    z-index: 1000;
    color: #fff;
    font-family: 'Helvetica', sans-serif;
    font-weight: 700;
    width: 280px;
    max-height: 90vh;
    overflow-y: auto;
  }

  .debug-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 15px;
    border-bottom: 1px solid #444;
    padding-bottom: 10px;
  }

  .debug-header h4 {
    margin: 0;
    color: #00ffff;
  }

  .debug-section {
    margin-bottom: 15px;
    border-bottom: 1px solid #333;
    padding-bottom: 10px;
  }

  .section-toggle {
    background: none;
    border: none;
    color: #00ffff;
    font-family: 'Helvetica', sans-serif;
    font-size: 0.9rem;
    cursor: pointer;
    padding: 5px 0;
    width: 100%;
    text-align: left;
    font-weight: 700;
  }

  .section-toggle:hover {
    color: #fff;
  }

  .debug-panel label {
    display: flex;
    flex-direction: column;
    margin-bottom: 10px;
    font-size: 0.8rem;
    padding-left: 10px;
  }

  .debug-panel input, .debug-panel select {
    margin-top: 5px;
    background: #222;
    color: #fff;
    border: 1px solid #555;
    padding: 4px;
  }

  .close-debug {
    background: none;
    border: none;
    color: #fff;
    cursor: pointer;
    font-size: 1.2rem;
    padding: 0 5px;
  }
  
  .close-debug:hover {
    color: #ff4444;
  }

  .ui-overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none; /* Klicks gehen durch zur 3D Szene (wenn nötig) */
    display: flex;
    flex-direction: column;
    justify-content: flex-end;
    padding: 4rem;
    box-sizing: border-box;
    /* perspective entfernt, da es Probleme verursachen kann */
  }

  .year-display {
    position: absolute;
    top: 10%;
    right: 5%;
    font-family: 'Helvetica', sans-serif;
    font-size: 15rem;
    font-weight: 700;
    color: rgba(255, 255, 255, 0.03); /* Sehr transparent */
    pointer-events: none;
    user-select: none;
    line-height: 1;
  }

  .content-box {
    pointer-events: auto;
    max-width: 600px;
    position: relative;
    z-index: 10;
    /* transform-style entfernt */
    
    /* Hologram styles */
    background: rgba(0, 20, 40, 0.6);
    border: 1px solid rgba(0, 255, 255, 0.3);
    box-shadow: 0 0 20px rgba(0, 255, 255, 0.1), inset 0 0 20px rgba(0, 255, 255, 0.1);
    padding: 2rem;
    border-radius: 4px;
    backdrop-filter: blur(5px);
  }

  /* Scanlines */
  .content-box::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: repeating-linear-gradient(
      0deg,
      transparent,
      transparent 2px,
      rgba(0, 255, 255, 0.05) 3px
    );
    pointer-events: none;
    z-index: -1;
  }

  .line-connector {
    width: 100px;
    height: 2px;
    background: #00ffff;
    margin-bottom: 1rem;
    position: relative;
    box-shadow: 0 0 10px #00ffff;
  }
  .line-connector::after {
    content: '';
    position: absolute;
    right: 0;
    top: -2px;
    width: 6px;
    height: 6px;
    background: #00ffff;
    border-radius: 50%;
    box-shadow: 0 0 10px #00ffff;
  }

  h3 {
    font-family: 'Helvetica', sans-serif;
    font-size: 3rem;
    font-weight: 700;
    margin: 0 0 1rem 0;
    color: #00ffff;
    text-shadow: 0 0 10px rgba(0, 255, 255, 0.5);
    text-transform: uppercase;
    letter-spacing: 2px;
  }

  p {
    font-family: 'Helvetica', sans-serif;
    font-size: 1.1rem;
    line-height: 1.6;
    color: rgba(200, 240, 255, 0.9);
    margin-bottom: 2rem;
    font-weight: 700;
    text-shadow: 0 0 5px rgba(0, 255, 255, 0.3);
  }

  .controls {
    display: flex;
    align-items: center;
    gap: 1.5rem;
  }

  .nav-btn {
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.2);
    color: #fff;
    width: 50px;
    height: 50px;
    border-radius: 50%;
    font-size: 1.2rem;
    cursor: pointer;
    transition: all 0.3s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    backdrop-filter: blur(10px);
  }

  .nav-btn:hover:not(:disabled) {
    background: #fff;
    color: #000;
    transform: scale(1.1);
  }

  .nav-btn:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }

  .next-btn {
    width: auto;
    padding: 0 1.5rem;
    border-radius: 25px;
  }

  .progress-bar {
    flex: 1;
    height: 2px;
    background: rgba(255, 255, 255, 0.1);
    position: relative;
  }

  .progress-fill {
    height: 100%;
    background: #00ffff;
    box-shadow: 0 0 10px #00ffff;
    transition: width 1s cubic-bezier(0.22, 1, 0.36, 1);
  }

  @media (max-width: 768px) {
    .ui-overlay {
      padding: 2rem;
    }
    .year-display {
      font-size: 8rem;
      top: 5%;
    }
    h3 {
      font-size: 2rem;
    }
  }
</style>

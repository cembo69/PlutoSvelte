<script>
  import { onMount, onDestroy, createEventDispatcher } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

  const dispatch = createEventDispatcher();

  let container;
  let renderer, scene, camera, controls;
  let animationId;
  let barycenterGroup;
  
  let plutoMesh;
  let charonMesh;
  
  let plutoScreenPosition = { x: 0, y: 0, visible: false };
  let charonScreenPosition = { x: 0, y: 0, visible: false };
  
  let isHovering = false;
  let activeInfo = null; // 'pluto' | 'charon' | null
  let resizeObserver;
  let isTransitioning = false;
  let hasNavigated = false;

  // Text & Debugging
  let headline = "Das Doppelsystem";
  let text = "Pluto und Charon tanzen um einen gemeinsamen Schwerpunkt außerhalb beider Körper – ein einzigartiges Phänomen im Sonnensystem. Sie sind gravitativ aneinander gebunden und zeigen sich immer dieselbe Seite.";
  
  let displayedText = "";
  let isTyping = false;
  let textIndex = 0;
  let typingTimeout;
  
  let showDebug = false;
  let textOffsetX = 0;
  let textOffsetY = -410; 
  let textScale = 0.8;
  
  // Debug variables
  let debugHeadline = headline;
  let debugText = text;
  let debugTextOffsetX = textOffsetX;
  let debugTextOffsetY = textOffsetY;
  let debugTextScale = textScale;

  onMount(() => {
    init();
    animate();
    startTyping();
    
    // Use ResizeObserver to handle size changes (e.g. from display:none to flex)
    resizeObserver = new ResizeObserver(() => {
      onWindowResize();
    });
    resizeObserver.observe(container);
  });

  onDestroy(() => {
    if (animationId) cancelAnimationFrame(animationId);
    if (renderer) renderer.dispose();
    if (resizeObserver) resizeObserver.disconnect();
    if (typingTimeout) clearTimeout(typingTimeout);
    window.removeEventListener('resize', onWindowResize);
  });

  function startTyping() {
    if (typingTimeout) clearTimeout(typingTimeout);
    displayedText = "";
    textIndex = 0;
    isTyping = true;
    typeNextChar();
  }

  function typeNextChar() {
    const currentText = showDebug ? debugText : text;
    if (textIndex < currentText.length) {
      displayedText += currentText[textIndex];
      textIndex++;
      typingTimeout = setTimeout(typeNextChar, 30);
    } else {
      isTyping = false;
    }
  }
  
  // Reactivity for debug
  $: if (showDebug) {
      displayedText = debugText;
  }

  function navigateTo(target) {
    if (isTransitioning) return;
    isTransitioning = true;
    
    // Define target position based on selection
    const targetObj = target === 'pluto' ? plutoMesh : charonMesh;
    if (!targetObj) return;

    const targetPos = new THREE.Vector3();
    targetObj.getWorldPosition(targetPos);
    
    // Calculate target camera position to match PlanetShowcase view
    // Pluto Showcase: CamDist 7.5, ModelX 5.35 (Right) -> Cam must be Left relative to Model
    // Charon Showcase: CamDist 4.0, ModelX -2.9 (Left) -> Cam must be Right relative to Model
    
    const currentCamPos = camera.position.clone();
    const direction = new THREE.Vector3().subVectors(currentCamPos, targetPos).normalize();
    const up = new THREE.Vector3(0, 1, 0);
    const right = new THREE.Vector3().crossVectors(direction, up).normalize();
    
    // Re-calculate direction to be perpendicular to right (to ensure flat plane mostly)
    const forward = new THREE.Vector3().crossVectors(up, right).normalize();
    
    let camDist, sideOffset;
    
    if (target === 'pluto') {
      camDist = 7.5;
      sideOffset = 5.35; // Positive means Model is Right, so Cam moves Left (-Right)
      // Wait, if Model is at +5.35, it is to the Right of center.
      // So Camera looks at Center.
      // Center = ModelPos - 5.35 * Right.
      // Camera = Center + 7.5 * Forward.
    } else {
      camDist = 4.0;
      sideOffset = -2.9; // Negative means Model is Left, so Cam moves Right (+Right)
    }
    
    // Calculate the "Center" point that the camera should look at
    // In Showcase, Camera looks at (0,0,0) and Model is at (sideOffset, 0, 0)
    // So Center is (ModelPos - sideOffset)
    const targetLookAt = targetPos.clone().sub(right.clone().multiplyScalar(sideOffset));
    
    // Camera position is back from the center
    const endCamPos = targetLookAt.clone().add(direction.multiplyScalar(camDist));
    const startTarget = controls.target.clone();
    
    const duration = 1200; // Slightly slower for better morph feel
    const startTime = performance.now();
    
    function animateTransition(time) {
      const elapsed = time - startTime;
      const progress = Math.min(elapsed / duration, 1);
      
      // Ease in out cubic
      const ease = progress < 0.5 ? 4 * progress * progress * progress : 1 - Math.pow(-2 * progress + 2, 3) / 2;
      
      camera.position.lerpVectors(startCamPos, endCamPos, ease);
      controls.target.lerpVectors(startTarget, targetLookAt, ease);
      
      // Trigger navigation earlier (at 60%) to overlap the scroll with the zoom
      // This creates a "morph" effect where we fly into the planet and it transforms into the showcase
      if (progress > 0.6 && !hasNavigated) {
        hasNavigated = true;
        dispatch('navigate', target);
      }
      
      if (progress < 1) {
        requestAnimationFrame(animateTransition);
      } else {
        // Reset state after a delay
        setTimeout(() => {
           isTransitioning = false;
           hasNavigated = false;
        }, 1000);
      }
    }
    
    requestAnimationFrame(animateTransition);
  }

  function init() {
    // Scene setup
    scene = new THREE.Scene();
    // Transparent background to blend with the deep black page
    scene.background = null;

    // Initial size might be 0 if hidden, but we set it anyway
    const width = container.clientWidth || 1;
    const height = container.clientHeight || 1;

    // Camera setup
    const fov = 45;
    const aspect = width / height;
    const near = 0.1;
    const far = 1000;
    camera = new THREE.PerspectiveCamera(fov, aspect, near, far);
    
    // Adjust camera distance based on screen aspect ratio to keep Charon in view
    const isPortrait = height > width;
    const distance = isPortrait ? 25 : 16;
    camera.position.set(0, isPortrait ? 8 : 5, distance);
    
    camera.lookAt(0, 0, 0);

    // Renderer setup
    renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
    renderer.setSize(width, height);
    renderer.setPixelRatio(window.devicePixelRatio);
    container.appendChild(renderer.domElement);

    // Controls
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.enableZoom = false;
    controls.enablePan = false;
    controls.minPolarAngle = Math.PI / 4;
    controls.maxPolarAngle = Math.PI * 0.75;

    // Lighting
    const ambientLight = new THREE.AmbientLight(0x404040, 0.5); // Soft white light
    scene.add(ambientLight);

    const sunLight = new THREE.DirectionalLight(0xffffff, 2);
    sunLight.position.set(10, 5, 10);
    scene.add(sunLight);

    // Barycenter Group
    barycenterGroup = new THREE.Group();
    scene.add(barycenterGroup);

    // Visualizing the Barycenter (The "invisible" point)
    const barycenterGeometry = new THREE.SphereGeometry(0.05, 16, 16);
    const barycenterMaterial = new THREE.MeshBasicMaterial({ color: 0xff3333, transparent: true, opacity: 0.8 });
    const barycenterMarker = new THREE.Mesh(barycenterGeometry, barycenterMaterial);
    scene.add(barycenterMarker);

    // Orbit Lines
    // Pluto Orbit (Radius ~1)
    const plutoOrbitGeo = new THREE.RingGeometry(0.98, 1.02, 64);
    const orbitMat = new THREE.MeshBasicMaterial({ color: 0xffffff, side: THREE.DoubleSide, transparent: true, opacity: 0.1 });
    const plutoOrbit = new THREE.Mesh(plutoOrbitGeo, orbitMat);
    plutoOrbit.rotation.x = Math.PI / 2;
    scene.add(plutoOrbit);

    // Charon Orbit (Radius ~8)
    const charonOrbitGeo = new THREE.RingGeometry(7.98, 8.02, 128);
    const charonOrbit = new THREE.Mesh(charonOrbitGeo, orbitMat);
    charonOrbit.rotation.x = Math.PI / 2;
    scene.add(charonOrbit);

    // Load Models
    const loader = new GLTFLoader();

    // Pluto
    loader.load('/3Dmodelle/pluto.glb', (gltf) => {
      plutoMesh = gltf.scene;
      // Scale adjustment if needed - assuming models are roughly unit scale or need normalization
      // I'll start with a guess and we might need to adjust
      plutoMesh.scale.set(2.8, 2.8, 2.8); 
      
      // Position Pluto slightly off-center (Barycenter is outside Pluto)
      // Pluto distance ~ 2000km from barycenter
      // Let's say 1 unit = 2000km roughly for visualization
      plutoMesh.position.set(-1, 0, 0); 
      barycenterGroup.add(plutoMesh);
      
      // Force compilation
      renderer.compile(scene, camera);
    }, undefined, (error) => {
      console.error('An error happened loading Pluto:', error);
    });

    // Charon
    loader.load('/3Dmodelle/charon.glb', (gltf) => {
      charonMesh = gltf.scene;
      // Charon is about half the size of Pluto
      charonMesh.scale.set(1.2, 1.2, 1.2);
      
      // Traverse the model to ensure all meshes are visible and have materials
      charonMesh.traverse((child) => {
        if (child.isMesh) {
          child.frustumCulled = false; // Prevent culling issues
          // Ensure material is double sided if needed
          if (child.material) {
            child.material.side = THREE.DoubleSide;
          }
        }
      });
      
      // Charon distance ~ 17000km from barycenter
      // If Pluto is at -1, Charon should be further out.
      // Ratio is roughly 1:8 for distances from barycenter (2000 vs 17000)
      charonMesh.position.set(8, 0, 0); 
      barycenterGroup.add(charonMesh);
      
      // Force compilation
      renderer.compile(scene, camera);
    }, undefined, (error) => {
      console.error('An error happened loading Charon:', error);
      // Fallback: Create a sphere if model fails to load
      const geometry = new THREE.SphereGeometry(0.5, 32, 32);
      const material = new THREE.MeshStandardMaterial({ 
        color: 0x888888,
        roughness: 0.8,
        metalness: 0.2
      });
      charonMesh = new THREE.Mesh(geometry, material);
      charonMesh.position.set(8, 0, 0);
      barycenterGroup.add(charonMesh);
    });

    window.addEventListener('resize', onWindowResize);
  }

  function onWindowResize() {
    if (!container || !renderer || !camera) return;
    
    const width = container.clientWidth;
    const height = container.clientHeight;
    
    if (width === 0 || height === 0) return;

    camera.aspect = width / height;
    camera.updateProjectionMatrix();
    renderer.setSize(width, height);

    // Update camera position for responsiveness
    const isPortrait = height > width;
    const distance = isPortrait ? 25 : 16;
    camera.position.set(0, isPortrait ? 8 : 5, distance);
    camera.lookAt(0, 0, 0);
  }

  function updateScreenPosition(obj, posStore) {
    if (!obj || !container) {
      posStore.visible = false;
      return;
    }

    const vector = new THREE.Vector3();
    // Get world position
    obj.getWorldPosition(vector);
    
    // Project to screen
    vector.project(camera);

    // Check if object is in front of camera
    if (vector.z > 1) {
      posStore.visible = false;
      return;
    }

    const x = (vector.x * .5 + .5) * container.clientWidth;
    const y = (-(vector.y * .5) + .5) * container.clientHeight;

    posStore.x = x;
    posStore.y = y;
    posStore.visible = true;
  }

  function animate() {
    animationId = requestAnimationFrame(animate);

    if (controls) controls.update();

    if (barycenterGroup && !isHovering && !activeInfo && !isTransitioning) {
      // Rotate the entire system around the barycenter (0,0,0)
      // This simulates the mutual orbit and tidal locking naturally
      barycenterGroup.rotation.y += 0.005;
    }

    if (plutoMesh) updateScreenPosition(plutoMesh, plutoScreenPosition);
    if (charonMesh) updateScreenPosition(charonMesh, charonScreenPosition);
    
    // Trigger Svelte reactivity for the positions
    plutoScreenPosition = plutoScreenPosition;
    charonScreenPosition = charonScreenPosition;

    renderer.render(scene, camera);
  }
</script>

<div class="scene-container" bind:this={container}></div>

<!-- Trackers for Interaction -->
{#if plutoScreenPosition.visible}
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <div 
    class="tracker"
    role="button"
    tabindex="0"
    style="transform: translate({plutoScreenPosition.x}px, {plutoScreenPosition.y}px);"
    on:mouseenter={() => { isHovering = true; activeInfo = 'pluto'; }}
    on:mouseleave={() => { isHovering = false; activeInfo = null; }}
    on:click|stopPropagation={() => navigateTo('pluto')}
  ></div>
{/if}

{#if charonScreenPosition.visible}
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <div 
    class="tracker"
    role="button"
    tabindex="0"
    style="transform: translate({charonScreenPosition.x}px, {charonScreenPosition.y}px);"
    on:mouseenter={() => { isHovering = true; activeInfo = 'charon'; }}
    on:mouseleave={() => { isHovering = false; activeInfo = null; }}
    on:click|stopPropagation={() => navigateTo('charon')}
  ></div>
{/if}

<div class="info-overlay" style="pointer-events: none;">
  {#if activeInfo === 'pluto'}
    <div class="info-card active" style="top: 20%; left: 10%; pointer-events: auto;">
      <h3>Pluto & Das Baryzentrum</h3>
      <p>Pluto und Charon kreisen um einen gemeinsamen Schwerpunkt im freien Raum (roter Punkt), ca. 900 km über Plutos Oberfläche. Das macht sie zu einem Doppelplanetensystem.</p>
    </div>
  {/if}

  {#if activeInfo === 'charon'}
    <div class="info-card active" style="bottom: 20%; right: 10%; pointer-events: auto;">
      <h3>Charon & Tidal Locking</h3>
      <p>Charon ist so massereich, dass er Pluto beeinflusst. Beide sind "tidal locked": Sie zeigen sich immer die gleiche Seite. Ihre Rotation dauert genau so lange wie ein Umlauf (6,4 Tage).</p>
    </div>
  {/if}
  
  <!-- Always visible hint -->
  {#if !activeInfo}
    <div class="hint-text">
      Hover für Infos • Klick zum Navigieren
    </div>
  {/if}
</div>

<!-- Typewriter Text Overlay -->
<div class="system-info" style="transform: translate(-50%, -50%) translate({showDebug ? debugTextOffsetX : textOffsetX}px, {showDebug ? debugTextOffsetY : textOffsetY}px) scale({showDebug ? debugTextScale : textScale});">
  <h2>{showDebug ? debugHeadline : headline}</h2>
  <p>{displayedText}</p>
</div>

<!-- Debug Panel -->
{#if showDebug}
  <div class="debug-panel">
    <h3>System Info Debugger</h3>
    
    <div class="control-group">
      <label>Headline</label>
      <input type="text" bind:value={debugHeadline} />
    </div>
    <div class="control-group">
      <label>Text</label>
      <textarea rows="6" bind:value={debugText} on:input={() => { textIndex = 0; displayedText = ""; typeNextChar(); }}></textarea>
    </div>

    <div class="control-group">
      <label>Scale: {debugTextScale}</label>
      <input type="range" min="0.5" max="2" step="0.1" bind:value={debugTextScale} />
    </div>
    <div class="control-group">
      <label>Offset X: {debugTextOffsetX}</label>
      <input type="range" min="-900" max="900" step="10" bind:value={debugTextOffsetX} />
    </div>
    <div class="control-group">
      <label>Offset Y: {debugTextOffsetY}</label>
      <input type="range" min="-500" max="500" step="10" bind:value={debugTextOffsetY} />
    </div>

    <div class="buttons">
      <button on:click={() => console.log({
        headline: debugHeadline, 
        text: debugText, 
        textScale: debugTextScale, 
        textOffsetX: debugTextOffsetX, 
        textOffsetY: debugTextOffsetY
      })}>Log</button>
      <button on:click={() => showDebug = false}>Close</button>
    </div>
  </div>
{/if}

<style>
  .scene-container {
    width: 100%;
    height: 100%;
    /* Ensure it takes up space */
    position: absolute;
    top: 0;
    left: 0;
  }

  .tracker {
    position: absolute;
    top: 0;
    left: 0;
    width: 100px; /* Size of the hit area */
    height: 100px;
    margin-left: -50px; /* Center it */
    margin-top: -50px;
    border-radius: 50%;
    /* background: rgba(255, 0, 0, 0.2); Debugging */
    z-index: 20;
    cursor: pointer;
  }

  .info-overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 30;
  }

  .info-card {
    position: absolute;
    background: rgba(0, 0, 0, 0.8);
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.3);
    padding: 1.5rem;
    border-radius: 16px;
    max-width: 350px;
    color: white;
    box-shadow: white;
    animation: fadeIn 0.8s ease-out;
  }

  .info-card h3 {
    margin: 0 0 0.5rem 0;
    font-size: 1.4rem;
    color: white; /* Light blue accent */
    text-transform: uppercase;
    letter-spacing: 1px;
  }

  .info-card p {
    margin: 0;
    font-size: 1rem;
    line-height: 1.6;
    color: #e0e0e0;
  }
  
  .hint-text {
    position: absolute;
    bottom: 50px;
    width: 100%;
    text-align: center;
    color: rgba(255, 255, 255, 0.5);
    font-size: 0.9rem;
    letter-spacing: 2px;
    text-transform: uppercase;
    animation: pulse 2s infinite;
  }

  .system-info {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 500px;
    pointer-events: none;
    text-align: center;
    z-index: 25; /* Below info-card (30) but above scene */
    /* Glass effect */
    background: rgba(0, 0, 0, 0.4);
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
    padding: 2.5rem;
    border-radius: 24px;
    border: 1px solid rgba(255, 255, 255, 0.1);
    box-shadow: 0 4px 30px rgba(0, 0, 0, 0.1);
  }
  
  .system-info h2 {
    font-family: 'Helvetica', sans-serif;
    font-weight: 700;
    font-size: 2.5rem;
    margin: 0 0 1.5rem 0;
    background: linear-gradient(to right, #fff, #aaa);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    letter-spacing: -0.02em;
  }
  
  .system-info p {
    font-family: 'Helvetica', sans-serif;
    font-weight: 700;
    font-size: 1.1rem;
    line-height: 1.6;
    color: rgba(255, 255, 255, 0.9);
    margin: 0;
    min-height: 4em;
  }

  @keyframes fadeIn {
    from { opacity: 0; transform: translateY(10px); }
    to { opacity: 1; transform: translateY(0); }
  }
  
  @keyframes pulse {
    0% { opacity: 0.3; }
    50% { opacity: 0.7; }
    100% { opacity: 0.3; }
  }

  @media (max-width: 768px) {
    .info-card {
      top: auto !important;
      bottom: 10% !important;
      left: 5% !important;
      right: 5% !important;
      max-width: none;
    }
  }

  .debug-panel {
    position: absolute;
    top: 20px;
    right: 20px;
    background: rgba(0, 0, 0, 0.9);
    border: 1px solid #333;
    border-radius: 8px;
    padding: 15px;
    width: 300px;
    z-index: 1000;
    color: white;
    font-family: 'Helvetica', sans-serif;
    font-weight: 700;
  }

  .debug-panel h3 {
    margin: 0 0 15px 0;
    font-size: 1.1rem;
    text-align: center;
    border-bottom: 1px solid #333;
    padding-bottom: 10px;
  }

  .control-group {
    margin-bottom: 12px;
    display: flex;
    flex-direction: column;
    gap: 5px;
  }

  .control-group label {
    font-size: 0.8rem;
    color: #aaa;
  }

  .control-group input[type="range"] {
    width: 100%;
    accent-color: #a855f7;
  }
  
  .control-group textarea {
    width: 100%;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.3);
    color: white;
    border-radius: 4px;
    padding: 5px;
    font-family: inherit;
    font-size: 12px;
    resize: vertical;
  }
  
  .control-group input[type="text"] {
    width: 100%;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.3);
    color: white;
    border-radius: 4px;
    padding: 5px;
    font-family: inherit;
    font-size: 12px;
  }

  .buttons {
    display: flex;
    gap: 10px;
    margin-top: 15px;
  }

  .buttons button {
    flex: 1;
    padding: 8px;
    background: #1a1a1a;
    border: 1px solid #333;
    color: white;
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.2s;
  }

  .buttons button:hover {
    background: #333;
    border-color: #555;
  }
</style>

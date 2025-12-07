<script>
  import { onMount, onDestroy } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

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

  onMount(() => {
    init();
    animate();
    
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
    window.removeEventListener('resize', onWindowResize);
  });

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
    const distance = isPortrait ? 35 : 22;
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
      plutoMesh.scale.set(1, 1, 1); 
      
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
      charonMesh.scale.set(0.5, 0.5, 0.5);
      
      // Charon distance ~ 17000km from barycenter
      // If Pluto is at -1, Charon should be further out.
      // Ratio is roughly 1:8 for distances from barycenter (2000 vs 17000)
      charonMesh.position.set(8, 0, 0); 
      barycenterGroup.add(charonMesh);
      
      // Force compilation
      renderer.compile(scene, camera);
    }, undefined, (error) => {
      console.error('An error happened loading Charon:', error);
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
    const distance = isPortrait ? 35 : 22;
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

    if (barycenterGroup && !isHovering && !activeInfo) {
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

  function handleTrackerClick(target) {
    if (activeInfo === target) {
      activeInfo = null;
    } else {
      activeInfo = target;
    }
  }
</script>

<div class="scene-container" bind:this={container} on:click={() => activeInfo = null}></div>

<!-- Trackers for Interaction -->
{#if plutoScreenPosition.visible}
  <div 
    class="tracker"
    style="transform: translate({plutoScreenPosition.x}px, {plutoScreenPosition.y}px);"
    on:mouseenter={() => isHovering = true}
    on:mouseleave={() => isHovering = false}
    on:click|stopPropagation={() => handleTrackerClick('pluto')}
  ></div>
{/if}

{#if charonScreenPosition.visible}
  <div 
    class="tracker"
    style="transform: translate({charonScreenPosition.x}px, {charonScreenPosition.y}px);"
    on:mouseenter={() => isHovering = true}
    on:mouseleave={() => isHovering = false}
    on:click|stopPropagation={() => handleTrackerClick('charon')}
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
      Klicke auf Pluto oder Charon für Details
    </div>
  {/if}
</div>

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
</style>

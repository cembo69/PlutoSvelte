<script>
  import { onMount, onDestroy } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
  import Loupe from './Loupe.svelte';
  import Folder from './Folder.svelte';

  export let modelPath;
  export let headline;
  export let text;
  export let scale = 1;
  export let rotationSpeed = 0.002;
  export let isActive = false;
  export let alignment = 'center'; // 'left' | 'right' | 'center'
  export let cameraDistance = 5;
  export let modelOffsetX = 0;
  export let modelOffsetY = 0;
  export let textOffsetX = 471;
  export let textOffsetY = 0;
  export let textScale = 0.7;
  export let initialRotationX = 0; // degrees
  export let initialRotationY = 0; // degrees
  export let images = []; // Array of image paths
  export let miniModelPath = null; // Path to a secondary "cute" model
  export let loupeConfig = {}; // { bottom, right, size, mag, handleRotation }

  // Interaction State
  let isDragging = false;
  let previousMousePosition = { x: 0, y: 0 };
  let additionalRotationX = 0;
  let additionalRotationY = 0;

  // Debugging
  let showDebug = false;
  
  // Text Layout Debugging
  let debugTextOffsetX = 0;
  let debugTextOffsetY = 0;
  let debugTextScale = 1;
  
  // Model Debugging
  let debugScale = scale;
  let debugModelOffsetX = modelOffsetX;
  let debugModelOffsetY = modelOffsetY;
  let debugCamDist = cameraDistance;
  let debugRotationX = initialRotationX;
  let debugRotationY = initialRotationY;
  
  // Mini Model Debugging
  let showMiniDebug = false; // Separate toggle for mini model
  let debugMiniScale = 0.4;
  let debugMiniX = 1.3;
  let debugMiniY = -1;
  let debugMiniZ = 2.5;

  // Loupe Debugging
  // Default values (Pluto)
  const defaultLoupe = {
    bottom: 412,
    right: 1044,
    size: 358,
    mag: 1.5,
    handleRotation: -109
  };

  $: currentLoupeConfig = { ...defaultLoupe, ...loupeConfig };

  let debugLoupeBottom = defaultLoupe.bottom;
  let debugLoupeRight = defaultLoupe.right;
  let debugLoupeSize = defaultLoupe.size;
  let debugLoupeMag = defaultLoupe.mag; 
  let debugLoupeHandleRotation = defaultLoupe.handleRotation;

  // Sync debug values with props when not debugging
  $: if (!showDebug) {
      debugLoupeBottom = currentLoupeConfig.bottom;
      debugLoupeRight = currentLoupeConfig.right;
      debugLoupeSize = currentLoupeConfig.size;
      debugLoupeMag = currentLoupeConfig.mag;
      debugLoupeHandleRotation = currentLoupeConfig.handleRotation;
  }

  // Lighting Debugging
  let debugAmbientIntensity = 1.5;
  let debugDirectionalIntensity = 2;
  let debugBackLightIntensity = 1;
  
  let ambientLight, directionalLight, backLight;

  // Text Content Debugging
  let debugHeadline = headline;
  let debugText = text;

  // Folder Debugging
  let debugFolderMarginTop = 17;
  let debugFolderOffsetX = -92;
  let debugFolderOffsetY = 154;

  let container;
  let renderer, scene, camera;
  let animationId;
  let model;
  let miniModel;
  let resizeObserver;
  
  // Responsive Logic
  let windowWidth = 1920;
  let windowHeight = 1080;
  // Reference resolution (Ultra Wide) where the offsets were tuned
  const referenceWidth = 3440; 
  const referenceAspect = 21 / 9;

  $: widthRatio = Math.min(1, windowWidth / referenceWidth); // Don't scale up, only down
  $: currentAspect = windowWidth / windowHeight;
  
  // Scale text offsets based on screen width
  $: responsiveTextX = (showDebug ? debugTextOffsetX : textOffsetX) * widthRatio;
  $: responsiveTextY = (showDebug ? debugTextOffsetY : textOffsetY) * widthRatio;
  
  // Scale model offset based on aspect ratio to keep it in frame
  // If screen gets narrower (aspect decreases), we need to pull the model closer to center
  $: aspectFactor = Math.min(1, currentAspect / referenceAspect);
  $: responsiveModelX = (showDebug ? debugModelOffsetX : modelOffsetX) * aspectFactor;

  // Typewriter effect variables
  let displayedText = "";
  let isTyping = false;
  let textIndex = 0;
  let typingTimeout;
  let wasActive = false;

  $: if (isActive !== wasActive) {
    if (isActive) {
      // Reset debug text to prop text on activation if not debugging
      if (!showDebug) {
        debugHeadline = headline;
        debugText = text;
      }
      startTyping();
    } else {
      clearTimeout(typingTimeout);
      displayedText = "";
      isTyping = false;
    }
    wasActive = isActive;
  }

  function startTyping() {
    if (isTyping) return;
    isTyping = true;
    displayedText = "";
    textIndex = 0;
    typeNextChar();
  }

  function typeNextChar() {
    const currentText = showDebug ? debugText : text;
    if (textIndex < currentText.length) {
      displayedText += currentText.charAt(textIndex);
      textIndex++;
      typingTimeout = setTimeout(typeNextChar, 30); // Typing speed
    } else {
      isTyping = false;
    }
  }

  onMount(() => {
    initThree();
    window.addEventListener('resize', handleResize);
    handleResize(); // Initial check
    
    // Initialize debug values
    debugTextOffsetX = textOffsetX;
    debugTextOffsetY = textOffsetY;
    debugTextScale = textScale;
    debugScale = scale;
    debugModelOffsetX = modelOffsetX;
    debugModelOffsetY = modelOffsetY;
    debugCamDist = cameraDistance;
    debugRotationX = initialRotationX;
    debugRotationY = initialRotationY;
  });

  onDestroy(() => {
    if (animationId) cancelAnimationFrame(animationId);
    window.removeEventListener('resize', handleResize);
    if (renderer) renderer.dispose();
    if (scene) {
      scene.traverse((object) => {
        if (object.isMesh) {
          object.geometry.dispose();
          if (object.material.isMaterial) {
            object.material.dispose();
          }
        }
      });
    }
  });

  function handleResize() {
    if (!container || !camera || !renderer) return;
    
    windowWidth = window.innerWidth;
    windowHeight = window.innerHeight;
    
    const width = container.clientWidth;
    const height = container.clientHeight;
    
    camera.aspect = width / height;
    camera.updateProjectionMatrix();
    renderer.setSize(width, height);
  }

  function initThree() {
    scene = new THREE.Scene();
    
    // Camera setup
    camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 1000);
    camera.position.z = cameraDistance;

    // Renderer setup
    renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
    renderer.setSize(container.clientWidth, container.clientHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.outputColorSpace = THREE.SRGBColorSpace; // Fix for dark colors
    container.appendChild(renderer.domElement);

    // Lighting
    ambientLight = new THREE.AmbientLight(0xffffff, debugAmbientIntensity);
    scene.add(ambientLight);

    directionalLight = new THREE.DirectionalLight(0xffffff, debugDirectionalIntensity);
    directionalLight.position.set(5, 3, 5);
    scene.add(directionalLight);
    
    // Backlight for rim effect
    backLight = new THREE.DirectionalLight(0x4444ff, debugBackLightIntensity);
    backLight.position.set(-5, 0, -5);
    scene.add(backLight);

    // Load Model
    const loader = new GLTFLoader();
    loader.load(modelPath, (gltf) => {
      model = gltf.scene;
      
      // Center the model
      const box = new THREE.Box3().setFromObject(model);
      const center = box.getCenter(new THREE.Vector3());
      model.position.sub(center);
      
      scene.add(model);
      
      // Initial scale and position
      updateModelTransform();
      
    }, undefined, (error) => {
      console.error('An error happened loading the model:', error);
    });

    // Load Mini Model (if provided)
    if (miniModelPath) {
      loader.load(miniModelPath, (gltf) => {
        miniModel = gltf.scene;
        
        // Center the mini model
        const box = new THREE.Box3().setFromObject(miniModel);
        const center = box.getCenter(new THREE.Vector3());
        miniModel.position.sub(center);
        
        scene.add(miniModel);
        updateMiniModelTransform();
      }, undefined, (error) => {
        console.error('Error loading mini model:', error);
      });
    }

    animate();
  }

  function updateModelTransform() {
    if (!model) return;
    
    const currentScale = showDebug ? debugScale : scale;
    const currentOffsetX = responsiveModelX;
    const currentOffsetY = showDebug ? debugModelOffsetY : modelOffsetY;
    
    model.scale.set(currentScale, currentScale, currentScale);
    model.position.x = currentOffsetX / 100; // Convert to Three.js units approx
    model.position.y = currentOffsetY / 100;
  }

  function updateMiniModelTransform() {
    if (!miniModel) return;
    
    miniModel.scale.set(debugMiniScale, debugMiniScale, debugMiniScale);
    miniModel.position.set(debugMiniX, debugMiniY, debugMiniZ);
  }

  function animate() {
    animationId = requestAnimationFrame(animate);
    
    if (model) {
      if (rotationSpeed !== 0 && !showDebug && !isDragging && additionalRotationX === 0 && additionalRotationY === 0) {
        model.rotation.y += rotationSpeed;
      } else {
        // Apply fixed rotation + interaction rotation
        const baseRotX = showDebug ? debugRotationX : initialRotationX;
        const baseRotY = showDebug ? debugRotationY : initialRotationY;
        
        // Convert degrees to radians
        model.rotation.x = (baseRotX + additionalRotationX) * (Math.PI / 180);
        model.rotation.y = (baseRotY + additionalRotationY) * (Math.PI / 180);
      }
      updateModelTransform();
    }

    if (miniModel) {
      // Cute animation: Rotate and bob
      miniModel.rotation.y += 0.01;
      miniModel.rotation.z = Math.sin(Date.now() * 0.002) * 0.1;
      miniModel.position.y = debugMiniY + Math.sin(Date.now() * 0.003) * 0.1;
      
      if (showDebug) updateMiniModelTransform();
    }
    
    if (camera) {
      camera.position.z = showDebug ? debugCamDist : cameraDistance;
    }
    
    renderer.render(scene, camera);
  }
  
  function toggleDebug() {
    showDebug = !showDebug;
    showMiniDebug = false; // Close mini debug when opening main debug
    // Sync values when opening debug
    if (showDebug) {
      debugTextOffsetX = textOffsetX;
      debugTextOffsetY = textOffsetY;
      debugTextScale = textScale;
      debugScale = scale;
      debugModelOffsetX = modelOffsetX;
      debugModelOffsetY = modelOffsetY;
      debugCamDist = cameraDistance;
      debugRotationX = initialRotationX;
      debugRotationY = initialRotationY;
      debugHeadline = headline;
      debugText = text;
    }
  }

  function toggleMiniDebug() {
    showMiniDebug = !showMiniDebug;
    showDebug = false; // Close main debug when opening mini debug
  }
  
  function logSettings() {
    console.log({
      scale: debugScale,
      cameraDistance: debugCamDist,
      modelOffsetX: debugModelOffsetX,
      modelOffsetY: debugModelOffsetY,
      textOffsetX: debugTextOffsetX,
      textOffsetY: debugTextOffsetY,
      textScale: debugTextScale,
      initialRotationX: debugRotationX,
      initialRotationY: debugRotationY,
      headline: debugHeadline,
      text: debugText
    });
    alert("Settings logged to console! Copy them to your config.");
  }

  // Interaction Handlers
  function handleMouseDown(event) {
    if (!isActive) return;
    isDragging = true;
    previousMousePosition = { x: event.clientX, y: event.clientY };
  }

  function handleMouseMove(event) {
    if (!isDragging || !isActive) return;
    
    const deltaMove = {
      x: event.clientX - previousMousePosition.x,
      y: event.clientY - previousMousePosition.y
    };
    
    // Adjust rotation sensitivity here
    additionalRotationY += deltaMove.x * 0.5;
    additionalRotationX += deltaMove.y * 0.5;
    
    previousMousePosition = { x: event.clientX, y: event.clientY };
  }

  function handleMouseUp() {
    isDragging = false;
  }

  $: if (miniModel) updateMiniModelTransform();
  
  // Update lights when debug values change
  $: if (ambientLight) ambientLight.intensity = debugAmbientIntensity;
  $: if (directionalLight) directionalLight.intensity = debugDirectionalIntensity;
  $: if (backLight) backLight.intensity = debugBackLightIntensity;

</script>

<div class="showcase-container" class:active={isActive}
  on:mousedown={handleMouseDown}
  on:mousemove={handleMouseMove}
  on:mouseup={handleMouseUp}
  on:mouseleave={handleMouseUp}
  role="application"
  aria-label="Interactive 3D Model Viewer"
>
  <div class="content-wrapper" style="
    --text-offset-x: {responsiveTextX}px;
    --text-offset-y: {responsiveTextY}px;
    --text-scale: {showDebug ? debugTextScale : textScale};
  ">
    <div class="model-container" bind:this={container}></div>
    
    <div class="text-container {alignment}" style="text-align: left;">
      <h1>{showDebug ? debugHeadline : headline}</h1>
      <p class="typewriter">{displayedText}<span class="cursor" class:typing={isTyping}>|</span></p>
      
      {#if images && images.length > 0}
        <div class="folder-container" style="
          margin-top: {showDebug ? debugFolderMarginTop : 50}px;
          transform: translate({showDebug ? debugFolderOffsetX : -92}px, {showDebug ? debugFolderOffsetY : 154}px);
        ">
          <Folder 
            images={images} 
            size={2} 
            color="#5227FF" 
            className="custom-folder" 
          />
        </div>
      {/if}
    </div>
  </div>
  
  <!-- Loupe / Magnifying Glass -->
  <Loupe 
    {scene} 
    mainCamera={camera} 
    isActive={isActive}
    bottom={debugLoupeBottom}
    right={debugLoupeRight}
    size={debugLoupeSize}
    handleRotation={debugLoupeHandleRotation}
    magnification={debugLoupeMag}
  />

  <!-- Debug Toggle Button (Always visible when active) -->
  {#if isActive}
    <div class="debug-controls">
      <button class="debug-toggle" on:click={() => showDebug = !showDebug}>
        {showDebug ? 'Close Debug' : 'Debug'}
      </button>
      {#if showDebug && miniModelPath}
        <button class="debug-toggle" on:click={() => showMiniDebug = !showMiniDebug}>
          {showMiniDebug ? 'Close Mini Debug' : 'Mini Debug'}
        </button>
      {/if}
    </div>
  {/if}
  
  <!-- Debug Panel -->
  {#if showDebug && isActive}
    <div class="debug-panel">
      <h3>Showcase Debugger</h3>
      
      <div class="section">
        <h4>Position & Size</h4>
        <label>
          Bottom: {debugLoupeBottom}px
          <input type="range" min="-500" max="1500" bind:value={debugLoupeBottom} />
        </label>
        <label>
          Right: {debugLoupeRight}px
          <input type="range" min="-500" max="1500" bind:value={debugLoupeRight} />
        </label>
        <label>
          Size: {debugLoupeSize}px
          <input type="range" min="50" max="1000" bind:value={debugLoupeSize} />
        </label>
        <label>
          Handle Rotation: {debugLoupeHandleRotation}°
          <input type="range" min="-180" max="180" bind:value={debugLoupeHandleRotation} />
        </label>
      </div>
      
      <div class="section">
        <h4>Text Position</h4>
        <label>Offset X: {debugTextOffsetX}px <input type="range" min="-1000" max="1000" bind:value={debugTextOffsetX} /></label>
        <label>Offset Y: {debugTextOffsetY}px <input type="range" min="-1000" max="1000" bind:value={debugTextOffsetY} /></label>
        <label>Scale: {debugTextScale} <input type="range" min="0.1" max="3" step="0.1" bind:value={debugTextScale} /></label>
      </div>

      <div class="section">
        <h4>Magnification</h4>
        <label>
          Factor (e.g. 2x, 3x): {debugLoupeMag}
          <input type="range" min="1.5" max="10" step="0.5" bind:value={debugLoupeMag} />
        </label>
      </div>

      <div class="section">
        <h4>Folder Position</h4>
        <label>Margin Top: {debugFolderMarginTop}px <input type="range" min="0" max="200" bind:value={debugFolderMarginTop} /></label>
        <label>Offset X: {debugFolderOffsetX}px <input type="range" min="-200" max="200" bind:value={debugFolderOffsetX} /></label>
        <label>Offset Y: {debugFolderOffsetY}px <input type="range" min="-200" max="200" bind:value={debugFolderOffsetY} /></label>
      </div>
      
      <button class="log-btn" on:click={() => console.log({
        loupeBottom: debugLoupeBottom,
        loupeRight: debugLoupeRight,
        loupeSize: debugLoupeSize,
        loupeMag: debugLoupeMag,
        loupeHandleRotation: debugLoupeHandleRotation,
        folderMarginTop: debugFolderMarginTop,
        folderOffsetX: debugFolderOffsetX,
        folderOffsetY: debugFolderOffsetY,
        textOffsetX: debugTextOffsetX,
        textOffsetY: debugTextOffsetY,
        textScale: debugTextScale
      })}>Log Settings</button>
    </div>
  {/if}
  <!-- Mini Model Debug Panel -->
  {#if showMiniDebug && isActive && miniModelPath}
    <div class="debug-panel" style="right: 340px;">
      <h3>Mini Model Debugger</h3>
      <div class="section">
        <h4>Position & Scale</h4>
        <label>Scale: {debugMiniScale}<input type="range" min="0.1" max="2" step="0.1" bind:value={debugMiniScale} /></label>
        <label>X: {debugMiniX}<input type="range" min="-5" max="5" step="0.1" bind:value={debugMiniX} /></label>
        <label>Y: {debugMiniY}<input type="range" min="-5" max="5" step="0.1" bind:value={debugMiniY} /></label>
        <label>Z: {debugMiniZ}<input type="range" min="-5" max="10" step="0.1" bind:value={debugMiniZ} /></label>
      </div>
      <div class="section">
        <h4>Lighting</h4>
        <label>Ambient: {debugAmbientIntensity}<input type="range" min="0" max="5" step="0.1" bind:value={debugAmbientIntensity} /></label>
        <label>Directional: {debugDirectionalIntensity}<input type="range" min="0" max="5" step="0.1" bind:value={debugDirectionalIntensity} /></label>
        <label>Backlight: {debugBackLightIntensity}<input type="range" min="0" max="5" step="0.1" bind:value={debugBackLightIntensity} /></label>
      </div>
      <button class="log-btn" on:click={() => console.log({
        miniScale: debugMiniScale, 
        miniX: debugMiniX, 
        miniY: debugMiniY, 
        miniZ: debugMiniZ,
        ambient: debugAmbientIntensity,
        directional: debugDirectionalIntensity,
        backlight: debugBackLightIntensity
      })}>Log Mini Settings</button>
    </div>
  {/if}
</div>

<style>
  .showcase-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    opacity: 0;
    pointer-events: none;
    transition: opacity 1s ease;
    overflow: hidden;
  }

  .showcase-container.active {
    opacity: 1;
    pointer-events: auto;
    cursor: grab;
  }

  .showcase-container.active:active {
    cursor: grabbing;
  }

  .content-wrapper {
    position: relative;
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .model-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 1;
  }

  .text-container {
    position: absolute;
    z-index: 2;
    width: 400px;
    color: white;
    text-shadow: 0 2px 10px rgba(0,0,0,0.8);
    transform: translate(var(--text-offset-x), var(--text-offset-y)) scale(var(--text-scale));
    pointer-events: none; /* Let clicks pass through to model if needed */
  }
  
  /* Alignment presets (can be overridden by offsets) */
  .text-container.left {
    left: 10%;
    text-align: left;
  }
  
  .text-container.right {
    right: 10%;
    text-align: right;
  }
  
  .text-container.center {
    text-align: center;
  }

  h1 {
    font-family: 'Helvetica', sans-serif;
    font-size: 5rem;
    margin: 0 0 1rem 0;
    font-weight: 700;
    letter-spacing: -2px;
    line-height: 1;
  }

  p {
    font-family: 'Helvetica', sans-serif;
    font-size: 1.2rem;
    line-height: 1.6;
    margin: 0;
    opacity: 0.9;
    font-weight: 700;
    white-space: pre-wrap; /* Preserve line breaks */
  }
  
  .cursor {
    display: inline-block;
    width: 2px;
    background-color: white;
    animation: blink 1s infinite;
    margin-left: 2px;
    vertical-align: text-bottom;
  }
  
  .cursor.typing {
    animation: none;
    opacity: 1;
  }
  
  @keyframes blink {
    0%, 100% { opacity: 1; }
    50% { opacity: 0; }
  }
  
  /* Image Gallery Removed - Replaced by Folder */
  .folder-container {
    margin-top: 50px;
    height: 200px; /* Reserve space for open folder */
    display: flex;
    justify-content: center; /* Center the folder */
    align-items: center;
    pointer-events: auto; /* Enable interaction */
  }

  /* Loupe Styles Removed - Moved to Loupe.svelte */
  
  /* Debug UI */
  .debug-controls {
    position: absolute;
    bottom: 20px;
    right: 20px;
    z-index: 1000;
    display: flex;
    gap: 10px;
  }

  .debug-toggle {
    padding: 8px 16px;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.3);
    color: white;
    border-radius: 4px;
    cursor: pointer;
    font-family: 'Helvetica', sans-serif;
    font-weight: 700;
  }
  
  .debug-toggle:hover {
    background: rgba(255, 255, 255, 0.2);
  }
  
  .debug-panel {
    position: absolute;
    top: 20px;
    right: 20px;
    width: 300px;
    background: rgba(0, 0, 0, 0.85);
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 8px;
    padding: 15px;
    z-index: 999;
    color: white;
    max-height: 90vh;
    overflow-y: auto;
    font-family: 'Helvetica', sans-serif;
    font-weight: 700;
  }
  
  .debug-panel h3 {
    margin-top: 0;
    border-bottom: 1px solid rgba(255, 255, 255, 0.2);
    padding-bottom: 10px;
  }
  
  .debug-panel h4 {
    margin: 10px 0 5px 0;
    color: #aaa;
    font-size: 0.9em;
  }
  
  .section {
    margin-bottom: 15px;
    padding-bottom: 15px;
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  }
  
  .debug-panel label {
    display: block;
    font-size: 0.8em;
    margin-bottom: 8px;
    color: #ccc;
  }
  
  .debug-panel input[type="range"] {
    width: 100%;
    margin-top: 4px;
  }
  
  .debug-panel input[type="text"],
  .debug-panel textarea {
    width: 100%;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.3);
    color: white;
    padding: 5px;
    border-radius: 4px;
    margin-top: 4px;
  }
  
  .log-btn {
    width: 100%;
    padding: 8px;
    background: #4CAF50;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-weight: bold;
  }
  
  .log-btn:hover {
    background: #45a049;
  }
</style>

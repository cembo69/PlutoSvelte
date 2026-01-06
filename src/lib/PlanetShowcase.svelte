<script>
  import { onMount, onDestroy, createEventDispatcher } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

  const dispatch = createEventDispatcher();

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
  export let textOffsetX = 0;
  export let textOffsetY = 0;
  export let textScale = 1;

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
  
  // Text Content Debugging
  let debugHeadline = headline;
  let debugText = text;

  let container;
  let renderer, scene, camera;
  let animationId;
  let model;
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
      // animateEntrance(); // Disabled for smoother morph from system view
    } else {
      clearTimeout(typingTimeout);
      displayedText = "";
      isTyping = false;
    }
    wasActive = isActive;
  }

  function animateEntrance() {
    if (!model) return;
    
    // Start slightly smaller and zoom in to final scale
    const startScale = scale * 0.8;
    const endScale = scale;
    
    const duration = 1000;
    const startTime = performance.now();
    
    function animate(time) {
      const elapsed = time - startTime;
      const progress = Math.min(elapsed / duration, 1);
      
      // Ease out cubic
      const ease = 1 - Math.pow(1 - progress, 3);
      
      const currentScale = startScale + (endScale - startScale) * ease;
      model.scale.set(currentScale, currentScale, currentScale);
      
      if (progress < 1) {
        requestAnimationFrame(animate);
      }
    }
    
    requestAnimationFrame(animate);
  }

  // Update displayed text immediately when debugging
  $: if (showDebug) {
    displayedText = debugText;
  }

  // Model Reactivity (using props now)
  $: if (model) {
    const currentScale = showDebug ? debugScale : scale;
    model.scale.set(currentScale, currentScale, currentScale);
    
    // Re-apply alignment logic with props
    const dist = showDebug ? debugCamDist : cameraDistance;
    const offset = dist * 0.3;
    let baseX = 0;
    if (alignment === 'left') baseX = offset;
    else if (alignment === 'right') baseX = -offset;
    
    // Apply responsive scaling to the base offset too (since FOV is horizontal)
    baseX *= aspectFactor;
    
    model.position.x = baseX + responsiveModelX;
    model.position.y = showDebug ? debugModelOffsetY : modelOffsetY;
  }

  $: if (camera) {
    camera.position.z = showDebug ? debugCamDist : cameraDistance;
  }

  onMount(() => {
    // Initialize debug values
    debugTextOffsetX = textOffsetX;
    debugTextOffsetY = textOffsetY;
    debugTextScale = textScale;
    debugHeadline = headline;
    debugText = text;
    debugScale = scale;
    debugModelOffsetX = modelOffsetX;
    debugModelOffsetY = modelOffsetY;
    debugCamDist = cameraDistance;
    
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
    if (!isActive) return;
    // Use debugText instead of text prop
    if (textIndex < debugText.length) {
      displayedText += debugText[textIndex];
      textIndex++;
      typingTimeout = setTimeout(typeNextChar, 30); // Faster typing speed
    } else {
      isTyping = false;
    }
  }

  function init() {
    scene = new THREE.Scene();
    scene.background = null; 

    // Initial size might be 0 if hidden, but we set it anyway
    const width = container.clientWidth || 1;
    const height = container.clientHeight || 1;

    const fov = 45;
    const aspect = width / height;
    const near = 0.01; // Lower near plane to prevent clipping when close
    const far = 2000; // Increase far plane just in case
    camera = new THREE.PerspectiveCamera(fov, aspect, near, far);
    camera.position.set(0, 0, cameraDistance);
    camera.lookAt(0, 0, 0);

    renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
    renderer.setSize(width, height);
    renderer.setPixelRatio(window.devicePixelRatio);
    container.appendChild(renderer.domElement);

    const ambientLight = new THREE.AmbientLight(0x404040, 0.8); 
    scene.add(ambientLight);

    const sunLight = new THREE.DirectionalLight(0xffffff, 2.5);
    sunLight.position.set(5, 3, 5);
    scene.add(sunLight);

    const loader = new GLTFLoader();
    loader.load(modelPath, (gltf) => {
      model = gltf.scene;
      model.scale.set(scale, scale, scale);
      
      // Center the model
      const box = new THREE.Box3().setFromObject(model);
      const center = box.getCenter(new THREE.Vector3());
      model.position.sub(center);
      
      // Traverse to disable frustum culling to prevent accidental clipping
      model.traverse((child) => {
        if (child.isMesh) {
          child.frustumCulled = false;
        }
      });
      
      // Apply alignment offset based on camera distance to ensure it stays in view
      // Factor 0.3 ensures the model is to the side but not cut off
      const offset = cameraDistance * 0.3;
      
      if (alignment === 'left') {
        model.position.x = offset; // Move model to right
      } else if (alignment === 'right') {
        model.position.x = -offset; // Move model to left
      }
      
      scene.add(model);
      
      // Force compilation to avoid lag when it becomes visible
      renderer.compile(scene, camera);
      
    }, undefined, (error) => {
      console.error(`Error loading ${modelPath}:`, error);
    });

    window.addEventListener('resize', onWindowResize);
  }

  function onWindowResize() {
    if (!container || !renderer || !camera) return;
    
    windowWidth = container.clientWidth;
    windowHeight = container.clientHeight;
    
    if (windowWidth === 0 || windowHeight === 0) return;

    camera.aspect = windowWidth / windowHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(windowWidth, windowHeight);
  }


  function animate() {
    animationId = requestAnimationFrame(animate);
    
    if (model) {
      model.rotation.y += rotationSpeed;
    }

    // Check for resize need (robustness against layout thrashing or missed events)
    if (container && renderer) {
      const width = container.clientWidth;
      const height = container.clientHeight;
      const canvas = renderer.domElement;
      
      // Check if canvas size matches container size (accounting for pixel ratio)
      if (width > 0 && height > 0 && (Math.abs(canvas.width - Math.floor(width * window.devicePixelRatio)) > 2 || Math.abs(canvas.height - Math.floor(height * window.devicePixelRatio)) > 2)) {
        onWindowResize();
      }
    }

    renderer.render(scene, camera);
  }
</script>

<div class="showcase-container {alignment}">
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <div 
    class="model-view" 
    bind:this={container}
    on:click={() => dispatch('click')}
    role="button"
    tabindex="0"
    style="cursor: pointer;"
  ></div>
  
  <div class="content-panel" style="opacity: {isActive ? 1 : 0}; transition: opacity 0.2s; transform: translate({responsiveTextX}px, {responsiveTextY}px) scale({showDebug ? debugTextScale : textScale});">
    <h2 class="typewriter-headline">{showDebug ? debugHeadline : headline}</h2>
    <div class="typewriter-text">
      {displayedText}
    </div>
  </div>

  <!-- Debug Toggle Button -->
  {#if isActive}
    <button 
      style="position: absolute; bottom: 20px; right: 20px; z-index: 10000; padding: 8px 16px; background: rgba(255,255,255,0.1); border: 1px solid rgba(255,255,255,0.3); color: white; border-radius: 4px; cursor: pointer;"
      on:click|stopPropagation={() => showDebug = !showDebug}
    >
      {showDebug ? 'Close Debug' : 'Debug'}
    </button>
  {/if}

  <!-- Debug Panel -->
  {#if showDebug}
    <div class="debug-panel">
      <h3>Showcase Debugger ({headline})</h3>
      
      <div class="tab-header">
        <span class="tab-title">Text Content</span>
      </div>
      
      <div class="control-group">
        <label>Headline</label>
        <input type="text" bind:value={debugHeadline} />
      </div>
      <div class="control-group">
        <label>Text</label>
        <textarea rows="6" bind:value={debugText}></textarea>
      </div>

      <div class="tab-header" style="margin-top: 15px;">
        <span class="tab-title">Text Layout</span>
      </div>

      <div class="control-group">
        <label>Scale: {debugTextScale}</label>
        <input type="range" min="0.5" max="2" step="0.1" bind:value={debugTextScale} />
      </div>
      <div class="control-group">
        <label>Offset X: {debugTextOffsetX}</label>
        <input type="range" min="-500" max="500" step="10" bind:value={debugTextOffsetX} />
      </div>
      <div class="control-group">
        <label>Offset Y: {debugTextOffsetY}</label>
        <input type="range" min="-500" max="500" step="10" bind:value={debugTextOffsetY} />
      </div>

      <div class="tab-header" style="margin-top: 15px;">
        <span class="tab-title">3D Model</span>
      </div>

      <div class="control-group">
        <label>Scale: {debugScale}</label>
        <input type="range" min="0.1" max="5" step="0.1" bind:value={debugScale} />
      </div>
      <div class="control-group">
        <label>Offset X: {debugModelOffsetX}</label>
        <input type="range" min="-10" max="10" step="0.1" bind:value={debugModelOffsetX} />
      </div>
      <div class="control-group">
        <label>Offset Y: {debugModelOffsetY}</label>
        <input type="range" min="-5" max="5" step="0.1" bind:value={debugModelOffsetY} />
      </div>
      <div class="control-group">
        <label>Cam Dist: {debugCamDist}</label>
        <input type="range" min="1" max="20" step="0.5" bind:value={debugCamDist} />
      </div>

      <div class="buttons">
        <button on:click={() => console.log({
          headline: debugHeadline, 
          text: debugText, 
          textScale: debugTextScale, 
          textOffsetX: debugTextOffsetX, 
          textOffsetY: debugTextOffsetY,
          scale: debugScale,
          modelOffsetX: debugModelOffsetX,
          modelOffsetY: debugModelOffsetY,
          cameraDistance: debugCamDist
        })}>Log All</button>
        <button on:click={() => showDebug = false}>Close</button>
      </div>
    </div>
  {/if}
</div>

<style>
  .showcase-container {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
  }
  
  .showcase-container.left {
    justify-content: flex-start;
  }
  
  .showcase-container.right {
    justify-content: flex-end;
  }

  .model-view {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 1;
  }

  .content-panel {
    position: relative;
    z-index: 2;
    width: 40%; /* Reduced width for side-by-side layout */
    max-width: 600px;
    
    /* Glass Card Style matching App.svelte */
    background: transparent;
    backdrop-filter: none;
    -webkit-backdrop-filter: none;
    border: none;
    box-shadow: none;
    border-radius: 24px;
    
    padding: 40px; /* Match App.svelte card-content padding */
    text-align: left; /* Align text to left inside the card */
    margin: 0 10%; /* Side margins instead of top margin */
    pointer-events: none; /* Let clicks pass through to model if needed */
  }

  .typewriter-headline {
    margin: 0 0 20px 0; /* Match App.svelte */
    font-size: 2.5rem; /* Match App.svelte */
    font-weight: 700; /* Match App.svelte */
    background: linear-gradient(to right, #fff, #aaa);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
  }

  .typewriter-text {
    font-size: 1.1rem; /* Match App.svelte */
    line-height: 1.6; /* Match App.svelte */
    color: #ccc; /* Match App.svelte */
    min-height: 3.2em; /* Reserve space for 2 lines */
    font-family: inherit; /* Remove Courier New */
  }

  @media (max-width: 768px) {
    .showcase-container {
      flex-direction: column-reverse; /* Text below model on mobile */
      justify-content: center;
    }
    
    .showcase-container.left, .showcase-container.right {
      justify-content: center;
    }
  }

  /* Debug Panel Styles */
  .debug-panel {
    position: absolute;
    top: 20px;
    right: 20px;
    background: rgba(0, 0, 0, 0.8);
    padding: 20px;
    border-radius: 10px;
    z-index: 10000;
    color: white;
    width: 250px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    pointer-events: auto;
  }
  
  .debug-panel h3 {
    margin-top: 0;
    font-size: 14px;
    border-bottom: 1px solid rgba(255, 255, 255, 0.2);
    padding-bottom: 10px;
    margin-bottom: 15px;
    color: white;
  }
  
  .control-group {
    margin-bottom: 10px;
  }
  
  .control-group label {
    display: block;
    margin-bottom: 5px;
    font-size: 11px;
    color: #ccc;
  }
  
  .control-group input {
    width: 100%;
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

  .tab-header {
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
    margin-bottom: 10px;
    padding-bottom: 5px;
  }

  .tab-title {
    font-size: 12px;
    font-weight: bold;
    color: #aaa;
    text-transform: uppercase;
  }
  
  .buttons {
    display: flex;
    gap: 10px;
  }
  
  .buttons button {
    flex: 1;
    padding: 5px;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.2);
    color: white;
    cursor: pointer;
    border-radius: 4px;
    font-size: 11px;
  }
  
  .buttons button:hover {
    background: rgba(255, 255, 255, 0.2);
  }
</style>
<script>
  import { onMount, onDestroy } from 'svelte';
  import * as THREE from 'three';

  export let scene;
  export let mainCamera;
  export let isActive = false;
  
  // Configuration Props
  export let bottom = 779;
  export let right = 1104;
  export let size = 358;
  export let handleRotation = -70;
  export let magnification = 1.5;

  let container;
  let renderer;
  let camera;
  let animationId;
  
  let windowWidth;
  let windowHeight;

  onMount(() => {
    windowWidth = window.innerWidth;
    windowHeight = window.innerHeight;
    window.addEventListener('resize', handleResize);
    
    initThree();
    animate();
  });

  onDestroy(() => {
    window.removeEventListener('resize', handleResize);
    if (animationId) cancelAnimationFrame(animationId);
    if (renderer) {
      renderer.dispose();
    }
  });

  function handleResize() {
    windowWidth = window.innerWidth;
    windowHeight = window.innerHeight;
    
    if (renderer && container) {
      renderer.setSize(size, size); // Renderer is always size x size
    }
  }

  // React to size changes
  $: if (renderer && size) {
    renderer.setSize(size, size);
  }

  function initThree() {
    if (!container) return;

    // Create a separate renderer for the loupe
    renderer = new THREE.WebGLRenderer({ 
      alpha: true, 
      antialias: true,
      preserveDrawingBuffer: true
    });
    renderer.setSize(size, size);
    renderer.setPixelRatio(window.devicePixelRatio);
    container.appendChild(renderer.domElement);

    // Create a camera for the loupe
    // We'll copy settings from the main camera in the loop
    camera = new THREE.PerspectiveCamera(45, 1, 0.1, 1000);
  }

  function animate() {
    animationId = requestAnimationFrame(animate);

    if (!isActive || !renderer || !scene || !mainCamera) return;

    // Sync loupe camera with main camera
    camera.copy(mainCamera);
    
    // Calculate Loupe Center in Window Coordinates
    // CSS positions are relative to bottom-right
    const loupeX = windowWidth - right - (size / 2);
    const loupeY = windowHeight - bottom - (size / 2);
    
    // Calculate the region of the main view we want to display
    // We want to display a region of size (size / magnification) centered at (loupeX, loupeY)
    const viewWidth = size / magnification;
    const viewHeight = size / magnification;
    
    const viewX = loupeX - (viewWidth / 2);
    const viewY = loupeY - (viewHeight / 2);
    
    // Use setViewOffset to "zoom in" to that specific part of the screen
    camera.setViewOffset(
        windowWidth, 
        windowHeight, 
        viewX, 
        viewY, 
        viewWidth, 
        viewHeight
    );
    
    renderer.render(scene, camera);
    camera.clearViewOffset();
  }
</script>

<div class="loupe-wrapper" class:active={isActive} 
     style="bottom: {bottom}px; right: {right}px; --loupe-size: {size}px; --loupe-handle-rotation: {handleRotation}deg;">
  <div class="loupe-handle"></div>
  <div class="loupe-frame" style="width: {size}px; height: {size}px;">
    <div class="loupe-canvas" bind:this={container}></div>
    <div class="loupe-overlay"></div>
    <div class="loupe-crosshair"></div>
  </div>
</div>

<style>
  /* Loupe Styles */
  .loupe-wrapper {
    position: absolute;
    z-index: 100;
    opacity: 0;
    transform: translateY(20px);
    transition: all 0.8s ease 0.5s; /* Delay appearance */
    pointer-events: none; /* Let clicks pass through */
  }

  .loupe-wrapper.active {
    opacity: 1;
    transform: translateY(0);
  }

  .loupe-frame {
    border-radius: 50%;
    border: 12px solid #333; /* Thick rim */
    background: rgba(0, 0, 0, 0.5);
    position: relative;
    box-shadow: 
      0 0 30px rgba(0, 0, 0, 0.8), 
      inset 0 0 20px rgba(0, 0, 0, 0.5),
      inset 0 0 0 2px rgba(255, 255, 255, 0.1), /* Inner highlight */
      0 0 0 1px rgba(0,0,0,1); /* Outer edge */
    overflow: hidden;
    backdrop-filter: blur(5px);
    z-index: 2; /* Above handle */
  }

  .loupe-handle {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 30px;
    height: 200px;
    background: linear-gradient(90deg, #1a1a1a, #444, #1a1a1a);
    transform-origin: top center;
    transform: rotate(var(--loupe-handle-rotation)) translateY(calc(var(--loupe-size) / 2 - 5px));
    border-radius: 15px;
    box-shadow: 5px 5px 15px rgba(0,0,0,0.8);
    z-index: 1; /* Below frame */
    border: 1px solid rgba(255,255,255,0.1);
    margin-left: -15px; /* Center horizontally */
  }

  .loupe-handle::after {
    content: '';
    position: absolute;
    top: 0;
    left: -5px;
    width: 40px;
    height: 20px;
    background: #222;
    border-radius: 5px;
    box-shadow: inset 0 0 5px rgba(0,0,0,0.8);
  }

  .loupe-canvas {
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    border-radius: 50%;
    mask-image: radial-gradient(circle, white 100%, black 100%);
    -webkit-mask-image: -webkit-radial-gradient(circle, white 100%, black 100%);
  }

  .loupe-overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    border-radius: 50%;
    background: radial-gradient(circle at 30% 30%, rgba(255, 255, 255, 0.1) 0%, rgba(255, 255, 255, 0) 20%),
                radial-gradient(circle at 50% 50%, rgba(0, 0, 0, 0) 50%, rgba(0, 0, 0, 0.4) 100%);
    box-shadow: inset 0 0 0 2px rgba(255, 255, 255, 0.1);
    pointer-events: none;
  }

  .loupe-crosshair {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: 40px;
    height: 40px;
    border: 1px solid rgba(255, 255, 255, 0.3);
    border-radius: 50%;
  }

  .loupe-crosshair::before, .loupe-crosshair::after {
    content: '';
    position: absolute;
    background: rgba(255, 255, 255, 0.5);
  }

  .loupe-crosshair::before {
    top: 50%;
    left: -10px;
    right: -10px;
    height: 1px;
  }

  .loupe-crosshair::after {
    left: 50%;
    top: -10px;
    bottom: -10px;
    width: 1px;
  }
</style>
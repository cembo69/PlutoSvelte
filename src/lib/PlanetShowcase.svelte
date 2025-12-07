<script>
  import { onMount, onDestroy } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

  export let modelPath;
  export let headline;
  export let text;
  export let scale = 1;
  export let rotationSpeed = 0.002;
  export let isActive = false;
  export let alignment = 'center'; // 'left' | 'right' | 'center'

  let container;
  let renderer, scene, camera;
  let animationId;
  let model;
  let resizeObserver;
  
  // Typewriter effect variables
  let displayedText = "";
  let isTyping = false;
  let textIndex = 0;
  let hasTyped = false;

  $: if (isActive && !hasTyped) {
    startTyping();
    hasTyped = true;
  }

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

  function startTyping() {
    displayedText = "";
    textIndex = 0;
    isTyping = true;
    typeNextChar();
  }

  function typeNextChar() {
    if (textIndex < text.length) {
      displayedText += text[textIndex];
      textIndex++;
      setTimeout(typeNextChar, 50); // Slower speed
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
    const near = 0.1;
    const far = 1000;
    camera = new THREE.PerspectiveCamera(fov, aspect, near, far);
    camera.position.set(0, 0, 5);
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
      
      // Apply alignment offset
      if (alignment === 'left') {
        model.position.x = 2; // Move model to right
      } else if (alignment === 'right') {
        model.position.x = -2; // Move model to left
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
    
    const width = container.clientWidth;
    const height = container.clientHeight;
    
    if (width === 0 || height === 0) return;

    camera.aspect = width / height;
    camera.updateProjectionMatrix();
    renderer.setSize(width, height);
  }


  function animate() {
    animationId = requestAnimationFrame(animate);

    if (model) {
      model.rotation.y += rotationSpeed;
    }

    renderer.render(scene, camera);
  }
</script>

<div class="showcase-container {alignment}">
  <div class="model-view" bind:this={container}></div>
  
  <div class="content-panel">
    <h2 class="typewriter-headline">{headline}</h2>
    <div class="typewriter-text">
      {displayedText}
    </div>
  </div>
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
  
    .typewriter-headline {
      font-size: 2rem;
    }
    .content-panel {
      width: 90%;
      margin: 0;
      margin-top: 40vh; /* Restore top margin for mobile */
      text-align: center;
    }
  }
</style>
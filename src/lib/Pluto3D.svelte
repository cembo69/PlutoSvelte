<script>
  import { onMount, onDestroy } from 'svelte';
  import { Renderer, Camera, Transform, GLTFLoader, Program, Mesh } from 'ogl';

  let canvasEl;
  let renderer, gl, camera, scene;
  let plutoModel;
  let animateId;

  function initWebGL() {
    renderer = new Renderer({ canvas: canvasEl, alpha: true, antialias: true });
    gl = renderer.gl;
    gl.clearColor(0, 0, 0, 0);

    camera = new Camera(gl, { fov: 45 });
    camera.position.set(0, 0, 5);
    camera.lookAt([0, 0, 0]);

    scene = new Transform();

    loadPluto();
    resize();
    window.addEventListener('resize', resize);
    startAnimation();
  }

  async function loadPluto() {
    try {
      console.log('Loading Pluto model...');
      const gltf = await GLTFLoader.load(gl, '/Pluto.glb', {
        draco: 'https://www.gstatic.com/draco/versioned/decoders/1.5.6/'
      });

      let pluto = gltf.scene || (gltf.scenes && gltf.scenes[0]);
      
      // Handle if it's an array
      if (Array.isArray(pluto)) {
        console.log('Scene is an array, taking first element');
        for (let i = 0; i < pluto.length; i++) {
          const scene = pluto[i];
          let meshCount = 0;
          function countMeshes(node) {
            if (node.geometry) meshCount++;
            if (node.children) node.children.forEach(countMeshes);
          }
          countMeshes(scene);
          if (meshCount > 0) {
            pluto = scene;
            break;
          }
        }
      }

      if (!pluto || Array.isArray(pluto)) {
        console.error('No valid scene in GLTF');
        return;
      }

      plutoModel = pluto;
      plutoModel.scale.set(1.5, 1.5, 1.5);
      plutoModel.setParent(scene);
      
      console.log('Pluto loaded!');
    } catch (err) {
      console.error('Failed to load Pluto:', err);
    }
  }

  function resize() {
    if (!renderer || !canvasEl) return;
    const parent = canvasEl.parentElement;
    renderer.setSize(parent.offsetWidth, parent.offsetHeight);
    camera.perspective({ aspect: gl.canvas.width / gl.canvas.height });
  }

  function startAnimation() {
    const update = (t) => {
      if (plutoModel) {
        // Langsame Rotation
        plutoModel.rotation.y = t * 0.0002;
      }

      if (renderer && scene && camera) {
        renderer.render({ scene, camera });
      }

      animateId = requestAnimationFrame(update);
    };
    animateId = requestAnimationFrame(update);
  }

  onMount(() => {
    initWebGL();
  });

  onDestroy(() => {
    if (animateId) cancelAnimationFrame(animateId);
    window.removeEventListener('resize', resize);
  });
</script>

<div class="pluto-container">
  <canvas bind:this={canvasEl} class="pluto-canvas"></canvas>
</div>

<style>
  .pluto-container {
    width: 100%;
    height: 100%;
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .pluto-canvas {
    width: 100%;
    height: 100%;
  }
</style>

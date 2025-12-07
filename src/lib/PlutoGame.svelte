<script>
  import { onMount, onDestroy } from 'svelte';
  import { Renderer, Camera, Transform, GLTFLoader, Vec3, Texture } from 'ogl';
  import Galaxy from './Galaxy.svelte';
  import SplitText from './SplitText.svelte';

  export let onExit = () => {};

  let canvas;
  let canvas3D;
  let ctx;
  let renderer, gl, camera, scene, ship3D;
  let animationId;
  let score = 0;
  let lives = 3;
  let gameState = 'start'; // start, playing, feedback, game_over, victory
  
  // Game Constants
  const PLAYER_SPEED = 8;
  const BULLET_SPEED = 10;
  const SHOOT_COOLDOWN = 20;
  const QUESTION_TIME = 15; // Zeit pro Frage in Sekunden

  // Game Objects
  let player = {
    x: 0,
    y: 0,
    width: 50,
    height: 40,
    color: '#00ff00',
    cooldown: 0
  };

  let bullets = [];
  let particles = [];
  let answerBlocks = [];
  let timeLeft = 0;
  
  // Input handling
  let keys = {
    ArrowLeft: false,
    ArrowRight: false,
    Space: false
  };

  // Quiz Data
  const questions = [
    { q: "Ist Pluto ein Planet?", a: ["Nein (Zwergplanet)", "Ja, natürlich", "Vielleicht"], correct: 0 },
    { q: "Wann wurde Pluto entdeckt?", a: ["1990", "1930", "2006"], correct: 1 },
    { q: "Wer entdeckte Pluto?", a: ["Galileo Galilei", "Isaac Newton", "Clyde Tombaugh"], correct: 2 },
    { q: "Wie viele Monde hat Pluto?", a: ["Keine", "Einen", "Fünf"], correct: 2 },
    { q: "Wie heißt Plutos größter Mond?", a: ["Charon", "Titan", "Europa"], correct: 0 },
    { q: "Warum ist Pluto kein Planet?", a: ["Zu weit weg", "Bahn nicht bereinigt", "Zu kalt"], correct: 1 },
    { q: "Woraus besteht Pluto hauptsächlich?", a: ["Gas", "Gestein & Eis", "Flüssiges Wasser"], correct: 1 },
    { q: "Wie lange dauert ein Pluto-Jahr?", a: ["248 Erdjahre", "100 Erdjahre", "10 Erdjahre"], correct: 0 },
    { q: "Welche Sonde besuchte Pluto?", a: ["Voyager 1", "New Horizons", "Cassini"], correct: 1 },
    { q: "In welchem Gürtel liegt Pluto?", a: ["Asteroidengürtel", "Oriongürtel", "Kuipergürtel"], correct: 2 }
  ];
  
  let currentQuestionIndex = 0;
  let feedbackText = "";
  let feedbackColor = "";
  let feedbackTimer = 0;

  // Debug Configuration
  let debugConfig = {
    rotX: 4.620,
    rotY: -3.060,
    rotZ: -0.050,
    posY: -6.000,
    posZ: 0.000,
    scale: 1.0
  };
  let currentTilt = 0;

  // Destruction Images
  let destructionImages = [];
  const destructionSources = [
    '/destroyprogress/Minecraft Block Abbauen Stage 1.png',
    '/destroyprogress/Minecraft Block Abbauen Stage 6.png',
    '/destroyprogress/Minecraft Block Abbauen Stage 9.png'
  ];

  function loadDestructionImages() {
    destructionSources.forEach(src => {
      const img = new Image();
      img.src = src;
      destructionImages.push(img);
    });
  }

  async function init3D() {
    if (!canvas3D) return;
    renderer = new Renderer({ canvas: canvas3D, alpha: true, antialias: true });
    gl = renderer.gl;
    gl.clearColor(0, 0, 0, 0);

    camera = new Camera(gl, { fov: 45 });
    camera.position.set(0, 0, 20);

    scene = new Transform();

    try {
      const gltf = await GLTFLoader.load(gl, '/ConceptJet.glb', {
        draco: 'https://www.gstatic.com/draco/versioned/decoders/1.5.6/'
      });
      
      let realShip = gltf.scene || (gltf.scenes && gltf.scenes[0]);
      
      if (Array.isArray(realShip)) {
        for (let i = 0; i < realShip.length; i++) {
          const s = realShip[i];
          let meshCount = 0;
          function countMeshes(node) {
            if (node.geometry) meshCount++;
            if (node.children) node.children.forEach(countMeshes);
          }
          countMeshes(s);
          if (meshCount > 0) {
            realShip = s;
            break;
          }
        }
      }

      if (realShip && !Array.isArray(realShip)) {
        ship3D = realShip;
        ship3D.scale.set(debugConfig.scale, debugConfig.scale, debugConfig.scale);
        
        // Set color to white
        function setWhiteColor(node) {
          if (node.program && node.program.uniforms) {
            // Force base color to white
            if (node.program.uniforms.uBaseColorFactor) {
               node.program.uniforms.uBaseColorFactor.value = new Float32Array([1, 1, 1, 1]);
            }
            // Add some emissive to make it brighter
            if (node.program.uniforms.uEmissiveFactor) {
               node.program.uniforms.uEmissiveFactor.value = new Float32Array([0.5, 0.5, 0.5, 1]);
            }
            
            // Replace textures with white texture
            const whiteTex = new Texture(gl, {
              image: new Uint8Array([255, 255, 255, 255]),
              width: 1,
              height: 1,
            });
            
            ['tBaseColor', 'tMap', 'uBaseColorTexture'].forEach(name => {
                if (node.program.uniforms[name]) {
                    node.program.uniforms[name].value = whiteTex;
                }
            });
          }
          if (node.children) node.children.forEach(setWhiteColor);
        }
        setWhiteColor(ship3D);

        ship3D.rotation.x = debugConfig.rotX;
        ship3D.rotation.y = debugConfig.rotY;
        ship3D.rotation.z = debugConfig.rotZ;
        ship3D.position.y = debugConfig.posY;
        ship3D.position.z = debugConfig.posZ;
        
        ship3D.setParent(scene);
      }
    } catch (e) {
      console.error('Failed to load ship', e);
    }
    
    resize3D();
  }

  function resize3D() {
    if (!renderer) return;
    renderer.setSize(window.innerWidth, window.innerHeight);
    camera.perspective({ aspect: gl.canvas.width / gl.canvas.height });
  }

  function initGame() {
    if (!canvas) return;
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
    ctx = canvas.getContext('2d');

    loadDestructionImages();

    // Reset Player
    player.x = canvas.width / 2 - player.width / 2;
    player.y = canvas.height - 80;

    loadQuestion();
  }

  function loadQuestion() {
    bullets = [];
    particles = [];
    answerBlocks = [];
    timeLeft = QUESTION_TIME;
    
    if (currentQuestionIndex >= questions.length) {
      gameState = 'victory';
      return;
    }

    const q = questions[currentQuestionIndex];
    // Smaller blocks to simulate depth/distance
    const blockWidth = 180; 
    const blockHeight = 60;
    const gap = 30;
    const totalWidth = (3 * blockWidth) + (2 * gap);
    const startX = (canvas.width - totalWidth) / 2;

    q.a.forEach((answer, i) => {
      answerBlocks.push({
        x: startX + i * (blockWidth + gap),
        y: 150, // Slightly lower to look "further back" in perspective if horizon is center, or just keep it high. Let's move it down a bit to 150 to be more central-ish but still top.
        width: blockWidth,
        height: blockHeight,
        text: answer,
        isCorrect: i === q.correct,
        color: '#00ffff',
        active: true,
        health: 3,
        maxHealth: 3
      });
    });
  }

  function createHitFeedback(x, y) {
    // Just a few small sparks
    for (let i = 0; i < 5; i++) {
      const angle = Math.random() * Math.PI * 2;
      const speed = Math.random() * 5 + 2;
      particles.push({
        x, y,
        vx: Math.cos(angle) * speed,
        vy: Math.sin(angle) * speed,
        life: 0.3, // Short life
        color: '#ffffff',
        type: 'spark',
        size: 1.5
      });
    }
  }

  function createExplosion(x, y, color) {
    // Shockwave
    particles.push({
      x, y,
      vx: 0,
      vy: 0,
      life: 1.0,
      color: color,
      type: 'shockwave',
      size: 1
    });

    // Debris
    for (let i = 0; i < 40; i++) {
      const angle = Math.random() * Math.PI * 2;
      const speed = Math.random() * 8 + 2;
      particles.push({
        x, y,
        vx: Math.cos(angle) * speed,
        vy: Math.sin(angle) * speed,
        life: 1.0,
        color: color,
        type: 'debris',
        size: Math.random() * 6 + 2
      });
    }
    
    // Sparks
    for (let i = 0; i < 20; i++) {
      const angle = Math.random() * Math.PI * 2;
      const speed = Math.random() * 15 + 5;
      particles.push({
        x, y,
        vx: Math.cos(angle) * speed,
        vy: Math.sin(angle) * speed,
        life: 0.6,
        color: color,
        type: 'spark',
        size: 2
      });
    }
  }

  function update() {
    // Always update particles regardless of game state
    particles.forEach((p, i) => {
      if (p.type === 'shockwave') {
        p.size += 5;
        p.life -= 0.05;
      } else if (p.type === 'spark') {
        p.x += p.vx;
        p.y += p.vy;
        p.life -= 0.05;
      } else {
        p.x += p.vx;
        p.y += p.vy;
        p.life -= 0.02;
        p.vx *= 0.95; // Friction
        p.vy *= 0.95;
      }
      
      if (p.life <= 0) particles.splice(i, 1);
    });

    if (gameState !== 'playing') return;

    // Player Movement
    if (keys.ArrowLeft && player.x > 0) player.x -= PLAYER_SPEED;
    if (keys.ArrowRight && player.x < canvas.width - player.width) player.x += PLAYER_SPEED;

    // Player Shooting
    if (player.cooldown > 0) player.cooldown--;
    if (keys.Space && player.cooldown <= 0) {
      bullets.push({
        x: player.x + player.width / 2 - 3,
        y: player.y,
        width: 6,
        height: 15,
        speed: BULLET_SPEED
      });
      player.cooldown = SHOOT_COOLDOWN;
    }

    // Update Bullets
    bullets.forEach((b, i) => {
      b.y -= b.speed;
      if (b.y < 0) bullets.splice(i, 1);
    });

    // Update Timer
    timeLeft -= 0.016; // approx 1/60 seconds
    if (timeLeft <= 0) {
      lives--;
      if (lives <= 0) {
        gameState = 'game_over';
      } else {
        feedbackText = "ZEIT ABGELAUFEN!";
        feedbackColor = "#ff0000";
        gameState = 'feedback';
        setTimeout(() => {
          // Reset for retry
          loadQuestion();
          gameState = 'playing';
        }, 1500);
      }
    }

    // Update Blocks (No movement anymore)
    /* 
    answerBlocks.forEach(block => {
       // Static blocks
    });
    */

    // Bullet Collisions with Blocks
    bullets.forEach((b, bIndex) => {
      answerBlocks.forEach((block, blockIndex) => {
        if (!block.active) return;

        if (
          b.x < block.x + block.width &&
          b.x + b.width > block.x &&
          b.y < block.y + block.height &&
          b.y + b.height > block.y
        ) {
          // Hit!
          bullets.splice(bIndex, 1);
          block.health--;
          
          // Small particle effect on hit
          createHitFeedback(b.x, b.y);

          if (block.health <= 0) {
            if (block.isCorrect) {
              // Correct Answer
              createExplosion(block.x + block.width/2, block.y + block.height/2, '#00ff00');
              score += 1000;
              feedbackText = "RICHTIG!";
              feedbackColor = "#00ff00";
              gameState = 'feedback';
              setTimeout(() => {
                currentQuestionIndex++;
                gameState = 'playing';
                loadQuestion();
              }, 1500);
            } else {
              // Wrong Answer
              createExplosion(block.x + block.width/2, block.y + block.height/2, '#ff0000');
              block.active = false; // Destroy wrong block
              lives--;
              score -= 200;
              feedbackText = "FALSCH!";
              feedbackColor = "#ff0000";
              
              // Show feedback briefly but continue if lives > 0
              if (lives <= 0) {
                gameState = 'game_over';
              }
            }
          }
        }
      });
    });

    // Particles
    /* Moved to top of update function */
  }

  function draw() {
    if (!ctx) return;
    
    // Clear
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Draw Question Text
    if (gameState === 'playing' || gameState === 'feedback') {
      // Question text is now handled by SplitText component in HTML overlay
      
      // Draw Timer Bar (Vertical on the right)
      const barWidth = 20;
      const barHeight = 300;
      const barX = canvas.width - 50;
      const barY = (canvas.height - barHeight) / 2;
      const pct = Math.max(0, timeLeft / QUESTION_TIME);
      
      // Label
      ctx.save();
      ctx.translate(barX + barWidth + 20, barY + barHeight / 2);
      ctx.rotate(-Math.PI / 2);
      ctx.fillStyle = '#fff';
      ctx.font = '900 16px "SF Pro Display", -apple-system, BlinkMacSystemFont, sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText("ZEIT", 0, 0);
      ctx.restore();

      // Background
      ctx.fillStyle = 'rgba(50, 50, 50, 0.5)';
      ctx.fillRect(barX, barY, barWidth, barHeight);
      
      // Progress (filling from bottom)
      const fillHeight = barHeight * pct;
      ctx.fillStyle = pct > 0.5 ? '#00ff00' : (pct > 0.2 ? '#ffff00' : '#ff0000');
      
      // Glow effect for timer
      ctx.shadowBlur = 15;
      ctx.shadowColor = ctx.fillStyle;
      ctx.fillRect(barX, barY + (barHeight - fillHeight), barWidth, fillHeight);
      ctx.shadowBlur = 0;
      
      // Border
      ctx.strokeStyle = '#fff';
      ctx.lineWidth = 2;
      ctx.strokeRect(barX, barY, barWidth, barHeight);
    }

    // Update & Render 3D Ship
    if (ship3D && renderer) {
      // Only render if playing or feedback
      if (gameState === 'playing' || gameState === 'feedback') {
        ship3D.visible = true;
        const dist = camera.position.z;
        const vFOV = camera.fov * Math.PI / 180;
        const visibleHeight = 2 * Math.tan(vFOV / 2) * dist;
        const visibleWidth = visibleHeight * camera.aspect;
        
        const playerCenterX = player.x + player.width / 2;
        const normalizedX = (playerCenterX / canvas.width) * 2 - 1;
        
        ship3D.position.x = normalizedX * (visibleWidth / 2);
        ship3D.position.y = debugConfig.posY;
        ship3D.position.z = debugConfig.posZ;
        ship3D.scale.set(debugConfig.scale, debugConfig.scale, debugConfig.scale);

        // Apply base rotation from debug
        // Perspective correction: Rotate ship to counteract perspective convergence
        // We subtract the angle to rotate it "outwards" so it looks parallel to the screen edges
        const perspectiveAngle = Math.atan(ship3D.position.x / camera.position.z);
        ship3D.rotation.x = debugConfig.rotX;
        ship3D.rotation.y = debugConfig.rotY - perspectiveAngle;

        // Tilt based on movement (added to base Z rotation)
        const targetTilt = keys.ArrowLeft ? 0.5 : (keys.ArrowRight ? -0.5 : 0);
        currentTilt += (targetTilt - currentTilt) * 0.1;
        ship3D.rotation.z = debugConfig.rotZ + currentTilt;

        renderer.render({ scene, camera });
      } else {
        ship3D.visible = false;
        renderer.render({ scene, camera }); // Render empty scene to clear canvas
      }
    } else if (gameState === 'playing' || gameState === 'feedback') {
      // Fallback 2D Player if 3D not loaded yet
      ctx.fillStyle = player.color;
      ctx.beginPath();
      ctx.moveTo(player.x + player.width/2, player.y);
      ctx.lineTo(player.x + player.width, player.y + player.height);
      ctx.lineTo(player.x + player.width/2, player.y + player.height - 10);
      ctx.lineTo(player.x, player.y + player.height);
      ctx.closePath();
      ctx.fill();
    }

    // Draw Answer Blocks
    if (gameState === 'playing' || gameState === 'feedback') {
      answerBlocks.forEach(block => {
        if (!block.active) return;
        
        // Block Body
        ctx.fillStyle = 'rgba(0, 255, 255, 0.1)';
        ctx.strokeStyle = '#00ffff';
        ctx.lineWidth = 2;
        ctx.fillRect(block.x, block.y, block.width, block.height);
        ctx.strokeRect(block.x, block.y, block.width, block.height);

        // Destruction Overlay
        const damage = block.maxHealth - block.health;
        if (damage > 0 && damage <= destructionImages.length) {
          const imgIndex = damage - 1;
          if (destructionImages[imgIndex] && destructionImages[imgIndex].complete) {
             ctx.drawImage(destructionImages[imgIndex], block.x, block.y, block.width, block.height);
          }
        }

        // Text
        ctx.fillStyle = '#ffffff';
        ctx.font = '900 14px "SF Pro Display", -apple-system, BlinkMacSystemFont, sans-serif';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(block.text, block.x + block.width/2, block.y + block.height/2);
      });
    }

    // Draw Bullets
    bullets.forEach(b => {
      ctx.save();
      ctx.shadowBlur = 10;
      ctx.shadowColor = '#ffff00';
      ctx.fillStyle = '#ffff00';
      
      // Glowing core
      ctx.beginPath();
      ctx.arc(b.x + b.width/2, b.y + b.height/2, b.width, 0, Math.PI * 2);
      ctx.fill();
      
      // Trail
      const gradient = ctx.createLinearGradient(b.x, b.y, b.x, b.y + b.height * 2);
      gradient.addColorStop(0, 'rgba(255, 255, 0, 0.8)');
      gradient.addColorStop(1, 'rgba(255, 100, 0, 0)');
      ctx.fillStyle = gradient;
      ctx.fillRect(b.x, b.y + b.height/2, b.width, b.height * 2);
      
      ctx.restore();
    });

    // Draw Particles
    particles.forEach(p => {
      ctx.save();
      ctx.globalAlpha = p.life;
      
      if (p.type === 'shockwave') {
        ctx.strokeStyle = p.color;
        ctx.lineWidth = 3;
        ctx.beginPath();
        ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
        ctx.stroke();
      } else {
        ctx.fillStyle = p.color;
        ctx.shadowBlur = 5;
        ctx.shadowColor = p.color;
        ctx.beginPath();
        if (p.type === 'spark') {
           ctx.rect(p.x, p.y, p.size, p.size * 3); // Elongated sparks
        } else {
           ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
        }
        ctx.fill();
      }
      
      ctx.restore();
    });

    // Draw Feedback Overlay
    if (gameState === 'feedback') {
      ctx.fillStyle = feedbackColor;
      ctx.font = '900 60px "SF Pro Display", -apple-system, BlinkMacSystemFont, sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText(feedbackText, canvas.width / 2, canvas.height / 2);
    }
  }

  function loop() {
    update();
    draw();
    animationId = requestAnimationFrame(loop);
  }

  function handleKey(e, isDown) {
    if (e.code === 'ArrowLeft' || e.key === 'a') keys.ArrowLeft = isDown;
    if (e.code === 'ArrowRight' || e.key === 'd') keys.ArrowRight = isDown;
    if (e.code === 'Space' || e.key === ' ') keys.Space = isDown;
  }

  function startGame() {
    score = 0;
    lives = 3;
    currentQuestionIndex = 0;
    gameState = 'playing';
    initGame();
  }

  onMount(() => {
    window.addEventListener('keydown', e => handleKey(e, true));
    window.addEventListener('keyup', e => handleKey(e, false));
    window.addEventListener('resize', () => {
      initGame();
      resize3D();
    });
    
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
    ctx = canvas.getContext('2d');
    
    init3D();
    loop();
  });

  onDestroy(() => {
    cancelAnimationFrame(animationId);
    window.removeEventListener('keydown', e => handleKey(e, true));
    window.removeEventListener('keyup', e => handleKey(e, false));
    window.removeEventListener('resize', initGame);
  });
</script>

<div class="game-container">
  <div class="galaxy-bg">
    <Galaxy density={0.3} starSpeed={0.2} />
  </div>
  <canvas bind:this={canvas3D} class="canvas-3d"></canvas>
  <canvas bind:this={canvas} class="canvas-2d"></canvas>

  <!-- HUD -->
  {#if gameState === 'playing' || gameState === 'feedback'}
    <div class="hud">
      <div class="score">SCORE: {score}</div>
      <div class="progress">FRAGE: {currentQuestionIndex + 1} / {questions.length}</div>
    </div>
    
    <div class="question-container">
      {#key currentQuestionIndex}
        <SplitText 
          text={questions[currentQuestionIndex]?.q || ""} 
          className="question-text"
          delay={100}
          duration={0.6}
        />
      {/key}
    </div>
    
    <div class="lives-container">
      <div class="lives-label">LIVES</div>
      {#each Array(lives) as _}
        <div class="heart">❤️</div>
      {/each}
    </div>
  {/if}

  <!-- Start Screen -->
  {#if gameState === 'start'}
    <div class="overlay start-screen">
      <h1>PLUTO QUIZ SHOOTER</h1>
      <p>Schieße auf die richtige Antwort!</p>
      <button class="btn" on:click={startGame}>MISSION STARTEN</button>
      <button class="btn secondary" on:click={onExit}>ZURÜCK</button>
    </div>
  {/if}

  <!-- Game Over -->
  {#if gameState === 'game_over'}
    <div class="overlay game-over">
      <h1>GAME OVER</h1>
      <p>Final Score: {score}</p>
      <button class="btn" on:click={startGame}>NEUSTART</button>
      <button class="btn secondary" on:click={onExit}>Back to Start</button>
    </div>
  {/if}

  <!-- Victory -->
  {#if gameState === 'victory'}
    <div class="overlay victory">
      <h1>MISSION ERFÜLLT!</h1>
      <p>Du bist ein Pluto-Experte!</p>
      <p>Score: {score}</p>
      <button class="btn" on:click={onExit}>Back to Start</button>
    </div>
  {/if}
</div>

<style>
  .game-container {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background: transparent; /* Changed from #000 to transparent for Galaxy */
    z-index: 9999;
    font-family: 'SF Pro Display', -apple-system, BlinkMacSystemFont, sans-serif;
    overflow: hidden;
  }

  .galaxy-bg {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 0;
    background: #000; /* Fallback */
  }

  canvas {
    display: block;
  }

  .canvas-3d {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 1;
  }

  .canvas-2d {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 2;
  }

  .hud {
    position: absolute;
    top: 20px;
    left: 20px;
    right: 20px;
    display: flex;
    justify-content: space-between;
    color: #00ffff;
    font-size: 24px;
    font-weight: bold;
    text-shadow: 0 0 10px #00ffff;
    pointer-events: none;
  }

  .lives-container {
    position: absolute;
    left: 20px;
    top: 50%;
    transform: translateY(-50%);
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    pointer-events: none;
    z-index: 10;
  }

  .question-container {
    position: absolute;
    top: 60px;
    left: 0;
    width: 100%;
    display: flex;
    justify-content: center;
    pointer-events: none;
    z-index: 20;
  }

  :global(.question-text) {
    font-size: 32px;
    font-weight: 900;
    color: #ffffff;
    text-align: center;
    text-shadow: 0 0 10px rgba(0, 255, 255, 0.5);
  }

  .lives-label {
    color: #fff;
    font-size: 16px;
    font-weight: 900;
    writing-mode: vertical-rl;
    text-orientation: mixed;
    transform: rotate(180deg);
    margin-bottom: 10px;
    letter-spacing: 2px;
  }

  .heart {
    font-size: 30px;
    filter: drop-shadow(0 0 5px rgba(255, 0, 0, 0.5));
  }

  .overlay {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    text-align: center;
    background: rgba(0, 20, 40, 0.95);
    padding: 50px;
    border: 2px solid #00ffff;
    border-radius: 20px;
    box-shadow: 0 0 30px rgba(0, 255, 255, 0.3);
    color: #fff;
    min-width: 500px;
    z-index: 100; /* Ensure overlay is above canvases */
  }

  h1 {
    font-family: 'SF Pro Display', -apple-system, BlinkMacSystemFont, sans-serif;
    font-weight: 900;
    font-size: 48px;
    margin-bottom: 20px;
    background: linear-gradient(to right, #00ffff, #ff00ff);
    background-clip: text;
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
  }

  .btn {
    background: transparent;
    border: 2px solid #fff;
    color: #fff;
    padding: 15px 40px;
    font-size: 20px;
    font-weight: bold;
    cursor: pointer;
    margin: 15px;
    transition: all 0.2s;
    text-transform: uppercase;
    border-radius: 50px;
  }

  .btn:hover {
    background: #fff;
    color: #000;
    box-shadow: 0 0 20px rgba(255, 255, 255, 0.5);
    transform: scale(1.05);
  }

  .btn.secondary {
    border-color: #fff;
    color: #fff;
    opacity: 0.7;
  }

  .btn.secondary:hover {
    background: #fff;
    color: #000;
    box-shadow: 0 0 20px rgba(255, 255, 255, 0.5);
    opacity: 1;
  }
</style>

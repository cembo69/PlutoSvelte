<script>
  import { onMount, createEventDispatcher } from 'svelte';
  
  const dispatch = createEventDispatcher();
  
  // State Management
  let view3D = false;
  let zoomLarge = true;
  let scaleMode = 'speed'; // 'speed', 'size', 'distance'
  let dataOpen = true;
  let activePlanet = 'pluto';
  let hideUI = true;
  let opening = true;
  let baseTilt = 63; // Default tilt in degrees

  // Saturn Ring Debugging
  let showSaturnDebug = false;
  let ringSize = 2; // em
  let ringBorderWidth = 0.3; // em
  let ringColor = "rgba(160, 147, 130, 0.7)";
  let ringShadowColor = "rgba(160, 147, 130, 0.3)";
  let ringShadowBlur = 10; // px
  let ringTiltX = 78; // deg
  let ringTiltY = 0; // deg

  export let scrollProgress = 0;

  // Initial Animation
  onMount(() => {
    // Load animation
    setTimeout(() => {
      opening = false;
      view3D = true;
      scaleMode = 'speed';
    }, 500);
    
    setTimeout(() => {
      hideUI = false;
    }, 2000);
  });

  const planets = [
    { id: 'sun', name: 'Sun', speed: '0 km/h', size: '4,370,005 km', distance: '149,598,262 km' },
    { id: 'mercury', name: 'Mercury', speed: '170,503 km/h', size: '15,329 km', distance: '57,909,227 km' },
    { id: 'venus', name: 'Venus', speed: '126,074 km/h', size: '38,024 km', distance: '108,209,475 km' },
    { id: 'earth', name: 'Earth', speed: '107,218 km/h', size: '40,030 km', distance: '149,598,262 km' },
    { id: 'mars', name: 'Mars', speed: '86,677 km/h', size: '21,296 km', distance: '227,943,824 km' },
    { id: 'jupiter', name: 'Jupiter', speed: '47,002 km/h', size: '439,263 km', distance: '778,340,821 km' },
    { id: 'saturn', name: 'Saturn', speed: '34,701 km/h', size: '365,882 km', distance: '1,426,666,422 km' },
    { id: 'uranus', name: 'Uranus', speed: '24,477 km/h', size: '159,354 km', distance: '2,870,658,186 km' },
    { id: 'neptune', name: 'Neptune', speed: '19,566 km/h', size: '154,704 km', distance: '4,498,396,441 km' },
    { id: 'pluto', name: 'Pluto', speed: '16,800 km/h', size: '7,232 km', distance: '5,906,440,628 km' }
  ];

  function toggleView() {
    view3D = !view3D;
  }

  function toggleZoom() {
    zoomLarge = !zoomLarge;
  }

  function selectPlanet(planetId) {
    activePlanet = planetId;
  }

  function setScaleMode(mode) {
    scaleMode = mode;
  }

  function toggleData() {
    dataOpen = !dataOpen;
  }

  $: bodyClass = `${opening ? 'opening' : ''} ${hideUI ? 'hide-UI' : ''} ${view3D ? 'view-3D' : 'view-2D'} ${zoomLarge ? 'zoom-large' : 'zoom-close'} ${dataOpen ? 'data-open' : 'data-close'}`;
  $: universeClass = `scale-${scaleMode === 'speed' ? 'stretched' : scaleMode === 'size' ? 's' : 'd'} set-${scaleMode}`;
  $: solarsysClass = activePlanet;


  $: scaleClass = scaleMode === 'speed' ? 'scale-stretched' : scaleMode === 'size' ? 'scale-s' : 'scale-d';
  $: dataLabel = scaleMode === 'speed' ? 'Orbit Velocity' : scaleMode === 'size' ? 'Equatorial Circumference' : 'From Sun';
  $: dataValue = planets.find(p => p.id === activePlanet)?.[scaleMode] || '';

  // Scroll Animation Logic
  // Wir starten bei baseTilt (Draufsicht) und rotieren zu 0deg (Seitenansicht) -> Wir fliegen in die Ebene
  // Gleichzeitig zoomen wir extrem rein (translateZ) für einen Warp-Effekt
  // WICHTIG: Kein translateY, damit die Sonne immer exakt in der Mitte bleibt!
  $: scrollTransform = (view3D && scrollProgress > 0) ? `
    transform: 
      rotateX(${baseTilt - scrollProgress * 63}deg) 
      translateZ(${scrollProgress * 5000}px);
    transition: none;
  ` : (view3D ? `transform: rotateX(${baseTilt}deg);` : '');
</script>

<div class="solar-system-container {bodyClass}">
  <!-- Navbar -->
  <div id="navbar" style="opacity: {1 - scrollProgress * 2}">
    <h1>3D Solar System<br><span>Interactive Model</span></h1>
    <div class="view-controls">
      <span class="view-label {view3D ? '' : 'active'}">2D</span>
      <label class="switch">
        <input type="checkbox" checked={view3D} on:change={toggleView}>
        <span class="slider"></span>
      </label>
      <span class="view-label {view3D ? 'active' : ''}">3D</span>
    </div>
  </div>

  <!-- Data Panel -->
  <div id="data" style="opacity: {1 - scrollProgress * 2}">
    {#each planets as planet}
      <a 
        class="{planet.id} {activePlanet === planet.id ? 'active' : ''}" 
        title={planet.name} 
        href="#{planet.id}" 
        on:click|preventDefault={() => selectPlanet(planet.id)}
      >
        {planet.name}
      </a>
    {/each}
  </div>

  <!-- Debug Panel for Saturn Ring -->
  {#if showSaturnDebug}
    <div class="debug-panel">
      <h3>Saturn Ring Debugger</h3>
      
      <div class="control-group">
        <label>Size (em): {ringSize}</label>
        <input type="range" min="1" max="5" step="0.1" bind:value={ringSize} />
      </div>
      
      <div class="control-group">
        <label>Border Width (em): {ringBorderWidth}</label>
        <input type="range" min="0.1" max="1" step="0.05" bind:value={ringBorderWidth} />
      </div>
      
      <div class="control-group">
        <label>Shadow Blur (px): {ringShadowBlur}</label>
        <input type="range" min="0" max="50" step="1" bind:value={ringShadowBlur} />
      </div>
      
      <div class="control-group">
        <label>Color</label>
        <input type="text" bind:value={ringColor} />
      </div>
      
      <div class="control-group">
        <label>Shadow Color</label>
        <input type="text" bind:value={ringShadowColor} />
      </div>

      <div class="control-group">
        <label>Tilt X (deg): {ringTiltX}</label>
        <input type="range" min="0" max="360" step="1" bind:value={ringTiltX} />
      </div>

      <div class="control-group">
        <label>Tilt Y (deg): {ringTiltY}</label>
        <input type="range" min="0" max="360" step="1" bind:value={ringTiltY} />
      </div>

      <div class="buttons">
        <button on:click={() => console.log({
          ringSize,
          ringBorderWidth,
          ringShadowBlur,
          ringColor,
          ringShadowColor,
          ringTiltX,
          ringTiltY
        })}>Log</button>
        <button on:click={() => showSaturnDebug = false}>Close</button>
      </div>
    </div>
  {/if}

  <!-- Universe -->
  <div id="universe" class={universeClass}>
    <div id="galaxy">
      <div id="solar-system" class={solarsysClass} style={scrollTransform}>
        
        <!-- Mercury -->
        <div id="mercury" class="orbit">
          <svg class="orbit-ring" viewBox="0 0 100 100">
            <circle class="visual-ring" cx="50" cy="50" r="49.5" />
            <circle class="hitbox-ring" cx="50" cy="50" r="49.5" />
          </svg>
          <div class="pos">
            <div class="planet">
              <dl class="infos">
                <dt>Mercury</dt>
                <dd><span>{dataLabel}</span></dd>
                <dd class="value">{planets[1][scaleMode]}</dd>
              </dl>
            </div>
          </div>
        </div>

        <!-- Venus -->
        <div id="venus" class="orbit">
          <svg class="orbit-ring" viewBox="0 0 100 100">
            <circle class="visual-ring" cx="50" cy="50" r="49.5" />
            <circle class="hitbox-ring" cx="50" cy="50" r="49.5" />
          </svg>
          <div class="pos">
            <div class="planet">
              <dl class="infos">
                <dt>Venus</dt>
                <dd><span>{dataLabel}</span></dd>
                <dd class="value">{planets[2][scaleMode]}</dd>
              </dl>
            </div>
          </div>
        </div>

        <!-- Earth -->
        <div id="earth" class="orbit">
          <svg class="orbit-ring" viewBox="0 0 100 100">
            <circle class="visual-ring" cx="50" cy="50" r="49.5" />
            <circle class="hitbox-ring" cx="50" cy="50" r="49.5" />
          </svg>
          <div class="pos">
            <div class="orbit">
              <div class="pos">
                <div class="moon"></div>
              </div>
            </div>
            <div class="planet">
              <dl class="infos">
                <dt>Earth</dt>
                <dd><span>{dataLabel}</span></dd>
                <dd class="value">{planets[3][scaleMode]}</dd>
              </dl>
            </div>
          </div>
        </div>

        <!-- Mars -->
        <div id="mars" class="orbit">
          <svg class="orbit-ring" viewBox="0 0 100 100">
            <circle class="visual-ring" cx="50" cy="50" r="49.5" />
            <circle class="hitbox-ring" cx="50" cy="50" r="49.5" />
          </svg>
          <div class="pos">
            <div class="planet">
              <dl class="infos">
                <dt>Mars</dt>
                <dd><span>{dataLabel}</span></dd>
                <dd class="value">{planets[4][scaleMode]}</dd>
              </dl>
            </div>
          </div>
        </div>

        <!-- Jupiter -->
        <div id="jupiter" class="orbit">
          <svg class="orbit-ring" viewBox="0 0 100 100">
            <circle class="visual-ring" cx="50" cy="50" r="49.5" />
            <circle class="hitbox-ring" cx="50" cy="50" r="49.5" />
          </svg>
          <div class="pos">
            <div class="planet">
              <dl class="infos">
                <dt>Jupiter</dt>
                <dd><span>{dataLabel}</span></dd>
                <dd class="value">{planets[5][scaleMode]}</dd>
              </dl>
            </div>
          </div>
        </div>

        <!-- Saturn -->
        <div id="saturn" class="orbit">
          <svg class="orbit-ring" viewBox="0 0 100 100">
            <circle class="visual-ring" cx="50" cy="50" r="49.5" />
            <circle class="hitbox-ring" cx="50" cy="50" r="49.5" />
          </svg>
          <div class="pos">
            <div class="planet">
              <div class="ring" style="
                width: {ringSize}em; 
                height: {ringSize}em; 
                margin-top: -{ringSize/2}em; 
                margin-left: -{ringSize/2}em; 
                border: {ringBorderWidth}em solid {ringColor}; 
                box-shadow: 0 0 {ringShadowBlur}px {ringShadowColor};
                transform: rotateX({ringTiltX}deg) rotateY({ringTiltY}deg);
              "></div>
              <dl class="infos">
                <dt>Saturn</dt>
                <dd><span>{dataLabel}</span></dd>
                <dd class="value">{planets[6][scaleMode]}</dd>
              </dl>
            </div>
          </div>
        </div>

        <!-- Uranus -->
        <div id="uranus" class="orbit">
          <svg class="orbit-ring" viewBox="0 0 100 100">
            <circle class="visual-ring" cx="50" cy="50" r="49.5" />
            <circle class="hitbox-ring" cx="50" cy="50" r="49.5" />
          </svg>
          <div class="pos">
            <div class="planet">
              <dl class="infos">
                <dt>Uranus</dt>
                <dd><span>{dataLabel}</span></dd>
                <dd class="value">{planets[7][scaleMode]}</dd>
              </dl>
            </div>
          </div>
        </div>

        <!-- Neptune -->
        <div id="neptune" class="orbit">
          <svg class="orbit-ring" viewBox="0 0 100 100">
            <circle class="visual-ring" cx="50" cy="50" r="49.5" />
            <circle class="hitbox-ring" cx="50" cy="50" r="49.5" />
          </svg>
          <div class="pos">
            <div class="planet">
              <dl class="infos">
                <dt>Neptune</dt>
                <dd><span>{dataLabel}</span></dd>
                <dd class="value">{planets[8][scaleMode]}</dd>
              </dl>
            </div>
          </div>
        </div>

        <!-- Pluto -->
        <div id="pluto" class="orbit">
          <svg class="orbit-ring" viewBox="0 0 100 100">
            <circle class="visual-ring" cx="50" cy="50" r="50" />
            <circle class="hitbox-ring" cx="50" cy="50" r="50" />
          </svg>
          <div class="pos">
            <div class="planet">
              <dl class="infos">
                <dt>Pluto</dt>
                <dd><span>{dataLabel}</span></dd>
                <dd class="value">{planets[9][scaleMode]}</dd>
              </dl>
            </div>
          </div>
        </div>

        <!-- Sun -->
        <div id="sun">
          <dl class="infos">
            <dt>Sun</dt>
            <dd><span>{activePlanet === 'sun' ? 'From Earth' : dataLabel}</span></dd>
            <dd class="value">{planets[0][scaleMode]}</dd>
          </dl>
        </div>

      </div>
    </div>
  </div>
</div>

<style>
  @import url('https://fonts.googleapis.com/css?family=Open+Sans:400,300');

  * {
    box-sizing: border-box;
  }

  .solar-system-container {
    position: relative;
    width: 100%;
    height: 100vh;
    font-size: 10px;
    font-family: 'Helvetica', sans-serif;
    font-weight: 700;
    background-color: transparent; /* Transparent background to show Galaxy */
    overflow: hidden;
    perspective: 1000px;
  }

  #universe {
    z-index: 1;
    position: absolute;
    overflow: hidden;
    width: 100%;
    height: 100%;
    transform-style: preserve-3d;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  #galaxy {
    position: relative;
    width: 100%;
    height: 100%;
    transform-style: preserve-3d;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  #solar-system {
    position: absolute;
    width: 100%;
    height: 100%;
    transform-style: preserve-3d;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .orbit {
    position: absolute;
    top: 50%;
    left: 50%;
    border: none;
    border-radius: 50%;
    transform-style: preserve-3d;
    animation-name: orbit;
    animation-iteration-count: infinite;
    animation-timing-function: linear;
    box-shadow: none;
    pointer-events: none;
  }

  #earth.orbit, #pluto.orbit {
    border: none;
    box-shadow: none;
    pointer-events: none; /* Container ignores events */
  }

  .orbit-ring {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none; /* SVG box ignores events */
    overflow: visible;
  }

  .orbit-ring circle {
    fill: none;
    vector-effect: non-scaling-stroke; /* Keep stroke thin regardless of scale */
    transition: stroke 0.3s, stroke-width 0.3s, filter 0.3s;
  }

  .orbit-ring .visual-ring {
    stroke: rgba(255, 255, 255, 0.2); /* Visible thin line by default */
    stroke-width: 2px; /* Thicker visual line */
    pointer-events: none; /* Click-through */
  }

  .orbit-ring .hitbox-ring {
    stroke: transparent; /* Invisible hit area */
    stroke-width: 100px; /* Extremely thick hit area for easier interaction */
    pointer-events: none; /* Disable interaction */
    cursor: default;
  }

  .orbit-ring:hover .visual-ring {
    stroke: rgba(255, 255, 255, 0.8); /* Brighter on hover */
    stroke-width: 4px; /* Thicker on hover */
    filter: drop-shadow(0 0 5px rgba(255, 255, 255, 0.5));
  }

  /* Z-Index Layering for Hover Detection - Inner planets on top */
  #sun { z-index: 100; }
  #mercury.orbit { z-index: 99; }
  #venus.orbit { z-index: 98; }
  #earth.orbit { z-index: 97; }
  #mars.orbit { z-index: 96; }
  #jupiter.orbit { z-index: 95; }
  #saturn.orbit { z-index: 94; }
  #uranus.orbit { z-index: 93; }
  #neptune.orbit { z-index: 92; }
  #pluto.orbit { z-index: 91; }

  .orbit .orbit {
    animation-name: suborbit;
  }

  .pos {
    position: absolute;
    top: 50%;
    width: 2em;
    height: 2em;
    margin-top: -1em;
    margin-left: -1em;
    transform-style: preserve-3d;
    animation-name: invert;
    animation-iteration-count: infinite;
    animation-timing-function: linear;
    pointer-events: none; /* Ensure pos container doesn't block but allows children */
  }

  #sun, .planet, .moon {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 1em;
    height: 1em;
    margin-top: -0.5em;
    margin-left: -0.5em;
    border-radius: 50%;
    transform-style: preserve-3d;
    pointer-events: auto; /* Explicitly enable pointer events for planets */
  }

  #sun {
    background-image: url('https://www.solarsystemscope.com/textures/download/8k_sun.jpg');
    background-size: cover;
    background-position: center;
    box-shadow: 0 0 60px rgba(255, 160, 60, 0.6), 0 0 120px rgba(255, 100, 0, 0.3);
    pointer-events: auto;
  }

  .planet {
    background-color: #202020;
    background-repeat: no-repeat;
    background-size: cover;
    background-position: center;
    animation-iteration-count: infinite;
    animation-timing-function: linear;
    box-shadow: inset -10px -10px 20px rgba(0,0,0,0.9), inset 2px 2px 5px rgba(255,255,255,0.15);
    pointer-events: none; /* Disable interaction */
    cursor: default;
    transition: box-shadow 0.3s ease, transform 0.3s ease;
  }

  /* Larger hitbox for planets */
  .planet::after {
    content: '';
    position: absolute;
    top: -200%;
    left: -200%;
    width: 500%;
    height: 500%;
    border-radius: 50%;
    cursor: default;
    pointer-events: none;
  }

  /* Pause animation on hover over orbit */
  .orbit:hover,
  .orbit:hover .pos,
  .orbit:hover .planet,
  .orbit:hover .moon,
  .orbit:hover .ring {
    animation-play-state: paused;
  }

  .planet:hover {
    box-shadow: inset -10px -10px 20px rgba(0,0,0,0.9), inset 2px 2px 5px rgba(255,255,255,0.15), 0 0 20px rgba(255, 255, 255, 0.6);
    transform: scale(1.2);
  }

  #earth.orbit:hover, #pluto.orbit:hover {
    /* Removed border styles as they are now handled by SVG hover */
    cursor: default; /* Container itself shouldn't show pointer */
  }

  /* Planet Specific Textures */
  #mercury .planet { background-image: url('https://www.solarsystemscope.com/textures/download/2k_mercury.jpg'); }
  #venus .planet { background-image: url('https://www.solarsystemscope.com/textures/download/2k_venus_surface.jpg'); }
  #earth .planet { background-image: url('https://www.solarsystemscope.com/textures/download/2k_earth_daymap.jpg'); box-shadow: inset -10px -10px 20px rgba(0,0,0,0.9), 0 0 5px rgba(100, 200, 255, 0.4); }
  #mars .planet { background-image: url('https://www.solarsystemscope.com/textures/download/2k_mars.jpg'); }
  #jupiter .planet { background-image: url('https://www.solarsystemscope.com/textures/download/2k_jupiter.jpg'); }
  #saturn .planet { background-image: url('https://www.solarsystemscope.com/textures/download/2k_saturn.jpg'); }
  #uranus .planet { background-image: url('https://www.solarsystemscope.com/textures/download/2k_uranus.jpg'); }
  #neptune .planet { background-image: url('https://www.solarsystemscope.com/textures/download/2k_neptune.jpg'); }
  #pluto .planet { background-image: url('https://upload.wikimedia.org/wikipedia/commons/thumb/e/ef/Pluto_in_True_Color_-_High-Res.jpg/800px-Pluto_in_True_Color_-_High-Res.jpg'); box-shadow: inset -6px -6px 12px rgba(0,0,0,0.9), 0 0 15px rgba(255, 255, 255, 0.3); }

  /* Extra large hitbox for Pluto */
  #pluto .planet::after {
    top: -1000%;
    left: -1000%;
    width: 2100%;
    height: 2100%;
  }

  /* Extra large hitbox for Pluto's orbit */
  #pluto.orbit .orbit-ring .hitbox-ring {
    stroke-width: 180px;
  }

  .moon {
    background-image: url('https://www.solarsystemscope.com/textures/download/2k_moon.jpg');
    background-size: cover;
    box-shadow: inset -4px -4px 8px rgba(0,0,0,0.9);
  }

  .ring {
    position: absolute;
    top: 50%;
    left: 50%;
    border-radius: 50%;
  }

  #saturn .ring {
    width: 2em;
    height: 2em;
    margin-top: -1em;
    margin-left: -1em;
    border: 0.3em solid rgba(160, 147, 130, 0.7);
    box-shadow: 0 0 10px rgba(160, 147, 130, 0.3);
  }

  /* Animations */
  @keyframes orbit {
    0% { transform: rotateZ(0deg); }
    100% { transform: rotateZ(-360deg); }
  }

  /* Pluto's inclined orbit (approx 17 degrees) */
  @keyframes orbit-pluto {
    0% { transform: rotateY(17deg) rotateZ(0deg); }
    100% { transform: rotateY(17deg) rotateZ(-360deg); }
  }

  @keyframes suborbit {
    0% { transform: rotateX(90deg) rotateZ(0deg); }
    100% { transform: rotateX(90deg) rotateZ(-360deg); }
  }

  @keyframes invert {
    0% { transform: rotateX(-90deg) rotateY(360deg) rotateZ(0deg); }
    100% { transform: rotateX(-90deg) rotateY(0deg) rotateZ(0deg); }
  }

  /* Planet Animations (Orbital Periods in Earth Years) */
  #mercury .pos, #mercury .planet, #mercury.orbit { animation-duration: 2.89s; }
  #venus .pos, #venus .planet, #venus.orbit { animation-duration: 7.38s; }
  #earth .pos, #earth .planet, #earth.orbit { animation-duration: 12s; }
  #earth .orbit .pos, #earth .orbit { animation-duration: 0.90s; }
  #mars .pos, #mars .planet, #mars.orbit { animation-duration: 22.57s; }
  #jupiter .pos, #jupiter .planet, #jupiter.orbit { animation-duration: 142.35s; }
  #saturn .pos, #saturn .planet, #saturn.orbit, #saturn .ring { animation-duration: 353.37s; }
  #uranus .pos, #uranus .planet, #uranus.orbit { animation-duration: 1008.20s; }
  #neptune .pos, #neptune .planet, #neptune.orbit { animation-duration: 1977.50s; }
  #pluto .pos, #pluto .planet, #pluto.orbit { animation-duration: 2975.00s; }
  
  /* Apply inclined orbit animation to Pluto */
  #pluto.orbit {
    animation-name: orbit-pluto;
  }

  /* 3D View */
  /* .view-3D #solar-system is now handled via inline style for dynamic tilt */

  .view-3D #sun {
    transform: rotateX(-90deg);
  }

  .view-3D .ring {
    transform: rotateX(90deg);
  }

  .view-3D .planet,
  .view-3D .moon {
    transform: rotateX(27deg); /* Counter-rotate to face camera at base tilt */
  }

  /* 2D View */
  .view-2D #sun,
  .view-2D .ring {
    transform: rotateX(0deg);
  }

  .view-2D .planet,
  .view-2D .moon {
    transform: rotateX(90deg);
  }

  /* Zoom Large */
  .zoom-large #solar-system {
    width: 100%;
  }

  .zoom-large.view-2D .scale-stretched #solar-system { font-size: 26%; }
  .zoom-large.view-3D .scale-stretched #solar-system { font-size: 62%; }

  /* Stretched Scale (Speed) */
  .scale-stretched #sun { font-size: 24em; }
  .scale-stretched #mercury .planet { font-size: 1.5em; }
  .scale-stretched #venus .planet { font-size: 3.72em; }
  .scale-stretched #earth .planet { font-size: 3.92em; }
  .scale-stretched #earth .moon { font-size: 1.2em; }
  .scale-stretched #mars .planet { font-size: 2.9em; }
  .scale-stretched #jupiter .planet { font-size: 12em; }
  .scale-stretched #saturn .planet { font-size: 10.8em; }
  .scale-stretched #uranus .planet { font-size: 4.68em; }
  .scale-stretched #neptune .planet { font-size: 4.9em; }
  .scale-stretched #pluto .planet { font-size: 1.5em; }
  
  /* Active Planet Sizes - Make them larger when selected */
  .scale-stretched .mercury #mercury .planet { font-size: 6em; }
  .scale-stretched .venus #venus .planet { font-size: 8em; }
  .scale-stretched .earth #earth .planet { font-size: 8em; }
  .scale-stretched .mars #mars .planet { font-size: 6em; }
  .scale-stretched .jupiter #jupiter .planet { font-size: 18em; }
  .scale-stretched .saturn #saturn .planet { font-size: 16em; }
  .scale-stretched .uranus #uranus .planet { font-size: 12em; }
  .scale-stretched .neptune #neptune .planet { font-size: 12em; }
  .scale-stretched .pluto #pluto .planet { font-size: 6em; }

  #solar-system {
    transition: transform 2.5s cubic-bezier(0.25, 0.1, 0.25, 1);
  }

  /* Orbits - Stretched */
  .scale-stretched #mercury.orbit { width: 32em; height: 32em; margin-top: -16em; margin-left: -16em; }
  .scale-stretched #venus.orbit { width: 46em; height: 46em; margin-top: -23em; margin-left: -23em; }
  .scale-stretched #earth.orbit { width: 64em; height: 64em; margin-top: -32em; margin-left: -32em; }
  .scale-stretched #earth .orbit { width: 6em; height: 6em; margin-top: -3em; margin-left: -3em; }
  .scale-stretched #mars.orbit { width: 84em; height: 84em; margin-top: -42em; margin-left: -42em; }
  .scale-stretched #jupiter.orbit { width: 120em; height: 120em; margin-top: -60em; margin-left: -60em; }
  .scale-stretched #saturn.orbit { width: 170em; height: 170em; margin-top: -85em; margin-left: -85em; }
  .scale-stretched #uranus.orbit { width: 210em; height: 210em; margin-top: -105em; margin-left: -105em; }
  .scale-stretched #neptune.orbit { width: 250em; height: 250em; margin-top: -125em; margin-left: -125em; }
  .scale-stretched #pluto.orbit { width: 290em; height: 290em; margin-top: -145em; margin-left: -145em; }

  /* Planet Starting Positions */
  #mercury .pos { left: 50%; top: -1%; }
  #venus .pos { left: 0; top: 50%; }
  #earth .pos { left: 100%; top: 50%; }
  #mars .pos { left: 50%; top: 100%; }
  #jupiter .pos { left: 100%; top: 50%; }
  #saturn .pos { left: 0%; top: 50%; }
  #uranus .pos { left: 0; top: 50%; }
  #neptune .pos { left: 50%; top: 0; }
  #pluto .pos { left: 50%; top: 100%; }
  #earth .orbit .pos { left: 100%; top: 50%; }

  /* UI Elements */
  #navbar, #controls, #data {
    background: rgba(0, 0, 0, 0.4);
    backdrop-filter: blur(10px);
  }

  #navbar {
    z-index: 99;
    position: absolute;
    top: 0;
    left: 0;
    padding: 16px;
    width: 100%;
    height: 48px;
  }

  #navbar a, #data a, #controls label {
    color: rgba(255, 255, 255, 0.6);
    display: block;
    position: relative;
    text-decoration: none;
    cursor: pointer;
  }

  #navbar a:hover, #data a:hover, #controls label:hover {
    color: #FFF;
  }

  #data a.active {
    color: #0CF;
  }

  #navbar a {
    position: absolute;
    top: 0;
    height: 48px;
    padding: 16px;
    font-size: 14px;
  }

  #toggle-data {
    left: 0;
  }

  .view-controls {
    position: absolute;
    top: 0;
    right: 0;
    height: 48px;
    padding: 0 16px;
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .view-label {
    font-size: 13px;
    color: rgba(255, 255, 255, 0.4);
    font-weight: 600;
    transition: color 0.3s;
    letter-spacing: 0.5px;
  }

  .view-label.active {
    color: #FFF;
  }

  .switch {
    position: relative;
    display: inline-block;
    width: 44px;
    height: 24px;
  }

  .switch input {
    opacity: 0;
    width: 0;
    height: 0;
  }

  .slider {
    position: absolute;
    cursor: pointer;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(255, 255, 255, 0.1);
    transition: .4s;
    border-radius: 24px;
    border: 1px solid rgba(255, 255, 255, 0.3);
    backdrop-filter: blur(4px);
  }

  .slider:before {
    position: absolute;
    content: "";
    height: 18px;
    width: 18px;
    left: 2px;
    bottom: 2px;
    background-color: white;
    transition: .4s cubic-bezier(0.4, 0.0, 0.2, 1);
    border-radius: 50%;
    box-shadow: 0 2px 4px rgba(0,0,0,0.2);
  }

  input:checked + .slider {
    background-color: rgba(255, 255, 255, 0.25);
    border-color: rgba(255, 255, 255, 0.6);
  }

  input:checked + .slider:before {
    transform: translateX(20px);
  }

  h1 {
    width: 100%;
    font-weight: 600;
    font-size: 14px;
    text-align: center;
    color: rgba(255, 255, 255, 0.8);
    margin: 0;
  }

  h1 span {
    display: inline;
    font-weight: 300;
    font-size: 14px;
    color: rgba(255, 255, 255, 0.5);
  }

  #data {
    z-index: 99;
    position: fixed;
    bottom: 30px;
    left: 50%;
    transform: translateX(-50%);
    top: auto;
    padding: 10px 20px;
    background: rgba(0, 0, 0, 0.6);
    border-radius: 30px;
    display: flex;
    gap: 15px;
    transition: all 0.3s ease;
    width: auto;
    opacity: 1;
  }

  .data-close #data {
    bottom: -100px;
  }

  .data-open #data {
    bottom: 30px;
  }

  #data a {
    margin-bottom: 0;
    padding: 6px 10px;
    font-size: 14px;
    white-space: nowrap;
  }

  /* Info Labels */
  dl.infos {
    position: absolute;
    display: block;
    opacity: 0;
    width: 100%;
    height: 100%;
    margin-top: -90%;
    margin-left: 90%;
    padding-left: 100%;
    transform-origin: 100% 100%;
    transform-style: preserve-3d;
    transform: rotateX(90deg);
    transition: opacity 0.3s ease, transform 0.3s ease;
  }

  dl.infos:before {
    position: absolute;
    content: '';
    width: 15px;
    height: 30px;
    left: 15px;
    bottom: 0;
    border-top: 1px solid white;
    border-left: 1px solid white;
    transform-style: preserve-3d;
    transform: skew(-45deg, 0deg);
    box-shadow: inset 1px 1px black;
  }

  dl.infos dt {
    position: absolute;
    left: 50px;
    bottom: 30px;
    color: #FFF;
    font-size: 14px;
    text-shadow: 1px 1px 2px black;
  }

  dl.infos dd.value {
    position: absolute;
    left: 50px;
    bottom: 30px;
    width: 300px;
    color: #FFF;
    font-size: 22px;
    text-shadow: 1px 1px 2px black;
  }

  dl.infos dd span {
    position: absolute;
    left: 50px;
    bottom: 14px;
    width: 300px;
    color: #FFF;
    font-size: 11px;
    text-shadow: 1px 1px 2px black;
  }

  .sun #sun .infos,
  .mercury #mercury .infos,
  .venus #venus .infos,
  .earth #earth .infos,
  .mars #mars .infos,
  .jupiter #jupiter .infos,
  .saturn #saturn .infos,
  .uranus #uranus .infos,
  .neptune #neptune .infos {
    opacity: 1;
    transform: rotateX(0deg);
  }

  .pluto #pluto .infos {
    opacity: 1;
    transform: rotateX(0deg) rotateZ(-17deg);
  }

  /* Active planet orbit visible (highlighted) */
  .mercury #mercury.orbit .orbit-ring .visual-ring,
  .venus #venus.orbit .orbit-ring .visual-ring,
  .earth #earth.orbit .orbit-ring .visual-ring,
  .mars #mars.orbit .orbit-ring .visual-ring,
  .jupiter #jupiter.orbit .orbit-ring .visual-ring,
  .saturn #saturn.orbit .orbit-ring .visual-ring,
  .uranus #uranus.orbit .orbit-ring .visual-ring,
  .neptune #neptune.orbit .orbit-ring .visual-ring,
  .pluto #pluto.orbit .orbit-ring .visual-ring {
    stroke: rgba(255, 255, 255, 1);
    stroke-width: 3px;
    filter: drop-shadow(0 0 10px rgba(255, 255, 255, 0.8));
  }

  /* Hide UI State */
  .hide-UI h1,
  .hide-UI #data,
  .hide-UI dl.infos {
    opacity: 0 !important;
    margin-top: -30px;
  }

  .hide-UI #data {
    margin-bottom: -30px;
  }

  .hide-UI #earth.orbit .orbit-ring .visual-ring,
  .hide-UI #pluto.orbit .orbit-ring .visual-ring {
    stroke: rgba(255, 255, 255, 0.2) !important;
    filter: none !important;
  }

  /* Opening Animation */
  .opening #sun,
  .opening .orbit,
  .opening .pos,
  .opening .planet,
  .opening .ring {
    transition-duration: 4s;
  }

  .opening #sun {
    box-shadow: 0 0 0 rgba(255, 160, 60, 0);
  }

  /* Transitions */
  #solar-system, .orbit, .planet, .ring {
    transition: all 0.8s ease-in-out;
  }

  h1, #data {
    transition: opacity 0.8s ease-in-out, margin 0.8s ease-in-out;
  }

  /* Debug Panel Styles */
  .debug-panel {
    position: absolute;
    top: 80px;
    right: 20px;
    background: rgba(0, 0, 0, 0.8);
    padding: 20px;
    border-radius: 10px;
    z-index: 10000;
    color: white;
    width: 250px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    pointer-events: auto;
    font-family: 'Helvetica', sans-serif;
    font-weight: 700;
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
  
  .control-group input[type="text"] {
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

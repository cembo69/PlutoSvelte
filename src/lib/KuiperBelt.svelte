<script>
  import { onMount } from 'svelte';

  let canvas;
  let containerWidth;
  let containerHeight;
  
  // UI State
  let timeProgress = 100;
  let showKuiperBelt = true;
  let showOrbits = true;
  let showScientificInfo = false;
  let hoveredOrbit = null;
  let hoveredMission = null;
  let hoveredSun = false;
  
  // Narrative / Typewriter
  let displayedText = "";
  const narrativeText = `Der Kuipergürtel ist wie ein breiter Ring aus Eis- und Gesteinsbrocken weit hinter Neptun. Dort kreisen viele kleine „Miniplaneten“ wie Pluto um die Sonne.

WO ER IST
Der Ring beginnt kurz hinter Neptun und geht noch viele Milliarden Kilometer weiter nach außen. Er liegt ungefähr in derselben Ebene wie die Bahnen der Planeten und bildet eine große, flache Scheibe.

WORAUS ER BESTEHT
Im Kuipergürtel gibt es unzählige Brocken aus Eis und Gestein, von Staubkörnern bis zu Zwergplaneten. Aus manchen dieser Brocken werden kurzperiodische Kometen, die manchmal ins innere Sonnensystem kommen.`;

  // Animation State
  let isPlaying = false;
  let animationFrame;

  // Derived state for year display (approximate mapping)
  // 0% = 1970, 100% = 2030
  $: currentYear = Math.floor(1970 + (timeProgress / 100) * 60);

  // Milestones for New Horizons
  const milestones = [
    { year: 2006, label: "Launch", progress: 60 },
    { year: 2015, label: "Pluto Flyby", progress: 75 },
    { year: 2019, label: "Arrokoth Flyby", progress: 82 },
    { year: 2021, label: "50 AU Milestone", progress: 85 }
  ];

  // Check for active milestone
  $: activeMilestone = milestones.find(m => Math.abs(currentYear - m.year) <= 1);

  // Configuration
  const VIEWBOX_WIDTH = 2000;
  const VIEWBOX_HEIGHT = 1200;
  
  const colors = {
    saturn: '#4ade80',
    uranus: '#e879f9',
    neptune: '#fb923c',
    pluto: '#facc15',
    path: '#22d3ee',
    newHorizons: '#ef4444',
    sun: '#f97316',
    text: '#ffffff'
  };

  const orbits = [
    { name: "Saturn", color: colors.saturn, rx: 150, ry: 100, rot: 0, type: "Gas Giant", dist: "9.5 AU", desc: "Ringed Planet" },
    { name: "Uranus", color: colors.uranus, rx: 250, ry: 180, rot: 0, type: "Ice Giant", dist: "19.8 AU", desc: "Tilted Axis" },
    { name: "Neptune", color: colors.neptune, rx: 350, ry: 260, rot: 0, type: "Ice Giant", dist: "30.1 AU", desc: "Windy World" },
    { name: "Pluto", color: colors.pluto, rx: 450, ry: 320, rot: -15, type: "Dwarf Planet", dist: "39.5 AU", desc: "Kuiper King" }
  ];

  const missions = [
    {
      name: "Pioneer 10",
      color: colors.path,
      dashed: true,
      d: "M 0 0 Q 200 50 1200 -300",
      endLabel: "Aldebaran",
      launchYear: 1972,
      type: "Flyby Probe",
      desc: "First to Jupiter",
      details: "Heading toward Aldebaran (Taurus)"
    },
    {
      name: "Pioneer 11",
      color: colors.path,
      dashed: true,
      d: "M 0 0 Q -100 50 -1000 150",
      endLabel: "Scutum",
      launchYear: 1973,
      type: "Flyby Probe",
      desc: "Jupiter & Saturn",
      details: "Heading toward Scutum constellation"
    },
    {
      name: "Voyager 1",
      color: colors.path,
      dashed: true,
      d: "M 0 0 Q 100 -200 300 -800",
      endLabel: "Ophiuchus",
      launchYear: 1977,
      type: "Interstellar",
      desc: "Furthest Human Object",
      details: "Entered Interstellar Space in 2012"
    },
    {
      name: "Voyager 2",
      color: colors.path,
      dashed: true,
      d: "M 0 0 Q 100 200 200 800",
      endLabel: "Pavo",
      launchYear: 1977,
      type: "Grand Tour",
      desc: "Jup, Sat, Ura, Nep",
      details: "Only probe to visit Uranus & Neptune"
    },
    {
      name: "New Horizons",
      color: colors.newHorizons,
      dashed: false,
      d: "M 0 0 L 800 160 L 1200 200",
      endLabel: "Kuiper Belt",
      launchYear: 2006,
      type: "Kuiper Mission",
      desc: "Pluto & Arrokoth",
      details: "Fastest launch from Earth ever"
    }
  ];

  // Path logic
  let pathElements = {};
  let probePositions = {};

  // Reactive statement to update probe positions based on timeProgress
  $: {
    missions.forEach(m => {
      if (pathElements[m.name]) {
        const len = pathElements[m.name].getTotalLength();
        
        // Calculate mission-specific progress
        // If current year is before launch, stay at 0
        if (currentYear < m.launchYear) {
          probePositions[m.name] = pathElements[m.name].getPointAtLength(0);
        } else {
          // Progress from Launch (0%) to 2030 (100% of visible path)
          // We need to define what "100%" means. Let's say the path drawn is the trajectory up to 2030+
          // Actually, let's map the path length to a specific timeframe or speed.
          // Simpler: Map the remaining time (Launch -> 2030) to the path length.
          // So at 2030, it reaches the end of the drawn line.
          
          const totalMissionYears = 2035 - m.launchYear; // Extend a bit beyond 2030 for visual continuity
          const yearsElapsed = currentYear - m.launchYear;
          const progress = Math.max(0, Math.min(1, yearsElapsed / totalMissionYears));
          
          const point = pathElements[m.name].getPointAtLength(len * progress);
          probePositions[m.name] = point;
        }
      }
    });
    // Force update for Svelte reactivity
    probePositions = probePositions;
  }

  // Particles Data
  let particles = [];
  
  function initParticles() {
    particles = [];
    const count = 4000;
    
    // Scale: Neptune (30 AU) = 350px (rx of Neptune orbit)
    // 1 AU = 11.66px
    const au = 350 / 30;

    for (let i = 0; i < count; i++) {
      const r = Math.random();
      let x, y, opacity, size;
      
      // 80% Classical Belt (30-50 AU) - "Cold Population"
      // Flacher Ring, kreisförmig
      if (Math.random() > 0.2) {
        const dist = 30 + r * 20; // 30-50 AU
        const angle = Math.random() * Math.PI * 2;
        const rx = dist * au;
        const ry = rx * 0.7; // Aspect ratio of the view (approx)
        
        // Wenig Streuung
        x = Math.cos(angle) * rx + (Math.random() - 0.5) * 15;
        y = Math.sin(angle) * ry + (Math.random() - 0.5) * 10;
        opacity = Math.random() * 0.5 + 0.2;
        size = Math.random() * 1.2 + 0.5;
      } 
      // 20% Scattered Disc (30-100 AU) - "Hot Population"
      // Exzentrisch, stärker geneigt (mehr vertikale Streuung im 2D Bild)
      else {
        // Bias towards inner edge, but extending far out
        const dist = 30 + (Math.pow(r, 2)) * 70; // 30-100 AU
        const angle = Math.random() * Math.PI * 2;
        
        const rx = dist * au;
        const ry = rx * 0.65; 
        
        // Mehr Chaos
        x = Math.cos(angle) * rx + (Math.random() - 0.5) * 40;
        y = Math.sin(angle) * ry + (Math.random() - 0.5) * 40;
        opacity = Math.random() * 0.3 + 0.1;
        size = Math.random() * 1.5 + 0.5;
      }

      particles.push({ x, y, size, opacity });
    }
  }

  // Canvas Drawing
  onMount(() => {
    initParticles();
    const ctx = canvas.getContext('2d');
    
    const draw = () => {
      if (!canvas) return;
      
      // Resize canvas to match container
      if (canvas.width !== containerWidth || canvas.height !== containerHeight) {
        canvas.width = containerWidth;
        canvas.height = containerHeight;
      }
      
      ctx.clearRect(0, 0, containerWidth, containerHeight);
      
      if (showKuiperBelt) {
        // Calculate scale to match SVG "meet" aspect ratio
        const scaleX = containerWidth / VIEWBOX_WIDTH;
        const scaleY = containerHeight / VIEWBOX_HEIGHT;
        const scale = Math.min(scaleX, scaleY);
        
        ctx.save();
        // Center the context
        ctx.translate(containerWidth / 2, containerHeight / 2);
        ctx.scale(scale, scale);
        
        // Draw Particles
        particles.forEach(p => {
          ctx.fillStyle = `rgba(200, 200, 220, ${p.opacity})`;
          ctx.beginPath();
          ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
          ctx.fill();
        });
        
        ctx.restore();
      }
      
      requestAnimationFrame(draw);
    };
    
    draw();
    startTypewriter();
  });

  function startTypewriter() {
    let i = 0;
    const speed = 30; // ms per char
    
    function type() {
      if (i < narrativeText.length) {
        displayedText += narrativeText.charAt(i);
        i++;
        setTimeout(type, speed);
      }
    }
    
    type();
  }

  function togglePlay() {
    if (timeProgress >= 100) {
      timeProgress = 0;
    }
    isPlaying = !isPlaying;
    if (isPlaying) {
      animateTimeline();
    } else {
      cancelAnimationFrame(animationFrame);
    }
  }

  function animateTimeline() {
    if (!isPlaying) return;
    
    // Increment progress (adjust speed here)
    timeProgress += 0.05; 
    
    if (timeProgress >= 100) {
      timeProgress = 100;
      isPlaying = false;
      return;
    }

    animationFrame = requestAnimationFrame(animateTimeline);
  }
</script>

<div class="kuiper-container" bind:clientWidth={containerWidth} bind:clientHeight={containerHeight}>
  
  <!-- Layer 1: Canvas (Kuiper Belt) -->
  <canvas bind:this={canvas}></canvas>

  <!-- Layer 2: SVG (Orbits & Paths) -->
  <svg viewBox="{-VIEWBOX_WIDTH/2} {-VIEWBOX_HEIGHT/2} {VIEWBOX_WIDTH} {VIEWBOX_HEIGHT}" preserveAspectRatio="xMidYMid meet">
    <defs>
      <marker id="arrow-blue" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto">
        <path d="M0,0 L0,6 L6,3 z" fill={colors.path} />
      </marker>
      <marker id="arrow-red" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto">
        <path d="M0,0 L0,6 L6,3 z" fill={colors.newHorizons} />
      </marker>
    </defs>

    <!-- Sun -->
    <g 
      on:mouseenter={() => hoveredSun = true}
      on:mouseleave={() => hoveredSun = false}
      style="cursor: help;"
    >
      <circle cx="0" cy="0" r="12" fill={colors.sun} filter="drop-shadow(0 0 15px {colors.sun})" />
      <text x="0" y="25" text-anchor="middle" fill={colors.sun} font-size="10" opacity="0.8">SUN</text>
    </g>

    <!-- Distance Markers / Zones -->
    {#if showOrbits}
      <g class="distance-markers" style="pointer-events: none;">
        <!-- 30 AU (Neptune / Inner Edge) -->
        <circle cx="0" cy="0" r="350" fill="none" stroke="rgba(255,255,255,0.1)" stroke-dasharray="4,4" />
        <text x="0" y="-360" text-anchor="middle" fill="rgba(255,255,255,0.3)" font-size="10">30 AU (Inner Edge)</text>
        
        <!-- 50 AU (Outer Edge / Cliff) -->
        <circle cx="0" cy="0" r="{350 * (50/30)}" fill="none" stroke="rgba(255,255,255,0.1)" stroke-dasharray="4,4" />
        <text x="0" y="-{350 * (50/30) + 10}" text-anchor="middle" fill="rgba(255,255,255,0.3)" font-size="10">50 AU (Outer Edge)</text>
      </g>
    {/if}

    <!-- Orbits -->
    {#if showOrbits}
      {#each orbits as orbit}
        <g 
          class="orbit-group" 
          on:mouseenter={() => hoveredOrbit = orbit.name}
          on:mouseleave={() => hoveredOrbit = null}
          style="opacity: {hoveredOrbit && hoveredOrbit !== orbit.name ? 0.1 : (hoveredMission ? 0.1 : 1)}; transition: opacity 0.3s;"
        >
          <ellipse 
            cx="0" cy="0" 
            rx={orbit.rx} ry={orbit.ry} 
            fill="none" 
            stroke="#333" 
            stroke-width={hoveredOrbit === orbit.name ? 2 : 1}
            transform="rotate({orbit.rot})"
          />
          <!-- Planet Marker -->
          <circle 
            cx={orbit.rx} cy="0" r={hoveredOrbit === orbit.name ? 40 : 25} 
            fill={orbit.color} 
            transform="rotate({orbit.rot})"
          />
          <!-- Always visible label for planets, but subtle -->
          <text 
            x={orbit.rx + (hoveredOrbit === orbit.name ? 50 : 35)} y="5" 
            fill={orbit.color} 
            font-size={hoveredOrbit === orbit.name ? "24" : "12"} 
            font-weight={hoveredOrbit === orbit.name ? "bold" : "normal"}
            opacity={hoveredOrbit === orbit.name ? 1 : 0.7}
            transform="rotate({orbit.rot})"
            style="transition: all 0.3s ease;"
          >{orbit.name}</text>
        </g>
      {/each}
    {/if}

    <!-- Missions -->
    {#each missions as mission}
      <g 
        class="mission-group"
        on:mouseenter={() => hoveredMission = mission.name}
        on:mouseleave={() => hoveredMission = null}
        style="opacity: {hoveredMission && hoveredMission !== mission.name ? 0.1 : (hoveredOrbit ? 0.1 : 1)}; transition: opacity 0.3s;"
      >
        <!-- Path -->
        <path 
          bind:this={pathElements[mission.name]}
          d={mission.d} 
          fill="none" 
          stroke={mission.color} 
          stroke-width={mission.name === "New Horizons" ? 3 : 1.5}
          stroke-dasharray={mission.dashed ? "5,5" : "none"}
          opacity="0.6"
        />
        
        <!-- Probe Icon -->
        {#if probePositions[mission.name]}
          <g transform="translate({probePositions[mission.name].x}, {probePositions[mission.name].y})">
            <!-- Pulsing effect for active mission -->
            {#if hoveredMission === mission.name || mission.name === "New Horizons"}
              <circle r="15" fill={mission.color} opacity="0.2">
                <animate attributeName="r" values="10;20;10" dur="2s" repeatCount="indefinite" />
                <animate attributeName="opacity" values="0.2;0;0.2" dur="2s" repeatCount="indefinite" />
              </circle>
            {/if}
            
            <circle r="5" fill={mission.color} stroke="#000" stroke-width="1" />
            
            <!-- Label always visible but subtle, bold on hover -->
            <text 
              x="10" y="5" 
              fill="white" 
              font-size={hoveredMission === mission.name ? "18" : "12"} 
              font-weight={hoveredMission === mission.name ? "bold" : "normal"}
              opacity={hoveredMission === mission.name || mission.name === "New Horizons" ? 1 : 0.7}
              style="transition: all 0.3s ease; text-shadow: 0 0 5px black;"
            >{mission.name}</text>
            
            <!-- Milestone Label for New Horizons -->
            {#if mission.name === "New Horizons" && activeMilestone}
              <g transform="translate(10, -20)">
                <rect x="-5" y="-15" width="140" height="25" fill="rgba(239, 68, 68, 0.9)" rx="4" />
                <text x="5" y="2" fill="white" font-size="12" font-weight="bold">
                  {activeMilestone.year}: {activeMilestone.label}
                </text>
              </g>
            {/if}
          </g>
        {/if}
      </g>
    {/each}

    <!-- Kuiper Belt Label (Integrated) -->
    {#if showKuiperBelt}
      <g opacity="0.4">
        <path id="curve" d="M -600 300 Q 0 500 600 300" fill="transparent" />
        <text width="500">
          <textPath xlink:href="#curve" startOffset="50%" text-anchor="middle" fill="#fff" font-size="24" letter-spacing="10" font-weight="bold">
            KUIPER BELT REGION
          </textPath>
        </text>
      </g>
    {/if}

    <!-- Disclaimer -->
    <text x="900" y="550" text-anchor="end" fill="#555" font-size="12">Illustration not to scale</text>
  </svg>

  <!-- UI Layer (Swiss Style) -->
  <div class="ui-layer">
    <header>
      <div class="header-left">
        <h1>Outer Solar System</h1>
        <div class="subtitle">Kuiper Belt Exploration</div>
      </div>
      <div class="year-display">
        <span class="label">YEAR</span>
        <span class="value">{currentYear}</span>
      </div>
    </header>

    <aside class="data-panel">
      {#if hoveredOrbit}
        {@const orbitData = orbits.find(o => o.name === hoveredOrbit)}
        <div class="data-block">
          <span class="label">TARGET</span>
          <span class="value" style="color: {orbitData.color}">{hoveredOrbit.toUpperCase()}</span>
          <div class="detail">{orbitData.desc}</div>
        </div>
        <div class="data-block">
          <span class="label">CLASSIFICATION</span>
          <span class="value">{orbitData.type}</span>
        </div>
        <div class="data-block">
          <span class="label">DISTANCE</span>
          <span class="value">{orbitData.dist}</span>
        </div>
      {:else if hoveredSun}
        <div class="data-block">
          <span class="label">TARGET</span>
          <span class="value" style="color: {colors.sun}">SUN</span>
          <div class="detail">Star (G2V Type)</div>
        </div>
        <div class="data-block">
          <span class="label">MASS</span>
          <span class="value">1.989 × 10^30 kg</span>
        </div>
        <div class="data-block">
          <span class="label">ROLE</span>
          <span class="value">Gravitational Center</span>
        </div>
      {:else if hoveredMission}
        {@const missionData = missions.find(m => m.name === hoveredMission)}
        <div class="data-block">
          <span class="label">MISSION</span>
          <span class="value" style="color: {missionData.color}">{hoveredMission.toUpperCase()}</span>
          <div class="detail">{missionData.desc}</div>
        </div>
        <div class="data-block">
          <span class="label">LAUNCH</span>
          <span class="value">{missionData.launchYear}</span>
        </div>
        <div class="data-block">
          <span class="label">DETAILS</span>
          <span class="value" style="font-size: 14px; line-height: 1.4;">{missionData.details}</span>
        </div>
      {:else if activeMilestone}
        <div class="data-block">
          <span class="label">MILESTONE</span>
          <span class="value" style="color: #ef4444">{activeMilestone.year}</span>
          <div class="detail">{activeMilestone.label}</div>
        </div>
      {:else}
        <div class="data-block">
          <span class="label">STATUS</span>
          <span class="value">MONITORING</span>
          <div class="detail">System Normal</div>
        </div>
      {/if}
      
      {#if showScientificInfo}
        <div class="scientific-grid">
          <div class="data-row">
            <span class="label">REGION</span>
            <span class="value">30–55 AU</span>
          </div>
          <div class="data-row">
            <span class="label">TYPE</span>
            <span class="value">Cold Classical</span>
          </div>
          <div class="data-row">
            <span class="label">OBJECTS</span>
            <span class="value">>100,000</span>
          </div>
        </div>
      {/if}
    </aside>

    <!-- Narrative Layer -->
    <div class="narrative-panel">
      {displayedText}<span class="cursor">_</span>
    </div>

    <footer>
      <div class="controls-left">
        <button class="text-btn" on:click={togglePlay}>
          {isPlaying ? 'STOP' : 'PLAY'}
        </button>
        <div class="timeline-wrapper">
          <input type="range" min="0" max="100" bind:value={timeProgress} />
        </div>
      </div>
      
      <div class="controls-right">
        <button class:active={showScientificInfo} on:click={() => showScientificInfo = !showScientificInfo}>DATA</button>
        <button class:active={showKuiperBelt} on:click={() => showKuiperBelt = !showKuiperBelt}>BELT</button>
        <button class:active={showOrbits} on:click={() => showOrbits = !showOrbits}>ORBITS</button>
      </div>
    </footer>
  </div>
</div>

<style>
  /* @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;800&display=swap'); */

  .kuiper-container {
    position: relative;
    width: 100%;
    height: 100vh;
    background: #050505;
    overflow: hidden;
    font-family: 'Helvetica', sans-serif;
    font-weight: 700;
    color: #fff;
  }

  canvas, svg {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
  }

  canvas { z-index: 1; }
  svg { z-index: 2; }

  /* UI Layer - Swiss Style Grid */
  .ui-layer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 10;
    display: grid;
    grid-template-rows: auto 1fr auto;
    grid-template-columns: 300px 1fr;
    pointer-events: none; /* Let clicks pass through to SVG/Canvas */
    padding: 40px;
    box-sizing: border-box;
  }

  /* Header */
  header {
    grid-column: 1 / -1;
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    border-bottom: 1px solid rgba(255, 255, 255, 0.2);
    padding-bottom: 20px;
    pointer-events: auto;
  }

  h1 {
    font-size: 24px;
    font-weight: 700;
    text-transform: uppercase;
    margin: 0;
    letter-spacing: -0.5px;
    line-height: 1;
  }

  .subtitle {
    font-size: 12px;
    color: #888;
    margin-top: 5px;
    text-transform: uppercase;
    letter-spacing: 1px;
  }

  .year-display {
    text-align: right;
  }

  .year-display .label {
    display: block;
    font-size: 10px;
    color: #666;
    margin-bottom: 2px;
  }

  .year-display .value {
    font-size: 48px;
    font-weight: 700;
    line-height: 0.8;
    letter-spacing: -2px;
  }

  /* Sidebar / Data Panel */
  .data-panel {
    grid-column: 1;
    padding-top: 40px;
    pointer-events: auto;
  }

  .data-block {
    margin-bottom: 40px;
  }

  .label {
    display: block;
    font-size: 10px;
    font-weight: 700;
    color: #666;
    text-transform: uppercase;
    letter-spacing: 1px;
    margin-bottom: 5px;
  }

  .value {
    display: block;
    font-size: 24px;
    font-weight: 700;
    margin-bottom: 5px;
  }

  .detail {
    font-size: 12px;
    color: #888;
  }

  .scientific-grid {
    border-top: 1px solid rgba(255, 255, 255, 0.2);
    padding-top: 20px;
    display: flex;
    flex-direction: column;
    gap: 15px;
  }

  .data-row .value {
    font-size: 14px;
    font-weight: 700;
    margin: 0;
  }

  /* Narrative Panel */
  .narrative-panel {
    grid-column: 2;
    grid-row: 2;
    align-self: end;
    justify-self: end;
    text-align: right;
    max-width: 400px;
    padding-bottom: 40px;
    padding-right: 0;
    font-family: 'Helvetica', sans-serif;
    font-weight: 700;
    font-size: 12px;
    line-height: 1.6;
    color: #aaa;
    white-space: pre-wrap;
    text-shadow: 0 0 2px black;
    pointer-events: none;
  }

  .cursor {
    animation: blink 1s step-end infinite;
  }

  @keyframes blink {
    50% { opacity: 0; }
  }

  /* Footer */
  footer {
    grid-column: 1 / -1;
    display: flex;
    justify-content: space-between;
    align-items: flex-end;
    border-top: 1px solid rgba(255, 255, 255, 0.2);
    padding-top: 20px;
    pointer-events: auto;
  }

  .controls-left {
    display: flex;
    align-items: center;
    gap: 20px;
    flex: 1;
  }

  .timeline-wrapper {
    flex: 1;
    max-width: 400px;
  }

  input[type="range"] {
    width: 100%;
    height: 2px;
    background: #333;
    appearance: none;
    outline: none;
  }

  input[type="range"]::-webkit-slider-thumb {
    appearance: none;
    width: 10px;
    height: 10px;
    background: #fff;
    border-radius: 0; /* Square thumb */
    cursor: pointer;
  }

  .controls-right {
    display: flex;
    gap: 1px; /* Gap for borders */
    background: rgba(255, 255, 255, 0.2); /* Border color */
  }

  button {
    background: transparent;
    border: none;
    color: #fff;
    font-family: inherit;
    font-size: 12px;
    font-weight: 600;
    text-transform: uppercase;
    cursor: pointer;
    padding: 10px 20px;
    transition: all 0.2s;
  }

  .text-btn {
    padding: 0;
    font-size: 14px;
    letter-spacing: 1px;
  }

  .text-btn:hover {
    color: #ef4444;
  }

  .controls-right button {
    background: #050505;
  }

  .controls-right button:hover {
    background: #111;
  }

  .controls-right button.active {
    background: #fff;
    color: #000;
  }
</style>
<script>
  import { onMount } from 'svelte';
  import SolarSystem3D from './lib/SolarSystem3D.svelte';
  import Galaxy from './lib/Galaxy.svelte';
  import Pluto3D from './lib/Pluto3D.svelte';
  import PlanetShowcase from './lib/PlanetShowcase.svelte';
  import PlanetShowcase2 from './lib/PlanetShowcase2.svelte';
  import PlutoCharonSystem from './lib/PlutoCharonSystem.svelte';
  import PlutoGame from './lib/PlutoGame.svelte';
  import DiscoveryJourney from './lib/DiscoveryJourney.svelte';

  // Bilder importieren
  import img1 from './bilderpluto/Bildschirmfoto 2025-12-03 um 03.15.43.png';
  import img2 from './bilderpluto/Bildschirmfoto 2025-12-03 um 03.15.59.png';
  import img3 from './bilderpluto/Bildschirmfoto 2025-12-03 um 03.16.10.png';
  import img4 from './bilderpluto/Bildschirmfoto 2025-12-03 um 03.16.17.png';
  import img5 from './bilderpluto/Bildschirmfoto 2025-12-03 um 03.16.26.png';
  import img6 from './bilderpluto/Bildschirmfoto 2025-12-03 um 03.16.33.png';
  import img7 from './bilderpluto/Bildschirmfoto 2025-12-03 um 03.16.40.png';
  import img8 from './bilderpluto/Bildschirmfoto 2025-12-03 um 03.16.46.png';
  import img9 from './bilderpluto/Bildschirmfoto 2025-12-03 um 03.16.55.png';
  import img10 from './bilderpluto/Bildschirmfoto 2025-12-03 um 03.17.10.png';
  import img11 from './bilderpluto/Bildschirmfoto 2025-12-03 um 03.17.20.png';

  // New Assets
  import plutoClose1 from './assets/closeup pluto.jpg';
  import plutoClose2 from './assets/plutocloseup2.jpg';
  import plutoClose3 from './assets/closeuppluto3.jpg';
  import charonClose1 from './assets/charoncloseup.avif';
  import charonClose2 from './assets/charon2.webp';
  
  // plutoCuteModel is now referenced directly via public path
  const plutoCuteModelPath = '/3Dmodelle/pluto_v2.glb';

  let scrollY = 0;
  let innerHeight = 0;
  let innerWidth = 0;
  let gameStarted = false;
  
  // Showcase 2 Overlay State
  let showSecondShowcase = false;
  let activeShowcaseData = null;

  function handleShowcaseClick(item) {
    activeShowcaseData = item;
    showSecondShowcase = true;
  }

  function handleBackClick() {
    showSecondShowcase = false;
    activeShowcaseData = null;
  }
  
  // Lichtgeschwindigkeits-Effekt beim Scrollen
  let lastScrollY = 0;
  let scrollVelocity = 0;
  let warpSpeed = 0;

  const discoveryStations = [
    {
      headline: "1905–1929: Die Suche nach Planet X",
      text: "Percival Lowell vermutet einen neunten Planeten jenseits von Neptun, der Bahnstörungen erklären soll. Systematische Suchprogramme beginnen am Lowell Observatory.",
      image: img1
    },
    {
      headline: "1930: Die Entdeckung",
      text: "Am 18. Februar entdeckt der junge Clyde Tombaugh einen beweglichen Lichtpunkt auf Fotoplatten. Am 13. März wird die Welt informiert: Pluto ist gefunden.",
      image: img2
    },
    {
      headline: "1978: Charon & Das Doppelsystem",
      text: "James Christy entdeckt Plutos riesigen Mond Charon. Dies ermöglicht erstmals präzise Massebestimmungen. Pluto entpuppt sich als winzig – viel kleiner als unser Mond.",
      image: img3
    },
    {
      headline: "1990er: Der Kuipergürtel",
      text: "Die Entdeckung weiterer eisiger Objekte jenseits von Neptun zeigt: Pluto ist nicht allein. Er ist Teil einer riesigen Trümmerwolke des frühen Sonnensystems.",
      image: img4
    },
    {
      headline: "2006: Start von New Horizons",
      text: "Die NASA startet die schnellste Sonde aller Zeiten. New Horizons hebt am 19. Januar ab, um als erste Mission überhaupt das Pluto-System zu besuchen.",
      image: img5
    },
    {
      headline: "2006: Die Degradierung",
      text: "Noch während New Horizons unterwegs ist, definiert die IAU 'Planet' neu. Pluto erfüllt das Kriterium der 'Bahnbereinigung' nicht und wird zum Zwergplaneten.",
      image: img6
    },
    {
      headline: "2007: Jupiter Swing-By",
      text: "New Horizons nutzt Jupiters Schwerkraft, um Schwung zu holen. Dabei entstehen spektakuläre Testbilder des Gasriesen und seiner Monde.",
      image: img7
    },
    {
      headline: "2015: Der historische Vorbeiflug",
      text: "Am 14. Juli rast die Sonde in nur 12.500 km Entfernung an Pluto vorbei. Die Bilder enthüllen eine aktive Welt mit Gletschern, Bergen und einer Atmosphäre.",
      image: img8
    },
    {
      headline: "2019: Arrokoth & Die Zukunft",
      text: "Die Reise geht weiter. New Horizons besucht Arrokoth, ein urtümliches Objekt des Kuipergürtels, und sendet bis heute Daten aus den Tiefen des Alls.",
      image: img9
    },
    {
      headline: "2021: 50 AE Meilenstein",
      text: "Am 17. April erreicht New Horizons eine Distanz von 50 Astronomischen Einheiten zur Sonne. Als erst fünftes Objekt der Geschichte tritt sie in diesen fernen Bereich ein und fotografiert die Position der Voyager 1.",
      image: img10
    }
  ];

  const texts = [
    {
      isSolarSystem: true,
      x: 0, y: 0
    },
    {
      isPlutoCharonSystem: true,
      x: 0, y: -500
    },
    {
      isPlanetShowcase: true,
      modelPath: '/3Dmodelle/pluto.glb',
      headline: "Pluto",
      text: "Der König des Kuipergürtels. Einst der neunte Planet, heute der bekannteste Zwergplanet. Seine rötliche Färbung und die markante Herz-Region 'Tombaugh Regio' faszinieren Forscher weltweit.",
      scale: 2.2,
      cameraDistance: 7.5,
      x: 1000, y: 0,
      // Showcase 2 Data
      images: [plutoClose1, plutoClose2, plutoClose3],
      miniModelPath: plutoCuteModelPath,
      loupeConfig: { bottom: 559, right: 425, size: 358, mag: 1.5, handleRotation: -109 },
      textOffsetX: -1000,
      textOffsetY: 0,
      textScale: 0.9
    },
    {
      isPlanetShowcase: true,
      modelPath: '/3Dmodelle/charon.glb',
      headline: "Charon",
      text: "Der dunkle Begleiter. Fast halb so groß wie Pluto selbst. Seine dunkle Polregion 'Mordor Macula' und riesige Canyons machen ihn zu einem der spannendsten Monde im Sonnensystem.",
      scale: 1.2,
      cameraDistance: 4.0,
      x: 1500, y: 500,
      // Showcase 2 Data
      images: [charonClose1, charonClose2],
      loupeConfig: { bottom: 200, right: 200, size: 300, mag: 2, handleRotation: 45 }
    },
    {
      isFinal: true,
      isEmpty: true,
      x: 0, y: 0
    }
  ];

  // Abstand zwischen den Texten auf der Z-Achse
  const zStep = 1500; 
  
  // Geglätteter Scroll-Wert für weiche Animationen
  let smoothedScrollY = 0;
  
  onMount(() => {
    const updateScroll = () => {
      // Lerp (Linear Interpolation) für weiches Nachziehen
      // Je kleiner der Faktor (0.03), desto langsamer/träger die Bewegung
      smoothedScrollY += (scrollY - smoothedScrollY) * 0.02;
      requestAnimationFrame(updateScroll);
    };
    updateScroll();
  });
  
  // Berechne die Z-Position basierend auf dem geglätteten Scroll
  $: cameraZ = innerHeight ? smoothedScrollY * (zStep / innerHeight) : 0;

  // Auto-Start Game when reaching the end
  $: {
    if (innerHeight > 0 && texts[texts.length - 1].isFinal) {
      const lastIndex = texts.length - 1;
      const targetScroll = lastIndex * innerHeight;
      // Trigger when we are very close to the final section
      if (Math.abs(smoothedScrollY - targetScroll) < 20 && !gameStarted) {
        gameStarted = true;
      }
    }
  }

  // Berechne Kamera X und Y Position für dynamischen Pfad
  let cameraX = 0;
  let cameraY = 0;
  let sceneRotation = 0;

  // Easing Funktion für geschmeidigere Übergänge (Ease In Out Cubic)
  // Das sorgt für sanftes Anfahren und Abbremsen zwischen den Stationen
  function easeInOutCubic(x) {
    return x < 0.5 ? 4 * x * x * x : 1 - Math.pow(-2 * x + 2, 3) / 2;
  }
  
  $: {
    if (innerHeight > 0) {
      // Nutze smoothedScrollY statt scrollY für die Berechnung
      const progress = smoothedScrollY / innerHeight;
      const index = Math.floor(progress);
      const fraction = progress - index;
      
      // Easing auf den Fortschritt anwenden
      const easedFraction = easeInOutCubic(fraction);
      
      // Aktuelles und nächstes Item holen (mit Fallback)
      const currentItem = texts[Math.min(index, texts.length - 1)] || texts[texts.length - 1];
      const nextItem = texts[Math.min(index + 1, texts.length - 1)] || currentItem;
      
      const currentX = currentItem.x || 0;
      const currentY = currentItem.y || 0;
      const nextX = nextItem.x || 0;
      const nextY = nextItem.y || 0;
      
      // Interpolation mit Easing statt linear
      cameraX = currentX + (nextX - currentX) * easedFraction;
      cameraY = currentY + (nextY - currentY) * easedFraction;

      // Rotation Logic
      const plutoIndex = texts.findIndex(t => t.headline === 'Pluto');
      const charonIndex = texts.findIndex(t => t.headline === 'Charon');
      const gameIndex = texts.findIndex(t => t.isFinal);

      if (plutoIndex !== -1 && charonIndex !== -1) {
        if (progress >= plutoIndex && progress < charonIndex) {
          // Cinematic Glide: Sanftes "Banking" (25°) statt harter 180°-Drehung
          // Das wirkt wie ein eleganter Vorbeiflug
          const rotProgress = (progress - plutoIndex) / (charonIndex - plutoIndex);
          sceneRotation = easeInOutCubic(rotProgress) * 25;
        } else if (gameIndex !== -1 && progress >= charonIndex && progress < gameIndex) {
          // Sanftes Zurückdrehen zur Frontalansicht für das Spiel
          const rotProgress = (progress - charonIndex) / (gameIndex - charonIndex);
          sceneRotation = 25 - easeInOutCubic(rotProgress) * 25;
        } else {
          sceneRotation = 0;
        }
      }
    }
  }

  // Scroll-Geschwindigkeit berechnen für Warp-Effekt (SEHR subtil)
  $: {
    scrollVelocity = scrollY - lastScrollY;
    lastScrollY = scrollY;
    
    // Warp-Effekt - MINIMAL und sanft
    if (Math.abs(scrollVelocity) > 15) {
      warpSpeed = Math.min(Math.abs(scrollVelocity) / 200, 0.3); // Max 0.3 (sehr klein)
    } else {
      warpSpeed *= 0.75; // Sehr schnell abklingen
    }
  }

  // Determine if we are currently viewing a 3D section to adjust galaxy background
  let isIn3DSection = false;
  let isInPlutoCharonSystem = false;
  $: {
    if (innerHeight > 0) {
      // Auch hier smoothedScrollY nutzen, damit der Hintergrund synchron wechselt
      const currentIndex = Math.round(smoothedScrollY / innerHeight);
      if (currentIndex >= 0 && currentIndex < texts.length) {
        const item = texts[currentIndex];
        isIn3DSection = !!(item.isPlanetShowcase || item.isPlutoCharonSystem);
        isInPlutoCharonSystem = !!item.isPlutoCharonSystem;
      } else {
        isIn3DSection = false;
        isInPlutoCharonSystem = false;
      }
    }
  }

  function handleNavigation(target) {
    // Find the index of the target showcase
    const index = texts.findIndex(t => t.isPlanetShowcase && t.headline.toLowerCase() === target.toLowerCase());
    if (index !== -1) {
      // Scroll to that index
      window.scrollTo({
        top: index * innerHeight,
        behavior: 'smooth'
      });
    }
  }

  function handleCardClick(index) {
    // Wenn wir nicht schon auf der Karte sind (ungefähr), scrollen wir hin
    const targetScroll = index * innerHeight;
    // Toleranz von 100px
    if (Math.abs(scrollY - targetScroll) > 100) {
      window.scrollTo({
        top: targetScroll,
        behavior: 'smooth'
      });
    }
  }

</script>

<svelte:window bind:scrollY bind:innerHeight bind:innerWidth />

<!-- 3D Solar System als ERSTE Section -->
<!-- <div class="solar-system-section">
  <SolarSystem3D />
</div> -->

<div class="galaxy-background">
  <Galaxy 
    mouseRepulsion={false}
    mouseInteraction={false}
    density={isIn3DSection ? 0.15 : 0.3}
    baseScale={isIn3DSection ? 150.0 : 30.0}
    glowIntensity={(isIn3DSection ? 0.2 : 0.5) + warpSpeed * 0.1}
    saturation={0.0}
    hueShift={240}
    scrollOffset={scrollY}
    speed={(isIn3DSection ? 0.05 : 1.0) + warpSpeed * 0.3}
    starSpeed={isIn3DSection ? 0.0 : 0.5}
    autoCenterRepulsion={warpSpeed * 0.15}
    twinkleIntensity={isIn3DSection ? 0.0 : 0.3}
  />
</div>

{#if showSecondShowcase && activeShowcaseData}
  <div class="showcase-overlay">
    <button class="back-btn" on:click={handleBackClick}>← Zurück</button>
    <PlanetShowcase2 
      modelPath={activeShowcaseData.modelPath}
      headline={activeShowcaseData.headline}
      text={activeShowcaseData.text}
      scale={activeShowcaseData.scale}
      isActive={true}
      images={activeShowcaseData.images}
      miniModelPath={activeShowcaseData.miniModelPath}
      loupeConfig={activeShowcaseData.loupeConfig}
      textOffsetX={activeShowcaseData.textOffsetX || 0}
      textOffsetY={activeShowcaseData.textOffsetY || 0}
      textScale={activeShowcaseData.textScale || 1}
    />
  </div>
{/if}

{#if gameStarted}
  <PlutoGame onExit={() => {
    gameStarted = false;
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }} />
{:else}
  <div class="viewport">
    <div class="scene" style="transform: rotateY({sceneRotation}deg);">
      {#each texts as item, i}
        {@const itemZ = -i * zStep}
        {@const currentZ = itemZ + cameraZ}
        
        <!-- 
          Konstanten Definitionen
        -->
        {@const isEven = i % 2 === 0}
        {@const isDesktop = innerWidth >= 768}
        {@const isFinal = !!item.isFinal}
        {@const isEmpty = !!item.isEmpty}
        {@const isPlutoCharonSystem = !!item.isPlutoCharonSystem}
        {@const isPlanetShowcase = !!item.isPlanetShowcase}
        {@const isDiscoveryJourney = !!item.isDiscoveryJourney}
        {@const isSolarSystem = !!item.isSolarSystem}
        {@const isKuiperBelt = !!item.isKuiperBelt}

        <!-- 
          Berechnung der Opazität:
          - Sichtweite weiter reduziert: Nächste Karte ist dunkler (Divisor 1200)
          - Karten, die "hinter" dem Betrachter sind (positive Z), blenden schneller aus (Divisor 300)
          - PlanetShowcase-Elemente:
            - Pluto (erstes Showcase): Standard Sichtweite (1200), damit er vom System aus NICHT sichtbar ist.
            - Charon (zweites Showcase): Hohe Sichtweite (2500), damit er von Pluto aus sichtbar ist.
        -->
        {@const isCharon = item.headline === 'Charon'}
        {@const visibilityRange = isCharon ? 2500 : 1200}
        {@const backVisibilityRange = isPlanetShowcase ? 2500 : 300}
        {@const opacity = currentZ > backVisibilityRange ? 0 : (currentZ > 0 ? (1 - currentZ / backVisibilityRange) : Math.max(0, 1 - Math.abs(currentZ) / visibilityRange))}
        
        <!-- 
          Blur Effekt für Tiefe:
          Weniger Blur, damit man den Text noch lesen kann
        -->
        {@const blur = Math.max(0, Math.abs(currentZ) / 1000)}
        
        <!-- 
          X-Positionierung:
          Abwechselnd links und rechts, aber nur auf größeren Screens (ab 768px).
          Die letzte Karte (isFinal) ist immer zentriert.
        -->
        {@const xShift = (isFinal || isEmpty || isPlutoCharonSystem || isPlanetShowcase || isDiscoveryJourney || isKuiperBelt || isSolarSystem) ? 0 : (!isDesktop ? 0 : (isEven ? -25 : 25))}
        
        <!--
          Layout-Richtung:
          Links: Bild links, Text rechts (row)
          Rechts: Text links, Bild rechts (row-reverse)
          Mobile: Immer untereinander
          Final: Immer untereinander (zentriert)
        -->
        {@const direction = isFinal ? 'column' : (!isDesktop ? 'column' : (isEven ? 'row' : 'row-reverse'))}

        <!--
          Interaktivität:
          Nur Karten, die sichtbar genug sind, sollen Events empfangen.
          Das verhindert, dass unsichtbare/ausblendende Karten Klicks abfangen,
          erlaubt aber Interaktion mit der aktuellen Karte, auch wenn sie nicht exakt bei Z=0 ist.
        -->
        {@const isInteractive = opacity > 0.2}
        {@const isActive = Math.abs(currentZ) < 50}

        {@const itemX = item.x || 0}
        {@const itemY = item.y || 0}
        {@const currentX = itemX - cameraX}
        {@const currentY = itemY - cameraY}

        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <div 
          class="glass-card {isFinal ? 'final-card' : ''}"
          on:click={() => handleCardClick(i)}
          role="button"
          tabindex="0"
          style="
            transform: translate3d(calc(-50% + {xShift}vw + {currentX}px), calc(-50% + {currentY}px), {currentZ}px) rotateY({-sceneRotation}deg);
            opacity: {opacity};
            filter: blur({blur}px);
            display: {(opacity <= 0.01 || isEmpty) ? 'none' : 'flex'};
            pointer-events: {isInteractive ? 'auto' : 'none'};
            flex-direction: {direction};
            {(isPlutoCharonSystem || isPlanetShowcase || isDiscoveryJourney || isSolarSystem) ? 'background: transparent; border: none; box-shadow: none; width: 100vw; height: 100vh; max-width: none; backdrop-filter: none; -webkit-backdrop-filter: none;' : ''}
          "
        >
          {#if isFinal}
            <div class="final-content">
              <h2>{item.headline}</h2>
              <button class="game-btn" on:click={() => gameStarted = true}>{item.buttonText}</button>
            </div>
          {:else if item.isSolarSystem}
            <div class="solar-system-container" style="width: 100%; height: 100%; position: relative;">
              <SolarSystem3D />
            </div>
          {:else if item.isDiscoveryJourney}
            <div class="discovery-journey-container" style="width: 100%; height: 100%; position: relative;">
              <DiscoveryJourney 
                stations={item.stations} 
                isActive={isActive} 
                on:complete={() => {
                  window.scrollTo({ top: (i + 1) * innerHeight, behavior: 'smooth' });
                }}
              />
            </div>
          {:else if item.isPluto3D}
            <div class="pluto-3d-container">
              <Pluto3D />
            </div>
          {:else if item.isPlutoCharonSystem}
            <div id="pluto-charon-section" class="pluto-charon-container" style="width: 100%; height: 100%; position: relative;">
              <PlutoCharonSystem on:navigate={(e) => handleNavigation(e.detail)} />
            </div>
          {:else if item.isPlanetShowcase}
            <div class="planet-showcase-container" style="width: 100%; height: 100%; position: relative;">
              <PlanetShowcase 
                modelPath={item.modelPath} 
                headline={item.headline} 
                text={item.text}
                scale={item.scale}
                cameraDistance={item.cameraDistance || 5}
                isActive={isActive}
                alignment={isEven ? 'left' : 'right'}
                on:click={() => handleShowcaseClick(item)}
              />
            </div>
          {:else}
            {#if item.image}
              <div class="card-image">
                <img src={item.image} alt={item.headline} />
              </div>
            {/if}
            <div class="card-content">
              <h2>{item.headline}</h2>
              <p>{item.text}</p>
            </div>
          {/if}
        </div>
      {/each}
    </div>
  </div>

  <!-- Scroll Container für Snap-Effekt -->
  <div class="scroll-container">
    {#each texts as _, i}
      <div class="scroll-section"></div>
    {/each}
  </div>
{/if}

<style>
  /* 3D Solar System Section - Fixed on top */
  .solar-system-section {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100vh;
    z-index: 100;
  }

  .showcase-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    z-index: 2000;
    background: black;
  }

  .back-btn {
    position: absolute;
    top: 40px;
    left: 40px;
    z-index: 2001;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.2);
    color: white;
    padding: 10px 20px;
    border-radius: 20px;
    cursor: pointer;
    font-size: 1rem;
    backdrop-filter: blur(10px);
    transition: all 0.3s ease;
  }

  .back-btn:hover {
    background: rgba(255, 255, 255, 0.2);
    transform: translateX(-5px);
  }

  :global(html) {
    scroll-snap-type: y mandatory;
    scroll-behavior: smooth;
  }

  :global(body) {
    margin: 0;
    background-color: #000000;
    color: #ffffff;
    overflow-x: hidden; /* Kein horizontales Scrollen */
  }

  .galaxy-background {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    z-index: -1;
  }

  .viewport {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    z-index: 10; /* Sicherstellen, dass die Viewport-Ebene über dem Scroll-Container liegt */
    perspective: 500px; /* Stärkerer 3D Effekt (vorher 1000px) */
    overflow: hidden;
    pointer-events: none; /* Damit das Scrollen auf dem Body funktioniert */
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .scene {
    position: relative;
    width: 100%;
    height: 100%;
    transform-style: preserve-3d;
    will-change: transform;
  }

  .glass-card {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 80%;
    max-width: 1000px; /* Deutlich breiter (vorher 600px) */
    padding: 0; /* Padding entfernt, da wir nun ein Layout haben */
    border-radius: 24px;
    background: transparent;
    backdrop-filter: none;
    -webkit-backdrop-filter: none;
    border: none;
    box-shadow: none;
    
    display: flex;
    flex-direction: row; /* Bild links/rechts, Text daneben */
    overflow: hidden; /* Damit das Bild nicht übersteht */
    
    will-change: transform, opacity, filter;
    /* transform wird inline gesetzt */
    pointer-events: auto; /* Damit Maus-Events (Hover) funktionieren */
  }

  .card-image {
    flex: 1;
    min-width: 300px;
    position: relative;
  }

  .card-image img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
  }

  .card-content {
    flex: 1;
    padding: 4rem;
    display: flex;
    flex-direction: column;
    justify-content: center;
  }

  .glass-card h2 {
    font-family: 'SF Pro Display', -apple-system, BlinkMacSystemFont, sans-serif;
    font-size: 3.5rem;
    font-weight: 900;
    margin: 0 0 1.5rem 0;
    background: linear-gradient(to bottom right, #ffffff, #a5a5a5);
    background-clip: text;
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    letter-spacing: -0.02em;
    line-height: 1.1;
  }

  .glass-card p {
    font-size: 1.5rem;
    line-height: 1.6;
    color: rgba(255, 255, 255, 0.8);
    margin: 0;
    font-weight: 300;
  }

  @media (max-width: 768px) {
    .glass-card {
      flex-direction: column !important; /* Wichtig, da inline-style überschrieben werden muss */
      max-width: 600px;
    }
    
    .card-image {
      height: 250px;
      min-width: auto;
      flex: none;
    }

    .card-content {
      padding: 2rem;
    }

    .glass-card h2 {
      font-size: 2rem;
    }
    .glass-card p {
      font-size: 1.1rem;
    }
  }

  .final-content {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    text-align: center;
    width: 100%;
    padding: 4rem;
  }

  .game-btn {
    margin-top: 2rem;
    padding: 1rem 3rem;
    font-size: 1.5rem;
    font-weight: 700;
    color: #fff;
    background: transparent;
    border: 2px solid #fff;
    border-radius: 50px;
    cursor: pointer;
    transition: all 0.3s ease;
    text-transform: uppercase;
    letter-spacing: 2px;
  }

  .game-btn:hover {
    transform: scale(1.05);
    background: #fff;
    color: #000;
    box-shadow: 0 0 20px rgba(255, 255, 255, 0.5);
  }

  .scroll-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    pointer-events: none; /* Container selbst soll keine Events fangen, aber Kinder schon? Nein, hier nur Spacer */
  }

  .scroll-section {
    height: 100vh;
    width: 100%;
    scroll-snap-align: start;
    pointer-events: none; /* Events durchlassen, damit die Glass Cards erreichbar sind */
  }
</style>

<script>
  export let color = '#5227FF';
  export let size = 1;
  export let images = []; // Array of image URLs
  export let className = '';

  let open = false;
  const maxItems = 3;
  let papers = [];
  
  // Prepare papers array reactively
  $: {
    const p = [...images.slice(0, maxItems)];
    while (p.length < maxItems) {
      p.push(null);
    }
    papers = p;
  }

  let paperOffsets = Array(maxItems).fill({ x: 0, y: 0 });

  function darkenColor(hex, percent) {
    let color = hex.startsWith('#') ? hex.slice(1) : hex;
    if (color.length === 3) {
      color = color.split('').map(c => c + c).join('');
    }
    const num = parseInt(color, 16);
    let r = (num >> 16) & 0xff;
    let g = (num >> 8) & 0xff;
    let b = num & 0xff;
    r = Math.max(0, Math.min(255, Math.floor(r * (1 - percent))));
    g = Math.max(0, Math.min(255, Math.floor(g * (1 - percent))));
    b = Math.max(0, Math.min(255, Math.floor(b * (1 - percent))));
    return '#' + ((1 << 24) + (r << 16) + (g << 8) + b).toString(16).slice(1).toUpperCase();
  }

  $: folderBackColor = darkenColor(color, 0.08);
  $: paper1 = darkenColor('#ffffff', 0.1);
  $: paper2 = darkenColor('#ffffff', 0.05);
  $: paper3 = '#ffffff';

  function handleClick() {
    open = !open;
    if (open) {
      // Reset offsets when opening
      paperOffsets = Array(maxItems).fill({ x: 0, y: 0 });
    }
  }

  function handlePaperMouseMove(e, index) {
    if (!open) return;
    e.stopPropagation(); // Prevent interfering with other interactions
    const rect = e.currentTarget.getBoundingClientRect();
    const centerX = rect.left + rect.width / 2;
    const centerY = rect.top + rect.height / 2;
    const offsetX = (e.clientX - centerX) * 0.15;
    const offsetY = (e.clientY - centerY) * 0.15;
    
    // Create a new array to trigger reactivity
    const newOffsets = [...paperOffsets];
    newOffsets[index] = { x: offsetX, y: offsetY };
    paperOffsets = newOffsets;
  }

  function handlePaperMouseLeave(index) {
    const newOffsets = [...paperOffsets];
    newOffsets[index] = { x: 0, y: 0 };
    paperOffsets = newOffsets;
  }
</script>

<div class={className} style="transform: scale({size}); display: inline-block;"
  on:mouseenter={() => { open = true; paperOffsets = Array(maxItems).fill({ x: 0, y: 0 }); }}
  on:mouseleave={() => open = false}
>
  <div 
    class="folder {open ? 'open' : ''}" 
    style="
      --folder-color: {color};
      --folder-back-color: {folderBackColor};
      --paper-1: {paper1};
      --paper-2: {paper2};
      --paper-3: {paper3};
    "
    on:click|stopPropagation={handleClick}
    on:keydown={(e) => e.key === 'Enter' && handleClick()}
    role="button"
    tabindex="0"
  >
    <div class="folder__back">
      {#each papers as item, i}
        <div
          class="paper paper-{i + 1}"
          on:mousemove={(e) => handlePaperMouseMove(e, i)}
          on:mouseleave={() => handlePaperMouseLeave(i)}
          on:click|stopPropagation={() => {}} 
          style={open ? `--magnet-x: ${paperOffsets[i]?.x || 0}px; --magnet-y: ${paperOffsets[i]?.y || 0}px;` : ''}
          role="img" 
        >
          {#if item}
            <img src={item} alt="Folder item {i+1}" class="paper-content" />
          {/if}
        </div>
      {/each}
      <div class="folder__front"></div>
      <div class="folder__front right"></div>
    </div>
  </div>
</div>

<style>
  .paper-content {
    width: 100%;
    height: 100%;
    object-fit: cover;
    border-radius: 8px;
    pointer-events: none; /* Prevent image dragging interfering */
  }

  :root {
    --folder-color: #70a1ff;
    --folder-back-color: #4785ff;
    --paper-1: #e6e6e6;
    --paper-2: #f2f2f2;
    --paper-3: #ffffff;
  }

  .folder {
    transition: all 0.2s ease-in;
    cursor: pointer;
    position: relative;
  }

  .folder:not(.folder--click):hover {
    transform: translateY(-8px);
  }

  .folder:not(.folder--click):hover .paper {
    transform: translate(-50%, 0%);
  }

  .folder:not(.folder--click):hover .folder__front {
    transform: skew(15deg) scaleY(0.6);
  }

  .folder:not(.folder--click):hover .right {
    transform: skew(-15deg) scaleY(0.6);
  }

  .folder.open {
    transform: translateY(-8px);
  }

  /* Paper 1 */
  .folder.open .paper:nth-child(1) {
    transform: translate(calc(-120% + var(--magnet-x, 0px)), calc(-70% + var(--magnet-y, 0px))) rotateZ(-15deg);
  }

  .folder.open .paper:nth-child(1):hover {
    transform: translate(calc(-120% + var(--magnet-x, 0px)), calc(-70% + var(--magnet-y, 0px))) rotateZ(-15deg) scale(1.1);
  }

  /* Paper 2 */
  .folder.open .paper:nth-child(2) {
    transform: translate(calc(10% + var(--magnet-x, 0px)), calc(-70% + var(--magnet-y, 0px))) rotateZ(15deg);
    height: 80%;
  }

  .folder.open .paper:nth-child(2):hover {
    transform: translate(calc(10% + var(--magnet-x, 0px)), calc(-70% + var(--magnet-y, 0px))) rotateZ(15deg) scale(1.1);
  }

  /* Paper 3 */
  .folder.open .paper:nth-child(3) {
    transform: translate(calc(-50% + var(--magnet-x, 0px)), calc(-100% + var(--magnet-y, 0px))) rotateZ(5deg);
    height: 80%;
  }

  .folder.open .paper:nth-child(3):hover {
    transform: translate(calc(-50% + var(--magnet-x, 0px)), calc(-100% + var(--magnet-y, 0px))) rotateZ(5deg) scale(1.1);
  }

  .folder.open .folder__front {
    transform: skew(15deg) scaleY(0.6);
  }

  .folder.open .right {
    transform: skew(-15deg) scaleY(0.6);
  }

  .folder__back {
    position: relative;
    width: 100px;
    height: 80px;
    background: var(--folder-back-color);
    border-radius: 0px 10px 10px 10px;
  }

  .folder__back::after {
    position: absolute;
    z-index: 0;
    bottom: 98%;
    left: 0;
    content: '';
    width: 30px;
    height: 10px;
    background: var(--folder-back-color);
    border-radius: 5px 5px 0 0;
  }

  .paper {
    position: absolute;
    z-index: 2;
    bottom: 10%;
    left: 50%;
    transform: translate(-50%, 10%);
    width: 70%;
    height: 80%;
    background: var(--paper-1);
    border-radius: 10px;
    transition: all 0.3s ease-in-out;
  }

  .paper:nth-child(2) {
    background: var(--paper-2);
    width: 80%;
    height: 70%;
  }

  .paper:nth-child(3) {
    background: var(--paper-3);
    width: 90%;
    height: 60%;
  }

  .folder__front {
    position: absolute;
    z-index: 3;
    width: 100%;
    height: 100%;
    background: var(--folder-color);
    border-radius: 5px 10px 10px 10px;
    transform-origin: bottom;
    transition: all 0.3s ease-in-out;
  }
  
  .folder__front.right {
    z-index: 3;
    width: 100%;
    height: 100%;
    background: rgba(0,0,0,0.1); /* Slight shadow for the right part? Or just same color? */
    /* The React CSS just says .right, but doesn't define color. 
       It inherits background from .folder__front unless overridden.
       Wait, .folder__front.right is a separate div.
       In React: <div className="folder__front right"></div>
       It has .folder__front class, so it gets background var(--folder-color).
    */
    transform-origin: bottom;
  }
</style>
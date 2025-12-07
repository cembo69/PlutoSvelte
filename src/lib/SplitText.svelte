<script>
  import { onMount } from 'svelte';
  import { gsap } from 'gsap';

  export let text = "";
  export let className = "";
  export let delay = 100;
  export let duration = 0.6;
  
  let container;

  onMount(() => {
    if (!container) return;
    const spans = container.querySelectorAll('.char');
    
    // Reset state first
    gsap.set(spans, { opacity: 0, y: 40 });

    gsap.to(spans, { 
        opacity: 1, 
        y: 0, 
        duration: duration, 
        stagger: 0.03, 
        ease: "power3.out",
        delay: delay / 1000
    });
  });
</script>

<div class={className} bind:this={container} aria-label={text}>
  {#each text.split('') as char}
    <span class="char" style="display: inline-block; white-space: pre;">{char}</span>
  {/each}
</div>

<style>
  .char {
    will-change: transform, opacity;
  }
</style>
<script setup lang="ts">
import { onMounted } from 'vue';

// Animación para la cabeza de perforación
let animationFrame: number | null = null;
let t = 0;

onMounted(() => {
  // Iniciar animación para el bit y la cabeza
  animateSchematic();
});

const animateSchematic = () => {
  t += 0.01;
  
  // Animar la cabeza de perforación
  const head = document.querySelector('.head') as HTMLElement;
  if (head) {
    head.style.transform = `rotate(${t * 60}deg)`;
  }
  
  // Animar burbujas de aire
  const bubble = document.getElementById('bubble');
  if (bubble) {
    const path = document.getElementById('airflow');
    if (path) {
      const len = path.getTotalLength();
      const point = path.getPointAtLength((t % 1) * len);
      bubble.setAttribute('cx', point.x.toString());
      bubble.setAttribute('cy', point.y.toString());
    }
  }
  
  animationFrame = requestAnimationFrame(animateSchematic);
};
</script>

<template>
  <div class="card">
    <h3>Vistas esquemáticas</h3>
    <div class="grid-views">
      <div class="viewMini">
        <h4>Perforación</h4>
        <svg class="schem" viewBox="0 0 300 170">
          <defs>
            <pattern id="rock" width="18" height="18" patternUnits="userSpaceOnUse">
              <circle cx="4" cy="6" r="1" fill="#2c3e66" />
              <circle cx="10" cy="12" r="1" fill="#2c3e66" />
              <circle cx="14" cy="4" r="1" fill="#2c3e66" />
            </pattern>
          </defs>
          <rect x="140" y="20" width="10" height="80" class="node" />
          <g id="bit">
            <circle cx="145" cy="100" r="12" fill="#80b3ff22" stroke="#7fb0ff" />
            <circle cx="145" cy="100" r="3.5" fill="#7fb0ff" />
          </g>
          <rect x="180" y="20" width="110" height="120" fill="url(#rock)" stroke="#2c3e66" />
          <path id="cuttings" d="M145 100 Q 210 95 280 100" stroke="#60a5fa" fill="none" stroke-width="2"
            stroke-dasharray="4 6" />
        </svg>
      </div>
      <div class="viewMini">
        <h4>Perforadora completa</h4>
        <svg class="schem" viewBox="0 0 300 170">
          <rect x="50" y="110" width="160" height="10" rx="6" class="node" />
          <rect x="70" y="98" width="40" height="10" rx="5" class="node" />
          <circle cx="95" cy="125" r="9" class="node" />
          <circle cx="170" cy="125" r="9" class="node" />
          <rect x="220" y="40" width="10" height="70" class="node" />
          <g class="head">
            <circle cx="225" cy="65" r="16" stroke="#7fb0ff" fill="none" stroke-width="2" />
            <circle cx="225" cy="65" r="7" fill="#7fb0ff22" />
          </g>
          <g transform="translate(26,6)">
            <rect x="0" y="30" width="24" height="56" rx="6" class="node" />
            <rect x="4" y="34" width="16" height="46" class="fill" opacity=".4" />
          </g>
        </svg>
      </div>
      <div class="viewMini">
        <h4>Compresor / Aire</h4>
        <svg class="schem" viewBox="0 0 300 170">
          <rect x="20" y="76" width="80" height="36" rx="8" class="node" />
          <circle cx="140" cy="95" r="22" class="node" />
          <path d="M100 95 L 120 95" stroke="#7fb0ff" />
          <path d="M166 95 L 220 95" stroke="#7fb0ff" />
          <rect x="220" y="76" width="8" height="36" class="node" />
          <path id="airflow" d="M30 95 C 90 70, 160 70, 220 87" stroke="#38bdf8" fill="none" stroke-width="2"
            stroke-dasharray="5 7" />
          <circle id="bubble" r="3.5" fill="#93c5fd" />
        </svg>
      </div>
    </div>
  </div>
</template>

<style scoped>
.grid-views {
  display: grid;
  grid-template-columns: 1fr;
  gap: 0.6rem;
}

@media (min-width: 780px) {
  .grid-views {
    grid-template-columns: 1fr 1fr 1fr;
  }
}

.viewMini {
  background: #0e1a3a;
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 0.4rem;
}

.viewMini h4 {
  margin: 0.2rem 0 0.5rem 0.2rem;
  font-size: 0.85rem;
  color: #9fb2d9;
}

.schem {
  width: 100%;
  height: 170px;
}

@media (min-width: 1100px) {
  .schem {
    height: 200px;
  }
}

.schem .node {
  fill: #1d2b4d;
  stroke: #4a67a8;
}

.schem .fill {
  fill: #60a5fa;
}

.schem text {
  fill: #cfe0ff;
  font-size: 0.75rem;
}

.head {
  transform-origin: 225px 65px;
}

@media (max-width: 480px) {
  .head {
    animation-duration: 9s;
  }
}
</style>
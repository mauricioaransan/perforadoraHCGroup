<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { clamp } from '../utils/helpers';

const props = defineProps<{
  state: {
    shift: {
      meters: number;
    };
    snr: number;
  };
}>(); 

// Referencias para elementos del mapa
const vp = ref<SVGGElement | null>(null);
const mmsvg = ref<SVGSVGElement | null>(null);
const mk = ref<SVGCircleElement | null>(null);
const ring = ref<SVGCircleElement | null>(null);
const crumb = ref<SVGPolylineElement | null>(null);
const mmc = ref<HTMLDivElement | null>(null);
const mmdist = ref<HTMLDivElement | null>(null);
const scaleBar = ref<HTMLDivElement | null>(null);
const scaleTxt = ref<HTMLSpanElement | null>(null);
const snrHeat = ref<SVGCircleElement | null>(null);

// Estado del mapa
const mm = ref({
  scale: 1,
  tx: 0,
  ty: 0,
  dragging: false,
  lx: 0,
  ly: 0
});

// Constantes
const MPP = 2; // Metros por píxel
let t = 0;
let lastX: number | null = null;
let lastY: number | null = null;
let trail: string[] = [];
let animationFrame: number | null = null;

onMounted(() => {
  // Inicializar eventos del mapa
  initMapEvents();
  
  // Actualizar escala
  updateScale();
  
  // Iniciar animación
  animateMap();
});

const initMapEvents = () => {
  if (!mmsvg.value) return;
  
  // Zoom in
  document.getElementById('zin')?.addEventListener('click', () => {
    mm.value.scale = Math.min(2.3, mm.value.scale * 1.2);
    applyMM();
    updateScale();
  });
  
  // Zoom out
  document.getElementById('zout')?.addEventListener('click', () => {
    mm.value.scale = Math.max(0.8, mm.value.scale / 1.2);
    applyMM();
    updateScale();
  });
  
  // Reset zoom
  document.getElementById('zreset')?.addEventListener('click', () => {
    mm.value.scale = 1;
    mm.value.tx = 0;
    mm.value.ty = 0;
    applyMM();
    updateScale();
  });
  
  // Drag events
  mmsvg.value.addEventListener('mousedown', (e) => {
    mm.value.dragging = true;
    mm.value.lx = e.clientX;
    mm.value.ly = e.clientY;
  });
  
  window.addEventListener('mouseup', () => {
    mm.value.dragging = false;
  });
  
  window.addEventListener('mousemove', (e) => {
    if (!mm.value.dragging) return;
    mm.value.tx += e.clientX - mm.value.lx;
    mm.value.ty += e.clientY - mm.value.ly;
    mm.value.lx = e.clientX;
    mm.value.ly = e.clientY;
    applyMM();
  });
  
  // Wheel zoom
  mmsvg.value.addEventListener('wheel', (e) => {
    e.preventDefault();
    const d = e.deltaY < 0 ? 1.1 : 0.9;
    mm.value.scale = clamp(mm.value.scale * d, 0.8, 2.3);
    applyMM();
    updateScale();
  }, { passive: false });
};

const applyMM = () => {
  if (!vp.value) return;
  vp.value.setAttribute('transform', `translate(${mm.value.tx} ${mm.value.ty}) scale(${mm.value.scale})`);
};

const updateScale = () => {
  if (!scaleBar.value || !scaleTxt.value) return;
  const meters = 200;
  const px = meters / MPP * mm.value.scale;
  scaleBar.value.style.width = px + 'px';
  scaleTxt.value.textContent = meters + ' m';
};

const animateMap = () => {
  t += 0.01;
  
  // Mover el marcador a lo largo de una ruta
  const x = 300 + 120 * Math.sin(t / 4) + 40 * Math.sin(t / 2.2);
  const y = 180 - 60 * Math.cos(t / 4.6) + 10 * Math.sin(t / 3.1);
  
  if (mk.value && ring.value) {
    mk.value.setAttribute('cx', x.toString());
    mk.value.setAttribute('cy', y.toString());
    ring.value.setAttribute('cx', x.toString());
    ring.value.setAttribute('cy', y.toString());
  }
  
  // Calcular distancia recorrida
  if (lastX !== null && lastY !== null) {
    const dx = x - lastX;
    const dy = y - lastY;
    const d = Math.hypot(dx, dy) * MPP;
    // La distancia se actualiza en el componente principal
  }
  lastX = x;
  lastY = y;
  
  // Actualizar trail (rastro)
  if (crumb.value) {
    trail.push(`${x},${y}`);
    if (trail.length > 220) trail.shift();
    crumb.value.setAttribute('points', trail.join(' '));
  }
  
  // Actualizar coordenadas UTM
  if (mmc.value) {
    const E = Math.round(624000 + x * MPP);
    const N = Math.round(3378000 + y * MPP);
    mmc.value.textContent = `UTM E ${E} · N ${N}`;
  }
  
  // Actualizar distancia recorrida
  if (mmdist.value) {
    mmdist.value.textContent = `Recorrido turno: ${(props.state.shift.meters / 1000).toFixed(2)} km`;
  }
  
  // Actualizar radio de cobertura SNR
  if (snrHeat.value) {
    const dx2 = x - 430;
    const dy2 = y - 120;
    const dist = Math.hypot(dx2, dy2);
    const r = clamp(160 - dist * 0.5, 40, 160);
    snrHeat.value.setAttribute('r', r.toString());
  }
  
  animationFrame = requestAnimationFrame(animateMap);
};
</script>

<template>
  <div class="card">
    <h3>Mini mapa de recorrido (Vista planta)</h3>
    <div class="minimap">
      <div class="mm-tools">
        <button id="zin">+</button>
        <button id="zout">−</button>
        <button id="zreset">Reset</button>
      </div>
      <div class="compass">
        <div class="arr"></div>
        <div>N</div>
      </div>
      <div ref="mmc" class="mm-coords" id="mmc">UTM E 0 · N 0</div>
      <div class="mm-legend">Pit Norte · Sector Oeste · RB06</div>
      <div class="scale">
        <div ref="scaleBar" class="bar" id="scaleBar"></div>
        <span ref="scaleTxt" id="scaleTxt">200 m</span>
      </div>
      <div ref="mmdist" class="mm-dist" id="mmdist">Recorrido turno: 0.00 km</div>
      <svg ref="mmsvg" id="mmsvg" viewBox="0 0 960 300">
        <defs>
          <pattern id="mmgrid" width="20" height="20" patternUnits="userSpaceOnUse">
            <path d="M 20 0 L 0 0 0 20" class="grid" />
          </pattern>
          <radialGradient id="snrgrad" cx="0" cy="0" r="1">
            <stop offset="0%" stop-color="#22c55e88" />
            <stop offset="60%" stop-color="#facc1588" />
            <stop offset="100%" stop-color="#ef444488" />
          </radialGradient>
        </defs>
        <rect width="960" height="300" fill="url(#mmgrid)" />
        <g ref="vp" id="vp">
          <ellipse cx="480" cy="160" rx="260" ry="120" class="pit" />
          <ellipse cx="480" cy="160" rx="200" ry="90" class="bench" />
          <ellipse cx="480" cy="160" rx="140" ry="65" class="bench" />
          <ellipse cx="480" cy="160" rx="90" ry="42" class="bench" />
          <path d="M80,250 C220,210 340,200 420,190 C560,180 660,160 880,140" class="road" />
          <polygon points="820,120 900,120 920,180 840,190" fill="#ef444433" stroke="#ef4444aa"
            stroke-dasharray="6 4" />
          <polygon points="120,220 200,210 240,250 150,260" fill="#22c55e33" stroke="#22c55e99"
            stroke-dasharray="6 4" />
          <g>
            <circle cx="430" cy="120" r="3" fill="#93c5fd" />
            <text x="435" y="124" fill="#cfe0ff" font-size="10">RB06</text>
            <circle cx="430" cy="120" r="60" class="cover" />
            <circle cx="430" cy="120" r="110" class="cover" opacity=".5" />
          </g>
          <g>
            <circle cx="260" cy="170" r="3" fill="#93c5fd" />
          </g>
          <g>
            <circle cx="700" cy="110" r="3" fill="#93c5fd" />
          </g>
          <circle ref="snrHeat" id="snrHeat" cx="430" cy="120" r="120" fill="url(#snrgrad)" opacity=".35" />
          <polyline ref="crumb" id="crumb" fill="none" stroke="#98b9ff" stroke-width="2" stroke-opacity=".8" />
          <circle ref="ring" id="ring" class="ring" cx="300" cy="180" r="18" />
          <circle ref="mk" id="mk" class="marker" cx="300" cy="180" r="5" />
        </g>
      </svg>
    </div>
  </div>
</template>

<style scoped>
/* Mini map */
.minimap {
  width: 100%;
  height: 260px;
  border: 1px solid var(--border);
  border-radius: 12px;
  background: #0e1733;
  position: relative;
  overflow: hidden;
}

@media (min-width: 900px) {
  .minimap {
    height: 300px;
  }
}

.minimap svg {
  width: 100%;
  height: 100%;
}

.grid {
  fill: none;
  stroke: rgba(255, 255, 255, 0.05);
  stroke-width: 0.7;
}

.pit {
  fill: rgba(52, 94, 170, 0.35);
  stroke: #7fb0ff;
  stroke-width: 1.1;
}

.bench {
  fill: none;
  stroke: #3c5d9a;
  stroke-width: 1;
  stroke-dasharray: 6 4;
}

.road {
  fill: none;
  stroke: #bcd3ff80;
  stroke-width: 1.2;
  stroke-dasharray: 4 3;
}

.cover {
  fill: none;
  stroke: #3ee9a4aa;
  stroke-width: 1.2;
  stroke-dasharray: 2 4;
}

.marker {
  fill: #22c55e;
}

.ring {
  fill: #22c55e33;
}

.mm-legend,
.mm-coords,
.mm-dist {
  position: absolute;
  font-size: 0.75rem;
  color: #dce9ff;
  background: #00000033;
  padding: 0.2rem 0.35rem;
  border-radius: 0.5rem;
  border: 1px solid var(--border);
}

.mm-legend {
  bottom: 0.35rem;
  left: 0.35rem;
}

.mm-coords {
  top: 0.35rem;
  right: 0.35rem;
}

.mm-dist {
  bottom: 0.35rem;
  right: 0.35rem;
}

.mm-tools {
  position: absolute;
  top: 0.35rem;
  left: 0.35rem;
  display: flex;
  gap: 0.35rem;
}

.mm-tools button {
  background: #122047;
  color: #cfe0ff;
  border: 1px solid var(--border);
  padding: 0.2rem 0.45rem;
  border-radius: 0.5rem;
  cursor: pointer;
  font-size: 0.9rem;
}

.compass {
  position: absolute;
  top: 0.35rem;
  left: 3.2rem;
  color: #dce9ff;
  font-size: 0.75rem;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.compass .arr {
  width: 0;
  height: 0;
  border-left: 6px solid transparent;
  border-right: 6px solid transparent;
  border-bottom: 10px solid #93c5fd;
  margin-bottom: 2px;
  filter: drop-shadow(0 0 6px #60a5fa);
}

.scale {
  position: absolute;
  bottom: 0.35rem;
  left: 7rem;
  color: #dce9ff;
  font-size: 0.75rem;
  display: flex;
  align-items: center;
  gap: 0.35rem;
}

.scale .bar {
  width: 120px;
  height: 6px;
  background: linear-gradient(90deg, #ffffff, #ffffff00);
  border-radius: 4px;
  border: 1px solid var(--border);
}
</style>
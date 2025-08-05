<script setup lang="ts">
import { ref, onMounted, watch } from 'vue';
import { clamp } from '../utils/helpers';

const props = defineProps<{
  state: {
    rtt: number;
    jit: number;
    loss: number;
    sla: number;
  };
}>();

// Referencias para los elementos SVG
const rttBar = ref<HTMLElement | null>(null);
const jitBar = ref<HTMLElement | null>(null);
const lossBar = ref<HTMLElement | null>(null);
const slaBar = ref<HTMLElement | null>(null);

// Referencias para los paquetes
const p1 = ref<SVGCircleElement | null>(null);
const p2 = ref<SVGCircleElement | null>(null);
const p3 = ref<SVGCircleElement | null>(null);

// Referencias para las rutas
const l1 = ref<SVGPathElement | null>(null);
const l2 = ref<SVGPathElement | null>(null);

let t = 0;
let animationFrame: number | null = null;

onMounted(() => {
  // Inicializar los semi-gauges
  drawSemiGauge('sg-rtt', props.state.rtt, 0, 320);
  drawSemiGauge('sg-jit', props.state.jit, 0, 80);
  drawSemiGauge('sg-loss', props.state.loss, 0, 5);
  drawSemiGauge('sg-sla', props.state.sla, 0, 100);
  
  // Actualizar barras
  updateBars();
  
  // Iniciar animación de paquetes
  animatePackets();
});

// Observar cambios en el estado
watch(() => props.state, () => {
  updateBars();
}, { deep: true });

const updateBars = () => {
  setBar(rttBar.value, clamp(1 - props.state.rtt / 320, 0, 1) * 100);
  setBar(jitBar.value, clamp(1 - props.state.jit / 80, 0, 1) * 100);
  setBar(lossBar.value, clamp(1 - props.state.loss / 5, 0, 1) * 100);
  setBar(slaBar.value, props.state.sla);
  
  // Actualizar semi-gauges
  drawSemiGauge('sg-rtt', props.state.rtt, 0, 320);
  drawSemiGauge('sg-jit', props.state.jit, 0, 80);
  drawSemiGauge('sg-loss', props.state.loss, 0, 5);
  drawSemiGauge('sg-sla', props.state.sla, 0, 100);
};

const setBar = (el: HTMLElement | null, pct: number) => {
  if (!el) return;
  el.style.width = Math.round(clamp(pct, 0, 100)) + '%';
  el.style.background = pct > 70 ? 'linear-gradient(90deg,#34d399,#22c55e)' : 'linear-gradient(90deg,#facc15,#f59e0b)';
};

const drawSemiGauge = (svgId: string, value: number, min: number, max: number) => {
  const svg = document.getElementById(svgId);
  if (!svg) return;
  svg.innerHTML = '';
  
  const box = svg.getBoundingClientRect();
  const w = box.width, h = box.height;
  const cx = w / 2, cy = h - 6, r = Math.min(w, h * 1.2) / 2;
  
  const arc = (a0: number, a1: number, clr: string, wid: number) => {
    const x0 = cx + r * Math.cos(a0), y0 = cy + r * Math.sin(a0);
    const x1 = cx + r * Math.cos(a1), y1 = cy + r * Math.sin(a1);
    const p = document.createElementNS('http://www.w3.org/2000/svg', 'path');
    p.setAttribute('d', `M ${x0} ${y0} A ${r} ${r} 0 0 0 ${x1} ${y1}`);
    p.setAttribute('fill', 'none');
    p.setAttribute('stroke', clr);
    p.setAttribute('stroke-width', wid.toString());
    p.setAttribute('stroke-linecap', 'round');
    svg.appendChild(p);
    return p;
  };
  
  const s = Math.PI, e = 2 * Math.PI;
  [['#22c55e', 0, .25], ['#facc15', .25, .55], ['#f59e0b', .55, .8], ['#ef4444', .8, 1]].forEach(([c, a, b]) => {
    arc(s + (e - s) * (a as number), s + (e - s) * (b as number), c as string, Math.max(6, w * .08));
  });
  
  const t = clamp((value - min) / (max - min), 0, 1), ang = s + (e - s) * t;
  const nd = document.createElementNS('http://www.w3.org/2000/svg', 'line');
  nd.setAttribute('x1', cx.toString());
  nd.setAttribute('y1', cy.toString());
  nd.setAttribute('x2', (cx + (r - 8) * Math.cos(ang)).toString());
  nd.setAttribute('y2', (cy + (r - 8) * Math.sin(ang)).toString());
  nd.setAttribute('stroke', '#cfe4ff');
  nd.setAttribute('stroke-width', Math.max(2, w * .03).toString());
  nd.setAttribute('stroke-linecap', 'round');
  svg.appendChild(nd);
  
  const g = document.createElementNS('http://www.w3.org/2000/svg', 'path');
  const wGlow = (e - s) * .08;
  const mk = (a0: number, a1: number) => {
    const x0 = cx + r * Math.cos(a0), y0 = cy + r * Math.sin(a0);
    const x1 = cx + r * Math.cos(a1), y1 = cy + r * Math.sin(a1);
    return `M ${x0} ${y0} A ${r} ${r} 0 0 0 ${x1} ${y1}`;
  };
  g.setAttribute('d', mk(ang - wGlow, ang + wGlow));
  g.setAttribute('stroke', t < .25 ? '#22c55e' : t < .55 ? '#facc15' : t < .8 ? '#f59e0b' : '#ef4444');
  g.setAttribute('stroke-width', Math.max(8, w * .12).toString());
  g.setAttribute('stroke-linecap', 'round');
  g.setAttribute('fill', 'none');
  g.setAttribute('opacity', '.75');
  g.setAttribute('class', 'glowPulse');
  svg.appendChild(g);
};

const moveAlongPath = (el: SVGCircleElement | null, path: SVGPathElement | null, t: number) => {
  if (!el || !path) return;
  const len = path.getTotalLength();
  const p = path.getPointAtLength((t % 1) * len);
  el.setAttribute('cx', p.x.toString());
  el.setAttribute('cy', p.y.toString());
};

const animatePackets = () => {
  t += Math.max(0.006, 0.1 - props.state.rtt / 3200);
  
  moveAlongPath(p1.value, l1.value, t * 0.9);
  moveAlongPath(p2.value, l2.value, t * 1.0);
  moveAlongPath(p3.value, l2.value, t * 1.1 + 0.3);
  
  animationFrame = requestAnimationFrame(animatePackets);
};
</script>

<template>
  <div class="card">
    <h3>Ruta de Comunicación (Perforadora → Torre RB06 → Core)</h3>
    <svg class="path" viewBox="0 0 960 210">
      <rect x="100" y="120" width="150" height="60" rx="8" class="node2" />
      <text x="175" y="195" class="legend" text-anchor="middle">Perforadora</text>
      <g transform="translate(430,30)">
        <rect x="0" y="60" width="20" height="120" class="node2" />
        <path d="M10 60 L30 30 M10 60 L-10 30" stroke="#4a67a8" fill="none" />
        <circle cx="10" cy="26" r="6" fill="#4a67a8" />
        <text x="10" y="195" class="legend" text-anchor="middle">Torre RB06</text>
      </g>
      <g transform="translate(680,60)">
        <ellipse cx="60" cy="40" rx="45" ry="28" class="node2" />
        <ellipse cx="35" cy="35" rx="22" ry="14" class="node2" />
        <ellipse cx="88" cy="33" rx="22" ry="14" class="node2" />
        <text x="60" y="95" class="legend" text-anchor="middle">Core</text>
      </g>
      <path ref="l1" id="l1" class="wave" d="M250 150 C 300 120, 340 120, 420 130" />
      <path ref="l2" id="l2" class="wave" d="M450 130 C 540 100, 600 90, 760 90" />
      <circle ref="p1" id="p1" r="4" class="packet" />
      <circle ref="p2" id="p2" r="4" class="packet" />
      <circle ref="p3" id="p3" r="4" class="packet" />
    </svg>
    <div class="comm-tiles">
      <div class="tile">
        <svg class="sg" id="sg-rtt"></svg>
        <div>
          <div class="t-title">RTT</div>
          <div class="t-val">{{ Math.round(state.rtt) }} <span class="unit">ms</span></div>
          <div class="bar"><i ref="rttBar"></i></div>
        </div>
      </div>
      <div class="tile">
        <svg class="sg" id="sg-jit"></svg>
        <div>
          <div class="t-title">Jitter</div>
          <div class="t-val">{{ state.jit.toFixed(1) }} <span class="unit">ms</span></div>
          <div class="bar"><i ref="jitBar"></i></div>
        </div>
      </div>
      <div class="tile">
        <svg class="sg" id="sg-loss"></svg>
        <div>
          <div class="t-title">Pérdida</div>
          <div class="t-val">{{ state.loss.toFixed(1) }} <span class="unit">%</span></div>
          <div class="bar"><i ref="lossBar"></i></div>
        </div>
      </div>
      <div class="tile">
        <svg class="sg" id="sg-sla"></svg>
        <div>
          <div class="t-title">SLA</div>
          <div class="t-val">{{ state.sla.toFixed(1) }} <span class="unit">%</span></div>
          <div class="bar"><i ref="slaBar"></i></div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Communication path */
.path {
  width: 100%;
  height: 210px;
}

.wave {
  stroke: #60a5fa;
  stroke-width: 2;
  fill: none;
  stroke-dasharray: 5 7;
  animation: dash 2.2s linear infinite;
}

@keyframes dash {
  to {
    stroke-dashoffset: -260;
  }
}

.packet {
  fill: #7dd3fc;
  filter: drop-shadow(0 0 8px rgba(96, 165, 250, 0.9));
}

.node2 {
  fill: #1d2b4d;
  stroke: #4a67a8;
}

.legend {
  font-size: 0.8rem;
  fill: #cfe0ff;
}

.comm-tiles {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.6rem;
  margin-top: 0.4rem;
}

@media (min-width: 900px) {
  .comm-tiles {
    grid-template-columns: repeat(4, 1fr);
  }
}

.tile {
  background: linear-gradient(180deg, #0d1a39, #0a142e);
  border: 1px solid var(--border);
  border-radius: 14px;
  padding: 0.5rem;
  display: grid;
  grid-template-columns: 84px 1fr;
  gap: 0.5rem;
  align-items: center;
}

.sg {
  width: 84px;
  height: 64px;
}

.t-title {
  font-size: 0.8rem;
  color: #9fb2d9;
}

.t-val {
  font-weight: 800;
  font-size: 1.4rem;
}

.unit {
  font-size: 0.9rem;
  opacity: 0.85;
}

.bar {
  height: 10px;
  border-radius: 999px;
  background: #09122a;
  border: 1px solid var(--border);
  overflow: hidden;
}

.bar > i {
  display: block;
  height: 100%;
  width: 0;
  background: linear-gradient(90deg, #34d399, #22c55e);
}

/* Neon glow */
.glowPulse {
  animation: glow 1.4s ease-in-out infinite;
}

@keyframes glow {
  0% {
    opacity: 0.55;
    filter: drop-shadow(0 0 6px #fff);
  }
  50% {
    opacity: 1;
    filter: drop-shadow(0 0 14px #fff);
  }
  100% {
    opacity: 0.55;
    filter: drop-shadow(0 0 6px #fff);
  }
}
</style>
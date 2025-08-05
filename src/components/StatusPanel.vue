<script setup lang="ts">
import { ref, onMounted, watch } from 'vue';
import { clamp } from '../utils/helpers';

const props = defineProps<{
  state: {
    rpm: number;
    adv: number;
    tmp: number;
    prs: number;
    fuel: number;
    snr: number;
    engineOn: boolean;
    shift: {
      start: number;
      startFuelPct: number;
      meters: number;
      drilled: number;
      onCount: number;
      uptime: number;
    };
  };
}>();

// Referencias para los elementos SVG
const rpmBar = ref<HTMLElement | null>(null);
const advBar = ref<HTMLElement | null>(null);
const tmpBar = ref<HTMLElement | null>(null);
const prsBar = ref<HTMLElement | null>(null);
const fuelBar = ref<HTMLElement | null>(null);
const snrBar = ref<HTMLElement | null>(null);

// Referencias para los valores
const engineState = ref('APAGADO');
const engineStateColor = ref('#ef4444');
const uptime = ref('00:00:00');
const drilled = ref('0.0');
const km = ref('0.00');
const startCount = ref('0');

onMounted(() => {
  // Inicializar los semi-gauges
  drawSemiGauge('sg-rpm', props.state.rpm, 0, 120);
  drawSemiGauge('sg-adv', props.state.adv, 0, 3);
  drawSemiGauge('sg-tmp', props.state.tmp, 50, 110);
  drawSemiGauge('sg-prs', props.state.prs, 120, 250);
  drawSemiGauge('sg-fuel', props.state.fuel, 0, 100);
  drawSemiGauge('sg-snr', props.state.snr, 0, 40);
  
  // Actualizar barras
  updateBars();
});

// Observar cambios en el estado
watch(() => props.state, () => {
  updateBars();
  updateEngineState();
  updateUptimeDisplay();
}, { deep: true });

const updateBars = () => {
  setBar(rpmBar.value, props.state.rpm / 120 * 100);
  setBar(advBar.value, props.state.adv / 3 * 100);
  setBar(tmpBar.value, (props.state.tmp - 50) / 60 * 100);
  setBar(prsBar.value, (props.state.prs - 120) / 130 * 100);
  setBar(fuelBar.value, props.state.fuel);
  setBar(snrBar.value, props.state.snr / 40 * 100);
  
  // Actualizar semi-gauges
  drawSemiGauge('sg-rpm', props.state.rpm, 0, 120);
  drawSemiGauge('sg-adv', props.state.adv, 0, 3);
  drawSemiGauge('sg-tmp', props.state.tmp, 50, 110);
  drawSemiGauge('sg-prs', props.state.prs, 120, 250);
  drawSemiGauge('sg-fuel', props.state.fuel, 0, 100);
  drawSemiGauge('sg-snr', props.state.snr, 0, 40);
  
  // Actualizar valores de texto
  drilled.value = props.state.shift.drilled.toFixed(1);
  km.value = (props.state.shift.meters / 1000).toFixed(2);
  startCount.value = props.state.shift.onCount.toString();
};

const updateEngineState = () => {
  engineState.value = props.state.engineOn ? 'ENCENDIDO' : 'APAGADO';
  engineStateColor.value = props.state.engineOn ? '#22c55e' : '#ef4444';
};

const updateUptimeDisplay = () => {
  const uh = Math.floor(props.state.shift.uptime / 3600);
  const um = Math.floor((props.state.shift.uptime % 3600) / 60);
  const us = Math.floor(props.state.shift.uptime % 60);
  uptime.value = `${String(uh).padStart(2, '0')}:${String(um).padStart(2, '0')}:${String(us).padStart(2, '0')}`;
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
</script>

<template>
  <div class="card">
    <h3>Estado rápido</h3>
    <div class="grid-tiles">
      <div class="tile">
        <svg class="sg" id="sg-rpm"></svg>
        <div>
          <div class="t-title">Rotación</div>
          <div class="t-val">{{ Math.round(state.rpm) }} <span class="unit">RPM</span></div>
          <div class="bar"><i ref="rpmBar"></i></div>
        </div>
      </div>
      <div class="tile">
        <svg class="sg" id="sg-adv"></svg>
        <div>
          <div class="t-title">Avance/min</div>
          <div class="t-val">{{ state.adv.toFixed(2) }} <span class="unit">m/min</span></div>
          <div class="bar"><i ref="advBar"></i></div>
        </div>
      </div>
      <div class="tile">
        <svg class="sg" id="sg-tmp"></svg>
        <div>
          <div class="t-title">Temp. motor</div>
          <div class="t-val">{{ Math.round(state.tmp) }} <span class="unit">°C</span></div>
          <div class="bar"><i ref="tmpBar"></i></div>
        </div>
      </div>
      <div class="tile">
        <svg class="sg" id="sg-prs"></svg>
        <div>
          <div class="t-title">Presión hidráulica</div>
          <div class="t-val">{{ Math.round(state.prs) }} <span class="unit">bar</span></div>
          <div class="bar"><i ref="prsBar"></i></div>
        </div>
      </div>
      <div class="tile">
        <svg class="sg" id="sg-fuel"></svg>
        <div>
          <div class="t-title">Combustible</div>
          <div class="t-val">{{ Math.round(state.fuel) }} <span class="unit">%</span></div>
          <div class="bar"><i ref="fuelBar"></i></div>
        </div>
      </div>
      <div class="tile">
        <svg class="sg" id="sg-snr"></svg>
        <div>
          <div class="t-title">SNR</div>
          <div class="t-val">{{ state.snr.toFixed(1) }} <span class="unit">dB</span></div>
          <div class="bar"><i ref="snrBar"></i></div>
        </div>
      </div>
    </div>

    <div class="meta">
      <div>
        <span class="k">Estado:</span>
        <span :style="{ color: engineStateColor, fontWeight: 900 }">{{ engineState }}</span>
      </div>
      <div><span class="k">Encendidos:</span> <span>{{ startCount }}</span></div>
      <div><span class="k">Uptime:</span> <span>{{ uptime }}</span></div>
      <div><span class="k">Perforado:</span> <span>{{ drilled }}</span> m</div>
      <div><span class="k">Recorrido:</span> <span>{{ km }}</span> km</div>
    </div>
  </div>
</template>

<style scoped>
.grid-tiles {
  display: grid;
  grid-template-columns: 1fr;
  gap: 0.6rem;
}

@media (min-width: 600px) {
  .grid-tiles {
    grid-template-columns: 1fr 1fr;
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

.meta {
  display: flex;
  flex-wrap: wrap;
  gap: 0.9rem;
  align-items: center;
  margin-top: 0.4rem;
}

.meta .k {
  color: #9fb2d9;
  margin-right: 0.35rem;
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
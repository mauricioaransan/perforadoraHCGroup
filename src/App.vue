<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import Header from './components/Header.vue';
import StatusPanel from './components/StatusPanel.vue';
import SchematicViews from './components/SchematicViews.vue';
import CommunicationPath from './components/CommunicationPath.vue';
import MiniMap from './components/MiniMap.vue';
import ThroughputChart from './components/ThroughputChart.vue';

// Estado global compartido
const state = ref({
  rpm: 80,
  adv: 1.8,
  tmp: 86,
  prs: 190,
  fuel: 76,
  snr: 24,
  rtt: 160,
  jit: 12,
  loss: 0.2,
  sla: 99,
  thr: 8,
  engineOn: false,
  shift: {
    start: Date.now(),
    startFuelPct: 76,
    meters: 0,
    drilled: 0,
    onCount: 0,
    uptime: 0
  }
});

let timer: number | null = null;

onMounted(() => {
  timer = window.setInterval(tick, 1500);
  tick();
});

onUnmounted(() => {
  if (timer) {
    clearInterval(timer);
    timer = null;
  }
});

const tick = () => {
  // Simulación de datos (se implementará en cada componente)
};

const resetShift = () => {
  state.value.shift = {
    start: Date.now(),
    startFuelPct: state.value.fuel,
    meters: 0,
    drilled: 0,
    onCount: 0,
    uptime: 0
  };
};

const togglePause = () => {
  if (timer) {
    clearInterval(timer);
    timer = null;
  } else {
    timer = window.setInterval(tick, 1500);
    tick();
  }
};
</script>

<template>
  <div class="wrap">
    <Header 
      :shift="state.shift" 
      :onCount="state.shift.onCount" 
      @reset-shift="resetShift" 
      @toggle-pause="togglePause" 
    />

    <section class="main">
      <!-- Estado rápido y Vistas esquemáticas -->
      <div class="row cols-2">
        <StatusPanel :state="state" />
        <SchematicViews />
      </div>

      <!-- Ruta de Comunicación y Mini mapa -->
      <div class="row cols-2">
        <CommunicationPath :state="state" />
        <MiniMap :state="state" />
      </div>

      <!-- Rendimiento (Throughput) -->
      <div class="row">
        <ThroughputChart :state="state" />
      </div>
    </section>
  </div>
</template>

<style>
:root{
  --bg:#060a16; --panel:#0d162e; --card:#122449; --ink:#e9f1ff; --muted:#9fb2d9;
  --border:rgba(255,255,255,.10);
}
*{box-sizing:border-box}
html,body{height:100%}
body{margin:0;color:var(--ink);font:14px/1.45 ui-sans-serif,system-ui,Segoe UI,Roboto,Helvetica,Arial;background:linear-gradient(180deg,#071022,#0a1327)}
.wrap{width:100%;max-width:1920px;margin:0 auto}

.main{display:grid;gap:.6rem;padding:.6rem}
.row{display:grid;gap:.6rem}
/* default: phones = single column */
.row.cols-2{grid-template-columns:1fr}
/* tablets */
@media(min-width:780px){ .row.cols-2{grid-template-columns:1fr 1fr} }
/* desktop wide */
@media(min-width:1200px){ .row.cols-2{grid-template-columns:900px 1fr} }

.card{background:linear-gradient(180deg,#1b2f63aa,#0f1e44dd);border:1px solid var(--border);border-radius:16px;padding:.7rem;overflow:hidden}
.card h3{margin:.1rem 0 .5rem;font-size:1rem;color:#cfe0ff}
.big{min-height:260px}
</style>

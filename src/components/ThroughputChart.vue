<script setup lang="ts">
import { ref, onMounted, watch } from 'vue';

const props = defineProps<{
  state: {
    thr: number;
  };
}>();

const thrCanvas = ref<HTMLCanvasElement | null>(null);
const thrData = ref<number[]>(Array.from({ length: 100 }, () => 8));
let thrCtx: CanvasRenderingContext2D | null = null;

onMounted(() => {
  if (!thrCanvas.value) return;
  
  // Inicializar el contexto del canvas
  const canvas = thrCanvas.value;
  const r = window.devicePixelRatio || 1;
  
  const resize = () => {
    if (!canvas) return;
    canvas.width = canvas.clientWidth * r;
    canvas.height = canvas.clientHeight * r;
    thrCtx = canvas.getContext('2d');
    drawThr();
  };
  
  resize();
  window.addEventListener('resize', resize);
});

// Observar cambios en el estado
watch(() => props.state.thr, (newVal) => {
  thrData.value.push(newVal);
  thrData.value.shift();
  drawThr();
});

const drawThr = () => {
  if (!thrCtx) return;
  
  const ctx = thrCtx;
  const w = ctx.canvas.width;
  const h = ctx.canvas.height;
  
  ctx.clearRect(0, 0, w, h);
  ctx.strokeStyle = "#88b6ff";
  ctx.lineWidth = 2;
  ctx.beginPath();
  
  const min = Math.min(...thrData.value);
  const max = Math.max(...thrData.value);
  
  thrData.value.forEach((v, i) => {
    const x = i / (thrData.value.length - 1) * w;
    const y = h - (v - min) / (max - min + 1e-6) * h;
    i ? ctx.lineTo(x, y) : ctx.moveTo(x, y);
  });
  
  ctx.stroke();
};
</script>

<template>
  <div class="card">
    <h3>Rendimiento (Throughput) · Últimos 60s</h3>
    <canvas ref="thrCanvas" id="thrChart" style="width:100%;height:110px;border:1px solid var(--border);border-radius:10px;background:#0e173366"></canvas>
  </div>
</template>

<style scoped>
/* Estilos específicos para el gráfico si son necesarios */
</style>
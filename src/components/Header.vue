<script setup lang="ts">
import { ref, onMounted } from 'vue';

const props = defineProps<{
  shift: {
    start: number;
    startFuelPct: number;
    meters: number;
    drilled: number;
    onCount: number;
    uptime: number;
  };
  onCount: number;
}>();

const emit = defineEmits(['reset-shift', 'toggle-pause']);

const shiftClock = ref('Turno 00:00:00');
const isPaused = ref(false);

let clockTimer: number | null = null;

onMounted(() => {
  clockTimer = window.setInterval(updateShiftClock, 1000);
  updateShiftClock();
});

const updateShiftClock = () => {
  const t = (Date.now() - props.shift.start) / 1000;
  const h = String(Math.floor(t / 3600)).padStart(2, '0');
  const m = String(Math.floor((t % 3600) / 60)).padStart(2, '0');
  const s = String(Math.floor(t % 60)).padStart(2, '0');
  shiftClock.value = `Turno ${h}:${m}:${s}`;
};

const resetShift = () => {
  emit('reset-shift');
};

const togglePause = () => {
  isPaused.value = !isPaused.value;
  emit('toggle-pause');
};
</script>

<template>
  <header>
    <div class="brand">
      <div class="logo"></div> HC GROUP · SMARTLINK
    </div>
    <div class="controls">
      <span class="pill" id="shiftClock">{{ shiftClock }}</span>
      <span class="pill">Encendidos: <b>{{ onCount }}</b></span>
      <button class="btn" @click="resetShift">Reiniciar</button>
      <button class="btn" @click="togglePause">{{ isPaused ? '▶' : '⏸' }}</button>
    </div>
  </header>
</template>

<style scoped>
header {
  position: sticky;
  top: 0;
  z-index: 10;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding: 0.6rem 0.8rem;
  border-bottom: 1px solid var(--border);
  background: rgba(9, 15, 30, 0.92);
  backdrop-filter: blur(6px);
}

.brand {
  display: flex;
  align-items: center;
  gap: 0.6rem;
  font-weight: 800;
}

.brand .logo {
  width: 26px;
  height: 26px;
  border-radius: 7px;
  background: conic-gradient(#22c55e, #60a5fa, #93c5fd, #22c55e);
  box-shadow: 0 0 14px #60a5fa inset, 0 0 8px #60a5fa;
}

.controls {
  display: flex;
  gap: 0.5rem;
  align-items: center;
  flex-wrap: wrap;
  justify-content: flex-end;
}

.btn,
.pill {
  padding: 0.4rem 0.6rem;
  border: 1px solid var(--border);
  border-radius: 0.7rem;
  background: #0f1c3f;
  color: #dbe9ff;
  font-size: 0.85rem;
}

.btn {
  cursor: pointer;
}
</style>
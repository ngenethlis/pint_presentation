<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'

const PACKETS = [
  { name: 'p₁',  switches: [0, 0, 1, 0, 0], digest: 'S₃', known: '{ S₃ }', note: 'New switch found' },
  { name: 'p₂',  switches: [1, 0, 0, 0, 0], digest: 'S₁', known: '{ S₁, S₃ }', note: 'New switch found' },
  { name: 'p₃',  switches: [0, 0, 1, 0, 0], digest: 'S₃', known: '{ S₁, S₃ }', note: 'Duplicate!' },
  { name: 'p₄',  switches: [1, 0, 0, 0, 0], digest: 'S₁', known: '{ S₁, S₃ }', note: 'Duplicate again!' },
  { name: 'p₆',  switches: [0, 1, 0, 0, 0], digest: 'S₂', known: '{ S₁, S₂, S₃ }', note: 'New switch found (implied)' },
  { name: 'p₈',  switches: [0, 0, 0, 0, 1], digest: 'S₅', known: '{ S₁, S₂, S₃, S₅ }', note: 'New switch found' },
  { name: 'p₁₂', switches: [0, 0, 0, 1, 0], digest: 'S₄', known: '{ S₁, S₂, S₃, S₄, S₅ }', note: '✓ All switches collected' }
]

const step = ref(-1)

function handleKeydown(e: KeyboardEvent) {
  if (e.key === 'ArrowRight' || e.key === 'Enter') {
    if (step.value < PACKETS.length - 1) { advance(); e.preventDefault(); e.stopPropagation(); }
  } else if (e.key === 'ArrowLeft') {
    if (step.value > -1) { back(); e.preventDefault(); e.stopPropagation(); }
  }
}

onMounted(() => window.addEventListener('keydown', handleKeydown, { capture: true }))
onUnmounted(() => window.removeEventListener('keydown', handleKeydown, { capture: true }))

function advance() { if (step.value < PACKETS.length - 1) step.value++ }
function back()    { if (step.value > -1) step.value-- }
function reset()   { step.value = -1 }

const current = computed(() => step.value >= 0 ? PACKETS[step.value] : null)

// Accumulate which switches (0-indexed) have been seen across all packets so far
const visitedSet = computed(() => {
  const s = new Set<number>()
  for (let i = 0; i <= step.value; i++) {
    PACKETS[i].switches.forEach((on, idx) => { if (on) s.add(idx) })
  }
  return s
})
</script>

<template>
  <div class="trace">
    <div class="net">
      <div class="src-node">SRC</div>
      <div class="wire"></div>
      <template v-for="i in 5" :key="i">
        <div class="sw" :class="{
          active:  current?.switches[i-1],
          visited: !current?.switches[i-1] && visitedSet.has(i-1)
        }">S{{ i }}</div>
        <div class="wire" :class="{ lit: current?.switches[i-1] }"></div>
      </template>
      <div class="sink lit">RCV</div>
    </div>

    <div class="dashboard">
      
      <Transition name="fade" mode="out-in">
        <div v-if="current" :key="step" class="dash-content">
          <div class="pkt-card">
            <div class="p-name">Packet {{ current.name }} arrives</div>
            <div class="p-dig">Digest = <b>{{ current.digest }}</b></div>
            <div class="p-note" :class="{ warn: current.note.includes('Duplicate'), success: current.note.includes('✓') }">
              {{ current.note }}
            </div>
          </div>
          <div class="state-card">
            <div class="s-title">Receiver Knowledge</div>
            <div class="s-known" :class="{ complete: step === PACKETS.length - 1 }">{{ current.known }}</div>
          </div>
        </div>
      </Transition>
    </div>

    <div class="ctrl">
      <button class="btn-back" :disabled="step <= -1" @click="back">← Back</button>
      <button class="btn-reset" :disabled="step < 0" @click="reset">↺ Reset</button>
      <button class="btn-next" :disabled="step >= PACKETS.length - 1" @click="advance">Next Packet →</button>
    </div>
  </div>
</template>

<style scoped>
/* Includes shared styles for all 3 components below */
*, *::before, *::after { box-sizing: border-box; }
.trace { display: flex; flex-direction: column; gap: 16px; font-family: 'JetBrains Mono', monospace; font-size: 13px; width: 100%; padding: 10px 0; }
.net { display: flex; align-items: center; height: 50px; }
.src-node, .sink { width: 44px; height: 36px; border-radius: 18px; border: 2px solid #555; display: flex; align-items: center; justify-content: center; font-size: 11px; font-weight: 700; color: #777; background: #1a1a2a; flex-shrink: 0; }
.sink.lit { border-color: #4ade80; color: #4ade80; background: #052e16; }
.wire { flex: 1; height: 2px; background: #3a3a3a; transition: background 0.3s; }
.wire.lit { background: #c084fc; }
.sw { width: 44px; height: 44px; border-radius: 9px; border: 2px solid #555; display: flex; align-items: center; justify-content: center; font-size: 13px; font-weight: 700; background: #1a1a2e; color: #666; transition: all 0.3s; flex-shrink: 0; }
.sw.active  { border-color: #c084fc; background: #2d1b4e; color: #e9d5ff; box-shadow: 0 0 12px rgba(192,132,252,0.4); transform: scale(1.1); }
.sw.visited { border-color: #4ade80; background: #052e16; color: #4ade80; opacity: 0.7; }

/* Dashboard Styles */
.dashboard { min-height: 120px; background: #111118; border: 1px solid #333; border-radius: 8px; padding: 16px; display: flex; align-items: stretch; gap: 16px; }
.placeholder { margin: auto; color: #666; font-style: italic; }
.dash-content { display: flex; width: 100%; gap: 16px; }
.pkt-card { flex: 1; background: #1a1a2a; border-radius: 6px; padding: 12px; border-left: 3px solid #c084fc; }
.p-name { color: #a1a1aa; font-size: 12px; margin-bottom: 4px; }
.p-dig { color: #fff; font-size: 16px; margin-bottom: 8px; }
.p-note { font-size: 12px; color: #60a5fa; }
.p-note.warn { color: #f87171; }
.p-note.success { color: #4ade80; font-weight: bold; }
.state-card { flex: 2; background: #1a1a2a; border-radius: 6px; padding: 12px; border-left: 3px solid #4ade80; }
.s-title { color: #a1a1aa; font-size: 12px; margin-bottom: 8px; }
.s-known { font-size: 18px; color: #e9d5ff; font-weight: bold; letter-spacing: 1px; }
.s-known.complete { color: #4ade80; }

/* Controls */
.ctrl { display: flex; gap: 10px; justify-content: center; margin-top: 4px; }
button { padding: 5px 18px; border-radius: 7px; border: 1px solid; cursor: pointer; font-family: inherit; font-size: 12px; transition: all 0.15s; }
button:disabled { opacity: 0.3; cursor: not-allowed; }
.btn-back, .btn-reset { background: #1a1a2e; border-color: #374151; color: #6b7280; }
.btn-back:hover:not(:disabled), .btn-reset:hover:not(:disabled) { background: #252540; border-color: #555; color: #aaa; }
.btn-next { background: #2d1b4e; border-color: #7c3aed; color: #c4b5fd; }
.btn-next:hover:not(:disabled) { background: #3d2b6e; border-color: #a855f7; }

.fade-enter-active, .fade-leave-active { transition: opacity 0.2s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>

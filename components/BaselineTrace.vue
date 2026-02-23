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

const step   = ref(-1)
const rootEl = ref<HTMLElement | null>(null)

function isSlideActive(): boolean {
  if (!rootEl.value) return false
  const rect = rootEl.value.getBoundingClientRect()
  return rect.width > 0 && rect.right > 0 && rect.left < window.innerWidth
}

function handleKeydown(e: KeyboardEvent) {
  if (!isSlideActive()) return
  if (e.key === 'ArrowRight' || e.key === 'Enter') {
    if (step.value < PACKETS.length - 1) { advance(); e.preventDefault(); e.stopImmediatePropagation(); }
  } else if (e.key === 'ArrowLeft') {
    if (step.value > -1) { back(); e.preventDefault(); e.stopImmediatePropagation(); }
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

// Is the current packet discovering a genuinely new switch?
const isNewDiscovery = computed(() => {
  if (!current.value || step.value < 0) return false
  const prevVisited = new Set<number>()
  for (let i = 0; i < step.value; i++) {
    PACKETS[i].switches.forEach((on, idx) => { if (on) prevVisited.add(idx) })
  }
  return current.value.switches.some((on, idx) => on && !prevVisited.has(idx))
})

const stepLabel = computed(() => {
  if (step.value < 0) return `0 / ${PACKETS.length}`
  return `${step.value + 1} / ${PACKETS.length}`
})
</script>

<template>
  <div class="trace" ref="rootEl">

    <!-- Progress bar: which switches have been found -->
    <div class="progress-bar">
      <div class="pb-label">Switches found:</div>
      <div class="pb-boxes">
        <div
          v-for="i in 5"
          :key="i"
          class="pb-box"
          :class="{ found: visitedSet.has(i-1) }"
        >S{{ i }}</div>
      </div>
      <div class="pb-count">{{ visitedSet.size }} / 5</div>
    </div>

    <!-- Network row -->
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

    <!-- Fixed info panel -->
    <div class="panel">
      <div class="panel-title">Coupon Collector path tracing: each packet samples one switch ID with probability 1/k</div>

      <Transition name="fade" mode="out-in">
        <!-- Initial state -->
        <div v-if="step < 0" :key="'init'" class="panel-init">
          Each packet carries one switch ID (written with probability 1/k = 1/5). Step through to see the coupon collector in action — how many packets does it take to find all 5 switches?
        </div>

        <!-- Final complete state -->
        <div v-else-if="step === PACKETS.length - 1 && current?.note.includes('✓')" :key="'done'" class="panel-complete">
          <span class="complete-icon">✓</span>
          <div class="complete-text">
            All 5 switches collected! Took <b>{{ PACKETS.length }} packets</b>
            <span class="complete-hint"> (k·ln k ≈ 8 expected)</span>
          </div>
        </div>

        <!-- Normal step: new or duplicate -->
        <div v-else :key="step" class="panel-step">
          <div class="step-header" :class="isNewDiscovery ? 'header-new' : 'header-dup'">
            <span class="step-badge">{{ isNewDiscovery ? '✓  NEW' : '✗  DUPLICATE' }}</span>
            <span class="step-digest">{{ current?.digest }}</span>
          </div>
          <div class="step-detail">
            <span class="detail-pkt">Packet {{ current?.name }}</span>
            <span class="detail-sep">·</span>
            <span class="detail-known">Known so far: <b>{{ current?.known }}</b></span>
          </div>
        </div>
      </Transition>
    </div>

    <!-- Controls -->
    <div class="ctrl">
      <button class="btn-back"  :disabled="step <= -1" @click="back">← Back</button>
      <button class="btn-reset" :disabled="step < 0"   @click="reset" title="Reset">↺</button>
      <span class="step-counter">Packet {{ stepLabel }}</span>
      <button class="btn-next"  :disabled="step >= PACKETS.length - 1" @click="advance">Next Packet →</button>
    </div>

  </div>
</template>

<style scoped>
*, *::before, *::after { box-sizing: border-box; }

.trace {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 6px 0;
  font-family: 'JetBrains Mono', monospace;
  font-size: 13px;
  width: 100%;
}

/* ── Progress Bar ── */
.progress-bar {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 6px 10px;
  background: #0d0d18;
  border: 1px solid #1f2937;
  border-radius: 8px;
}
.pb-label {
  font-size: 11px;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  flex-shrink: 0;
}
.pb-boxes {
  display: flex;
  gap: 6px;
  flex: 1;
}
.pb-box {
  width: 36px;
  height: 28px;
  border-radius: 6px;
  border: 2px solid #374151;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 11px;
  font-weight: 700;
  color: #4b5563;
  background: #1a1a2a;
  transition: all 0.35s;
}
.pb-box.found {
  border-color: #4ade80;
  background: #052e16;
  color: #4ade80;
}
.pb-count {
  font-size: 13px;
  font-weight: 700;
  color: #9ca3af;
  min-width: 36px;
  text-align: right;
}

/* ── Network ── */
.net { display: flex; align-items: center; height: 50px; }
.src-node, .sink {
  width: 44px; height: 36px; border-radius: 18px; border: 2px solid #555;
  display: flex; align-items: center; justify-content: center;
  font-size: 11px; font-weight: 700; color: #777; background: #1a1a2a; flex-shrink: 0;
}
.sink.lit { border-color: #4ade80; color: #4ade80; background: #052e16; }
.wire { flex: 1; height: 2px; background: #3a3a3a; transition: background 0.3s; }
.wire.lit { background: #c084fc; }
.sw {
  width: 44px; height: 44px; border-radius: 9px; border: 2px solid #555;
  display: flex; align-items: center; justify-content: center;
  font-size: 13px; font-weight: 700; background: #1a1a2e; color: #666;
  transition: all 0.3s; flex-shrink: 0;
}
.sw.active  { border-color: #c084fc; background: #2d1b4e; color: #e9d5ff; box-shadow: 0 0 12px rgba(192,132,252,0.4); transform: scale(1.1); }
.sw.visited { border-color: #4ade80; background: #052e16; color: #4ade80; opacity: 0.7; }

/* ── Info Panel ── */
.panel {
  background: #111118;
  border: 1px solid #2d2d3a;
  border-radius: 10px;
  padding: 12px 16px;
  min-height: 90px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.panel-title {
  font-size: 11px;
  color: #6b7280;
  border-bottom: 1px solid #1f2937;
  padding-bottom: 6px;
  line-height: 1.4;
}
.panel-init {
  flex: 1;
  display: flex;
  align-items: center;
  color: #4b5563;
  font-size: 13px;
  font-style: italic;
  line-height: 1.5;
}

/* Step display */
.panel-step {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 8px;
  justify-content: center;
}
.step-header {
  display: flex;
  align-items: center;
  gap: 14px;
  border-radius: 8px;
  padding: 8px 14px;
}
.header-new { background: #052e16; border: 2px solid #4ade80; }
.header-dup { background: #1a0808; border: 2px solid #f87171; }

.step-badge {
  font-size: 14px;
  font-weight: 700;
  flex-shrink: 0;
}
.header-new .step-badge { color: #4ade80; }
.header-dup .step-badge { color: #f87171; }

.step-digest {
  font-size: 18px;
  font-weight: 700;
  flex: 1;
}
.header-new .step-digest { color: #bbf7d0; }
.header-dup .step-digest { color: #fecaca; }

.step-detail {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  color: #9ca3af;
  flex-wrap: wrap;
}
.detail-sep { color: #4b5563; }
.detail-known b { color: #d1d5db; }

/* Complete state */
.panel-complete {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 12px;
  background: #052e16;
  border: 2px solid #4ade80;
  border-radius: 8px;
  padding: 10px 16px;
}
.complete-icon  { font-size: 24px; color: #4ade80; flex-shrink: 0; }
.complete-text  { font-size: 15px; color: #bbf7d0; }
.complete-hint  { font-size: 12px; color: #6b7280; }

/* ── Controls ── */
.ctrl {
  display: flex;
  align-items: center;
  gap: 8px;
  justify-content: center;
  margin-top: 4px;
}
.step-counter {
  font-size: 13px;
  color: #6b7280;
  min-width: 88px;
  text-align: center;
  font-variant-numeric: tabular-nums;
}
button {
  padding: 5px 14px;
  border-radius: 7px;
  border: 1px solid;
  cursor: pointer;
  font-family: inherit;
  font-size: 12px;
  transition: all 0.15s;
}
button:disabled { opacity: 0.3; cursor: not-allowed; }
.btn-back { background: #1a1a2e; border-color: #374151; color: #6b7280; }
.btn-back:hover:not(:disabled) { background: #252540; border-color: #555; color: #aaa; }
.btn-reset {
  background: transparent;
  border-color: transparent;
  color: #4b5563;
  padding: 5px 8px;
  font-size: 14px;
}
.btn-reset:hover:not(:disabled) { color: #9ca3af; }
.btn-next { background: #2d1b4e; border-color: #7c3aed; color: #c4b5fd; }
.btn-next:hover:not(:disabled) { background: #3d2b6e; border-color: #a855f7; }

.fade-enter-active { transition: opacity 0.2s ease; }
.fade-leave-active { transition: opacity 0.15s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>

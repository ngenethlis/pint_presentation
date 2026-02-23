<script setup lang="ts">
import { ref, computed, watch, onMounted, nextTick, onUnmounted } from 'vue'

// Reservoir sampling: switch i writes if g(pkt, i) < 1/i
const STEPS = [
  { id: 'S₁', hash: 0.08, threshold: 1.00 },  // always writes (1/1)
  { id: 'S₂', hash: 0.67, threshold: 0.50 },  // skips   (1/2)
  { id: 'S₃', hash: 0.19, threshold: 0.33 },  // writes  (1/3)
  { id: 'S₄', hash: 0.44, threshold: 0.25 },  // skips   (1/4)
]
const writes = STEPS.map(s => s.hash < s.threshold)

const step        = ref(-1)   // -1=not started  0-3=at switch  4=at sink
const hasLaunched = ref(false) // once launched, packet stays on screen (even on Back)

// ── Measure switch pixel positions after mount ──────────────────
const rootEl  = ref<HTMLElement | null>(null)
const netEl   = ref<HTMLElement | null>(null)
const pktEl   = ref<HTMLElement | null>(null)
const swEls   = ref<HTMLElement[]>([])
const sinkEl  = ref<HTMLElement | null>(null)
const swPx    = ref<number[]>([])
const sinkPx  = ref(0)

function measure() {
  if (!netEl.value || !pktEl.value) return
  const cRect = netEl.value.getBoundingClientRect()
  const pW    = pktEl.value.getBoundingClientRect().width

  swPx.value = swEls.value.map(el => {
    const r = el.getBoundingClientRect()
    return (r.left - cRect.left) + r.width / 2 - pW / 2
  })

  if (sinkEl.value) {
    const r = sinkEl.value.getBoundingClientRect()
    sinkPx.value = (r.left - cRect.left) + r.width / 2 - pW / 2
  }
}

// ── Keyboard Navigation ──────────────────────────────────────────
function isSlideActive(): boolean {
  if (!rootEl.value) return false
  const rect = rootEl.value.getBoundingClientRect()
  return rect.width > 0 && rect.right > 0 && rect.left < window.innerWidth
}

function handleKeydown(e: KeyboardEvent) {
  if (!isSlideActive()) return
  if (e.key === 'Enter' || e.key === 'ArrowRight') {
    if (step.value < STEPS.length) {
      advance()
      e.preventDefault()
      e.stopImmediatePropagation()
    }
  } else if (e.key === 'ArrowLeft') {
    if (step.value > -1) {
      back()
      e.preventDefault()
      e.stopImmediatePropagation()
    }
  }
}

onMounted(() => {
  nextTick(measure)
  window.addEventListener('keydown', handleKeydown, { capture: true })
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown, { capture: true })
})

// ── Packet position ──────────────────────────────────────────────
const pktLeft = computed(() => {
  if (step.value < 0) return hasLaunched.value ? `${swPx.value[0] ?? 0}px` : '-140px'
  if (step.value >= STEPS.length) return `${sinkPx.value}px`
  return `${swPx.value[step.value] ?? 0}px`
})

// ── Digest content ───────────────────────────────────────────────
const digest = computed(() => {
  if (step.value < 0) return '∅'
  let d = '∅'
  for (let i = 0; i <= Math.min(step.value, STEPS.length - 1); i++) {
    if (writes[i]) d = STEPS[i].id
  }
  return d
})

// Flash the digest field green ONLY when stepping forward onto a write
const digestFlash = ref(false)
watch(step, (newVal, oldVal) => {
  if (newVal > oldVal && newVal >= 0 && newVal < STEPS.length && writes[newVal]) {
    digestFlash.value = true
    setTimeout(() => (digestFlash.value = false), 700)
  }
})

// ── Controls ─────────────────────────────────────────────────────
function advance() { if (step.value < STEPS.length) { if (step.value === -1) hasLaunched.value = true; step.value++ } }
function back()    { if (step.value > -1) step.value-- }
function reset()   { step.value = -1; hasLaunched.value = false }

const btnLabel = computed(() => {
  if (step.value < 0)               return '▶  Launch packet'
  if (step.value < STEPS.length - 1) return `Next: ${STEPS[step.value + 1].id}  →`
  if (step.value === STEPS.length - 1) return 'Deliver  →'
  return '✓  Delivered'
})

// Step counter label: shows e.g. "1 / 4"
const stepLabel = computed(() => {
  if (step.value < 0) return `0 / ${STEPS.length}`
  if (step.value >= STEPS.length) return `${STEPS.length} / ${STEPS.length}`
  return `${step.value + 1} / ${STEPS.length}`
})

// State helpers
const swState = (i: number) => {
  if (i > step.value)  return 'pending'
  if (i === step.value) return 'active'
  return writes[i] ? 'wrote' : 'skipped'
}
</script>

<template>
  <div class="trace" ref="rootEl">

    <!-- Network row -->
    <div class="net" ref="netEl">
      <div
        ref="pktEl"
        class="packet"
        :class="{ visible: step >= 0 || hasLaunched }"
        :style="{ left: pktLeft }"
      >
        <span class="plabel">pkt_42</span>
        <span class="dig" :class="{ flash: digestFlash }">{{ digest }}</span>
      </div>

      <div class="src-node">SRC</div>
      <div class="wire"></div>

      <template v-for="(sw, i) in STEPS" :key="i">
        <div
          class="sw"
          :ref="(el) => { if (el) swEls[i] = el as HTMLElement }"
          :class="swState(i)"
        >{{ sw.id }}</div>
        <div class="wire" :class="{ lit: step > i }"></div>
      </template>

      <div class="sink" ref="sinkEl" :class="{ lit: step >= STEPS.length }">SNK</div>
    </div>

    <!-- Fixed info panel -->
    <div class="panel">
      <div class="panel-title">Per-flow sampling via global hash: each switch writes its ID if hash &lt; threshold</div>

      <Transition name="fade" mode="out-in">
        <!-- Initial state -->
        <div v-if="step < 0" :key="'init'" class="panel-init">
          Each switch independently computes <span class="mono">g(packet, i)</span> and writes its ID if the result is below threshold <span class="mono">1/i</span>
        </div>

        <!-- At a switch -->
        <div v-else-if="step < STEPS.length" :key="step" class="panel-card" :class="writes[step] ? 'card-write' : 'card-skip'">
          <div class="card-left">
            <div class="card-sw-id">{{ STEPS[step].id }}</div>
            <div class="card-sw-label">Switch {{ step + 1 }}</div>
          </div>
          <div class="card-mid">
            <div class="card-row">
              <span class="card-label">hash</span>
              <span class="card-val mono">g(pkt_42, {{ step + 1 }}) = <b>{{ STEPS[step].hash }}</b></span>
            </div>
            <div class="card-row">
              <span class="card-label">threshold</span>
              <span class="card-val mono">1/{{ step + 1 }} = {{ STEPS[step].threshold.toFixed(2) }}</span>
            </div>
            <div class="card-compare">
              <span class="mono">{{ STEPS[step].hash }} {{ writes[step] ? '&lt;' : '≥' }} {{ STEPS[step].threshold.toFixed(2) }}</span>
            </div>
          </div>
          <div class="card-right">
            <div class="result-box" :class="writes[step] ? 'result-write' : 'result-skip'">
              <div class="result-icon">{{ writes[step] ? '✓' : '✗' }}</div>
              <div class="result-action">
                <template v-if="writes[step]">
                  WRITES <span class="mono">{{ STEPS[step].id }}</span> into digest
                </template>
                <template v-else>
                  SKIPS — digest stays <span class="mono">{{ digest }}</span>
                </template>
              </div>
            </div>
          </div>
        </div>

        <!-- Delivered -->
        <div v-else :key="'done'" class="panel-delivered">
          <span class="delivered-label">Delivered</span>
          <span class="delivered-sep">·</span>
          digest = <span class="mono delivered-digest">{{ digest }}</span>
          <span class="delivered-sep">·</span>
          receiver learns <span class="mono delivered-digest">{{ digest }}</span> is on the path
        </div>
      </Transition>
    </div>

    <!-- Controls -->
    <div class="ctrl">
      <button class="btn-back"  :disabled="step <= -1" @click="back">← Back</button>
      <button class="btn-reset" :disabled="step < 0"   @click="reset" title="Reset">↺</button>
      <span class="step-counter">{{ stepLabel }}</span>
      <button class="btn-next"  :disabled="step >= STEPS.length" @click="advance">{{ btnLabel }}</button>
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
  font-family: 'JetBrains Mono', 'Fira Code', ui-monospace, monospace;
  font-size: 13px;
  width: 100%;
}

/* ── Network ── */
.net {
  position: relative;
  display: flex;
  align-items: center;
  height: 64px;
  overflow: visible;
}

/* Moving packet */
.packet {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  display: inline-flex;
  align-items: center;
  gap: 7px;
  background: #2d1b4e;
  border: 2px solid #a855f7;
  border-radius: 7px;
  padding: 5px 11px;
  z-index: 20;
  white-space: nowrap;
  opacity: 0;
  pointer-events: none;
  transition: left 0.55s cubic-bezier(0.4,0,0.2,1), opacity 0.25s;
}
.packet.visible { opacity: 1; }

.plabel { color: #c084fc; font-size: 11px; font-weight: 700; }

.dig {
  background: #1a0a2e;
  border: 1px solid #7c3aed;
  border-radius: 4px;
  padding: 1px 9px;
  color: #e9d5ff;
  font-weight: 700;
  min-width: 30px;
  text-align: center;
  transition: background 0.3s, border-color 0.3s, color 0.3s;
}
.dig.flash {
  background: #14532d;
  border-color: #4ade80;
  color: #86efac;
}

/* Nodes */
.src-node, .sink {
  width: 44px;
  height: 36px;
  border-radius: 18px;
  border: 2px solid #555;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 11px;
  font-weight: 700;
  color: #777;
  background: #1a1a2a;
  flex-shrink: 0;
  transition: border-color 0.4s, color 0.4s;
}
.sink.lit { border-color: #4ade80; color: #4ade80; }

.wire {
  flex: 1;
  height: 2px;
  background: #3a3a3a;
  transition: background 0.5s;
}
.wire.lit { background: #4ade80; }

/* Switch boxes */
.sw {
  width: 54px;
  height: 54px;
  border-radius: 9px;
  border: 2px solid #555;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  font-weight: 700;
  flex-shrink: 0;
  background: #1a1a2e;
  color: #666;
  transition: all 0.3s;
}
.sw.active  {
  border-color: #60a5fa;
  background: #1e3a5f;
  color: #93c5fd;
  box-shadow: 0 0 16px rgba(96,165,250,0.45);
}
.sw.wrote   { border-color: #4ade80; background: #14532d; color: #86efac; }
.sw.skipped { border-color: #2d3748; color: #4a5568; }

/* ── Info Panel ── */
.panel {
  background: #111118;
  border: 1px solid #2d2d3a;
  border-radius: 10px;
  padding: 12px 16px;
  min-height: 96px;
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

/* Card layout: left | mid | right */
.panel-card {
  flex: 1;
  display: flex;
  align-items: stretch;
  gap: 12px;
  border-radius: 8px;
  padding: 8px 12px;
}
.card-write { background: #071a0e; border: 1px solid #166534; }
.card-skip  { background: #1a0808; border: 1px solid #7f1d1d; }

.card-left {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-width: 52px;
  flex-shrink: 0;
}
.card-sw-id {
  font-size: 20px;
  font-weight: 700;
  color: #60a5fa;
  line-height: 1;
}
.card-sw-label {
  font-size: 10px;
  color: #6b7280;
  margin-top: 3px;
}

.card-mid {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 4px;
  border-left: 1px solid #1f2937;
  border-right: 1px solid #1f2937;
  padding: 0 12px;
}
.card-row {
  display: flex;
  align-items: baseline;
  gap: 8px;
}
.card-label {
  font-size: 10px;
  color: #6b7280;
  min-width: 60px;
  text-align: right;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}
.card-val {
  font-size: 13px;
  color: #d1d5db;
}
.card-compare {
  margin-top: 4px;
  font-size: 13px;
  color: #9ca3af;
  border-top: 1px solid #1f2937;
  padding-top: 4px;
}

.card-right {
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 160px;
  flex-shrink: 0;
}

.result-box {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  text-align: center;
  border-radius: 8px;
  padding: 8px 16px;
  width: 100%;
}
.result-write {
  background: #14532d;
  border: 2px solid #4ade80;
}
.result-skip {
  background: #450a0a;
  border: 2px solid #f87171;
}
.result-icon {
  font-size: 22px;
  font-weight: 700;
  line-height: 1;
}
.result-write .result-icon { color: #4ade80; }
.result-skip  .result-icon { color: #f87171; }
.result-action {
  font-size: 12px;
  line-height: 1.4;
}
.result-write .result-action { color: #bbf7d0; }
.result-skip  .result-action { color: #fecaca; }

/* Delivered state */
.panel-delivered {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  font-size: 14px;
  color: #93c5fd;
  flex-wrap: wrap;
}
.delivered-label { font-weight: 700; color: #4ade80; }
.delivered-sep   { color: #4b5563; }
.delivered-digest { color: #4ade80; font-weight: 700; }

.mono { font-family: 'JetBrains Mono', 'Fira Code', ui-monospace, monospace; }

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
  min-width: 52px;
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

.btn-back {
  background: #1a1a2e;
  border-color: #374151;
  color: #6b7280;
}
.btn-back:hover:not(:disabled) { background: #252540; border-color: #555; color: #aaa; }

.btn-reset {
  background: transparent;
  border-color: transparent;
  color: #4b5563;
  padding: 5px 8px;
  font-size: 14px;
}
.btn-reset:hover:not(:disabled) { color: #9ca3af; }

.btn-next {
  background: #2d1b4e;
  border-color: #7c3aed;
  color: #c4b5fd;
}
.btn-next:hover:not(:disabled) { background: #3d2b6e; border-color: #a855f7; }

/* Fade transition */
.fade-enter-active { transition: opacity 0.2s ease; }
.fade-leave-active { transition: opacity 0.15s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>

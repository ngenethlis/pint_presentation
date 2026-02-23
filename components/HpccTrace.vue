<script setup lang="ts">
import { ref, computed, watch, onMounted, nextTick, onUnmounted } from 'vue'

// HPCC Bottleneck tracking: c = floor(log_1.05(v))
const STEPS = [
  { id: 'S₁', util: 30000, encoded: 211 }, // 1.05^211 ≈ 29646
  { id: 'S₂', util: 75000, encoded: 230 }, // 1.05^230 ≈ 74737 (bottleneck)
  { id: 'S₃', util: 10000, encoded: 188 }, // 1.05^188 ≈ 9668
  { id: 'S₄', util: 50000, encoded: 221 }, // 1.05^221 ≈ 48296
]

// Determine dynamically which switches will write (update the max)
const writes: boolean[] = []
let currentMax = 0
for (let i = 0; i < STEPS.length; i++) {
  if (STEPS[i].encoded > currentMax) {
    writes.push(true)
    currentMax = STEPS[i].encoded
  } else {
    writes.push(false)
  }
}

const step = ref(-1)   // -1=not started  0-3=at switch  4=at sink

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
  if (e.key === 'ArrowRight' || e.key === 'Enter') {
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
  if (step.value < 0)             return '-140px'
  if (step.value >= STEPS.length) return `${sinkPx.value}px`
  return `${swPx.value[step.value] ?? 0}px`
})

// ── Digest (Max Encoded Value) ───────────────────────────────────
const digest = computed(() => {
  if (step.value < 0) return 0
  let d = 0
  for (let i = 0; i <= Math.min(step.value, STEPS.length - 1); i++) {
    if (STEPS[i].encoded > d) d = STEPS[i].encoded
  }
  return d
})

function prevDigest(currentStep: number) {
  let d = 0
  for (let i = 0; i < currentStep; i++) {
    if (STEPS[i].encoded > d) d = STEPS[i].encoded
  }
  return d
}

const decodedValue = computed(() => {
  return Math.round(Math.pow(1.05, digest.value)).toLocaleString()
})

// True bottleneck: switch with highest encoded value
const trueBottleneck = computed(() => {
  return STEPS.reduce((max, sw) => sw.encoded > max.encoded ? sw : max)
})

// Multiplicative error on delivery
const errorPct = computed(() => {
  const decoded = Math.round(Math.pow(1.05, digest.value))
  return (Math.abs(trueBottleneck.value.util - decoded) / trueBottleneck.value.util * 100).toFixed(2)
})

// Per-switch: decode error for each encoded value
function swDecodeError(sw: typeof STEPS[0]) {
  const decoded = Math.round(Math.pow(1.05, sw.encoded))
  return (Math.abs(sw.util - decoded) / sw.util * 100).toFixed(2)
}

// Flash the digest field green ONLY on forward write
const digestFlash = ref(false)
watch(step, (newVal, oldVal) => {
  if (newVal > oldVal && newVal >= 0 && newVal < STEPS.length && writes[newVal]) {
    digestFlash.value = true
    setTimeout(() => (digestFlash.value = false), 700)
  }
})

// ── Controls ─────────────────────────────────────────────────────
function advance() { if (step.value < STEPS.length) step.value++ }
function back()    { if (step.value > -1) step.value-- }
function reset()   { step.value = -1 }

const btnLabel = computed(() => {
  if (step.value < 0)               return '▶  Launch'
  if (step.value < STEPS.length - 1) return `Next: ${STEPS[step.value + 1].id}  →`
  if (step.value === STEPS.length - 1) return 'Deliver  →'
  return '✓  Delivered'
})

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
        :class="{ visible: step >= 0 }"
        :style="{ left: pktLeft }"
      >
        <span class="plabel">INT_HDR</span>
        <span class="dig" :class="{ flash: digestFlash }">{{ digest }}</span>
      </div>

      <div class="src-node">SRC</div>
      <div class="wire"></div>

      <template v-for="(sw, i) in STEPS" :key="i">
        <div
          class="sw"
          :ref="(el) => { if (el) swEls[i] = el as HTMLElement }"
          :class="swState(i)"
        >
          <span class="sw-id">{{ sw.id }}</span>
          <span class="sw-util">{{ sw.util / 1000 }}k</span>
        </div>
        <div class="wire" :class="{ lit: step > i }"></div>
      </template>

      <div class="sink" ref="sinkEl" :class="{ lit: step >= STEPS.length }">SNK</div>
    </div>

    <!-- Fixed info panel -->
    <div class="panel">
      <div class="panel-title">Bottleneck tracking: packet carries a running maximum — each switch updates it if its utilisation is higher</div>

      <Transition name="fade" mode="out-in">
        <!-- Initial state -->
        <div v-if="step < 0" :key="'init'" class="panel-init">
          Each switch compares its link utilisation to the packet's current maximum and overwrites it if higher. After traversal, the receiver knows the bottleneck bandwidth.
        </div>

        <!-- At a switch -->
        <div v-else-if="step < STEPS.length" :key="step" class="panel-card" :class="writes[step] ? 'card-write' : 'card-skip'">
          <div class="card-left">
            <div class="card-sw-id">{{ STEPS[step].id }}</div>
            <div class="card-util">{{ STEPS[step].util.toLocaleString() }} Mbps</div>
          </div>
          <div class="card-mid">
            <div class="card-section-label">Encoding</div>
            <div class="card-row">
              <span class="card-label">encode</span>
              <span class="card-val mono">c = ⌊log<sub>1.05</sub>({{ STEPS[step].util.toLocaleString() }})⌋ = <b>{{ STEPS[step].encoded }}</b></span>
            </div>
            <div class="card-row">
              <span class="card-label">pkt max was</span>
              <span class="card-val mono"><b>{{ prevDigest(step) }}</b></span>
            </div>
            <div class="card-row card-decode-check">
              <span class="card-label">decode check</span>
              <span class="card-val mono">1.05<sup>{{ STEPS[step].encoded }}</sup> ≈ {{ Math.round(Math.pow(1.05, STEPS[step].encoded)).toLocaleString() }} &nbsp;(err {{ swDecodeError(STEPS[step]) }}%)</span>
            </div>
          </div>
          <div class="card-right">
            <div class="result-box" :class="writes[step] ? 'result-write' : 'result-skip'">
              <div class="result-icon">{{ writes[step] ? '✓' : '✗' }}</div>
              <div class="result-action">
                <template v-if="writes[step]">
                  NEW MAX<br>updates digest to <span class="mono">{{ STEPS[step].encoded }}</span>
                </template>
                <template v-else>
                  <span class="mono">{{ STEPS[step].encoded }} ≤ {{ prevDigest(step) }}</span><br>skips — digest stays <span class="mono">{{ digest }}</span>
                </template>
              </div>
            </div>
          </div>
        </div>

        <!-- Delivered: 2-column summary -->
        <div v-else :key="'done'" class="panel-delivered-card">
          <div class="delivered-col">
            <div class="delivered-col-title">Packet Result</div>
            <div class="delivered-row">
              <span class="dlabel">Bottleneck digest:</span>
              <span class="mono dval-green">{{ digest }} <span class="dunit">(1 byte)</span></span>
            </div>
            <div class="delivered-row">
              <span class="dlabel">Decoded:</span>
              <span class="mono dval-green">1.05<sup>{{ digest }}</sup> ≈ {{ decodedValue }} Mbps</span>
            </div>
          </div>
          <div class="delivered-divider"></div>
          <div class="delivered-col">
            <div class="delivered-col-title">Accuracy Check</div>
            <div class="delivered-row">
              <span class="dlabel">True bottleneck:</span>
              <span class="mono dval">{{ trueBottleneck.util.toLocaleString() }} Mbps</span>
            </div>
            <div class="delivered-row">
              <span class="dlabel">Error:</span>
              <span class="mono dval-green">{{ errorPct }}% &lt; 5% ✓</span>
            </div>
          </div>
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

.sw {
  width: 56px;
  height: 56px;
  border-radius: 9px;
  border: 2px solid #555;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 2px;
  flex-shrink: 0;
  background: #1a1a2e;
  color: #666;
  transition: all 0.3s;
}
.sw-id   { font-size: 13px; font-weight: 700; line-height: 1; }
.sw-util { font-size: 10px; font-weight: 400; opacity: 0.7; line-height: 1; }

.sw.active {
  border-color: #60a5fa;
  background: #1e3a5f;
  color: #93c5fd;
  box-shadow: 0 0 16px rgba(96,165,250,0.45);
}
.sw.active .sw-util { opacity: 0.9; color: #bfdbfe; }
.sw.wrote   { border-color: #4ade80; background: #14532d; color: #86efac; }
.sw.skipped { border-color: #2d3748; color: #4a5568; }

/* ── Info Panel ── */
.panel {
  background: #111118;
  border: 1px solid #2d2d3a;
  border-radius: 10px;
  padding: 12px 16px;
  min-height: 104px;
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
  min-width: 64px;
  flex-shrink: 0;
}
.card-sw-id {
  font-size: 20px;
  font-weight: 700;
  color: #60a5fa;
  line-height: 1;
}
.card-util {
  font-size: 10px;
  color: #9ca3af;
  margin-top: 4px;
  text-align: center;
}

.card-mid {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 3px;
  border-left: 1px solid #1f2937;
  border-right: 1px solid #1f2937;
  padding: 0 12px;
}
.card-section-label {
  font-size: 10px;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 2px;
}
.card-row {
  display: flex;
  align-items: baseline;
  gap: 8px;
  flex-wrap: wrap;
}
.card-label {
  font-size: 10px;
  color: #6b7280;
  min-width: 72px;
  text-align: right;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  flex-shrink: 0;
}
.card-val {
  font-size: 13px;
  color: #d1d5db;
}
.card-decode-check { opacity: 0.65; }

.card-right {
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 156px;
  flex-shrink: 0;
}

.result-box {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  text-align: center;
  border-radius: 8px;
  padding: 8px 12px;
  width: 100%;
}
.result-write { background: #14532d; border: 2px solid #4ade80; }
.result-skip  { background: #450a0a; border: 2px solid #f87171; }

.result-icon {
  font-size: 22px;
  font-weight: 700;
  line-height: 1;
}
.result-write .result-icon { color: #4ade80; }
.result-skip  .result-icon { color: #f87171; }
.result-action {
  font-size: 11px;
  line-height: 1.5;
}
.result-write .result-action { color: #bbf7d0; }
.result-skip  .result-action { color: #fecaca; }

/* Delivered 2-col card */
.panel-delivered-card {
  flex: 1;
  display: flex;
  align-items: stretch;
  gap: 0;
  background: #0a1628;
  border: 1px solid #1d4ed8;
  border-radius: 8px;
  padding: 8px 0;
}
.delivered-col {
  flex: 1;
  padding: 4px 16px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.delivered-col-title {
  font-size: 11px;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  border-bottom: 1px solid #1f2937;
  padding-bottom: 4px;
  margin-bottom: 2px;
}
.delivered-divider {
  width: 1px;
  background: #1f2937;
  flex-shrink: 0;
}
.delivered-row {
  display: flex;
  align-items: baseline;
  gap: 8px;
  flex-wrap: wrap;
}
.dlabel  { font-size: 12px; color: #6b7280; min-width: 120px; }
.dval    { font-size: 13px; color: #d1d5db; }
.dval-green { font-size: 13px; color: #4ade80; font-weight: 700; }
.dunit   { font-size: 11px; color: #6b7280; }

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

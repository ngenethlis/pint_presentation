<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'

const PACKETS = [
  { name: 'p₁',  switches: [0, 1, 0, 1, 0], digest: 'S₂ ⊕ S₄', known: '{}',
    eqs: [{ text: 'p₁: S₂ ⊕ S₄', status: '2 unknowns — wait' }],
    note: 'Two unknowns, cannot solve yet' },

  { name: 'p₂',  switches: [0, 0, 0, 1, 0], digest: 'S₄', known: '{ S₄ }',
    eqs: [
      { text: 'p₁: S₂ ⊕ S₄', status: '2 unknowns' },
      { text: 'p₂: S₄',       status: '1 unknown → solve!', done: true }
    ],
    note: 'One unknown → solve directly: S₄ decoded' },

  { name: 'back-sub into p₁', switches: [0, 1, 0, 0, 0], digest: 'sub S₄ → S₂', known: '{ S₂, S₄ }',
    eqs: [
      { text: 'p₁: S₂ ⊕ <s>S₄</s>', status: 'substitute S₄ → solve S₂', done: true },
      { text: 'p₂: S₄',              status: '✓ solved', done: true }
    ],
    note: 'Substitute known S₄ into p₁ → S₂ decoded' },

  { name: 'p₃', switches: [1, 1, 0, 1, 0], digest: 'S₁ ⊕ S₂ ⊕ S₄', known: '{ S₁, S₂, S₄ }',
    eqs: [
      { text: 'p₁: S₂',                          status: '✓ solved', done: true },
      { text: 'p₂: S₄',                          status: '✓ solved', done: true },
      { text: 'p₃: S₁ ⊕ <s>S₂</s> ⊕ <s>S₄</s>', status: 'substitute → S₁ decoded', done: true }
    ],
    note: 'Substitute known S₂, S₄ → S₁ decoded' },

  { name: 'p₄ + p₅ (summary)', switches: [0, 0, 1, 0, 1], digest: 'S₃ ⊕ S₅, S₅', known: '{ S₁, S₂, S₃, S₄, S₅ }',
    eqs: [
      { text: 'p₄: S₃ ⊕ <s>S₅</s>', status: 'substitute → S₃ decoded', done: true },
      { text: 'p₅: S₅',              status: '→ S₅ decoded', done: true }
    ],
    note: '✓ All 5 switches recovered in ~5 packets' }
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

// Count how many switches are known from the `known` field
const knownCount = computed(() => {
  if (!current.value) return 0
  return (current.value.known.match(/S/g) || []).length
})

// Which switch indices (0-based) are known
const knownSwitches = computed(() => {
  const known = new Set<number>()
  if (!current.value) return known
  const matches = current.value.known.matchAll(/S(\d)/g)
  for (const m of matches) known.add(parseInt(m[1]) - 1)
  return known
})

const stepLabel = computed(() => {
  if (step.value < 0) return `0 / ${PACKETS.length}`
  return `${step.value + 1} / ${PACKETS.length}`
})
</script>

<template>
  <div class="trace" ref="rootEl">

    <!-- Progress bar: recovered switches -->
    <div class="progress-bar">
      <div class="pb-label">Switches recovered:</div>
      <div class="pb-boxes">
        <div
          v-for="i in 5"
          :key="i"
          class="pb-box"
          :class="{ known: knownSwitches.has(i-1) }"
        >S{{ i }}</div>
      </div>
      <div class="pb-count">{{ knownCount }} / 5</div>
    </div>

    <!-- Network row -->
    <div class="net">
      <div class="src-node">SRC</div>
      <div class="wire"></div>
      <template v-for="i in 5" :key="i">
        <div class="sw" :class="{ active: current?.switches[i-1] }">S{{ i }}</div>
        <div class="wire" :class="{ lit: current?.switches[i-1] }"></div>
      </template>
      <div class="sink lit">RCV</div>
    </div>

    <!-- Fixed info panel: 2-column dashboard -->
    <div class="panel">
      <div class="panel-title">XOR equation system: digests = XOR of traversed switch IDs, solved by back-substitution (Gaussian elimination over 𝔽₂)</div>

      <Transition name="fade" mode="out-in">
        <!-- Initial state -->
        <div v-if="!current" :key="'init'" class="panel-init">
          <div class="init-body">
            Packets carry XOR digests of traversed switches. We collect equations and solve by back-substitution (Gaussian elimination over 𝔽₂).
            <div class="init-example">e.g. packet through S₂, S₄ &nbsp;→&nbsp; digest = <span class="mono"><b>S₂ ⊕ S₄</b></span></div>
          </div>
        </div>

        <!-- Active step: 2-col dashboard -->
        <div v-else :key="step" class="dash-content">

          <!-- Left: packet info -->
          <div class="dash-left">
            <div class="left-label">Packet</div>
            <div class="left-name">{{ current.name }}</div>
            <div class="left-digest mono" v-html="current.digest"></div>
            <div class="left-note" :class="{ 'note-success': current.note.includes('✓') }">{{ current.note }}</div>
            <div class="left-known">Known: <span class="mono known-val">{{ current.known }}</span></div>
          </div>

          <!-- Right: equation system -->
          <div class="dash-right">
            <div class="eq-title">Equation System</div>
            <div class="eq-list">
              <div
                v-for="(eq, idx) in current.eqs"
                :key="idx"
                class="eq-line"
                :class="{ done: eq.done, pending: !eq.done }"
              >
                <span class="eq-text mono" v-html="eq.text"></span>
                <span class="eq-status">{{ eq.status }}</span>
              </div>
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
      <button class="btn-next"  :disabled="step >= PACKETS.length - 1" @click="advance">Next Step →</button>
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
.pb-boxes { display: flex; gap: 6px; flex: 1; }
.pb-box {
  width: 36px; height: 28px; border-radius: 6px;
  border: 2px solid #374151;
  display: flex; align-items: center; justify-content: center;
  font-size: 11px; font-weight: 700;
  color: #4b5563; background: #1a1a2a;
  transition: all 0.35s;
}
.pb-box.known { border-color: #4ade80; background: #052e16; color: #4ade80; }
.pb-count { font-size: 13px; font-weight: 700; color: #9ca3af; min-width: 36px; text-align: right; }

/* ── Network ── */
.net { display: flex; align-items: center; height: 50px; }
.src-node, .sink {
  width: 44px; height: 36px; border-radius: 18px; border: 2px solid #555;
  display: flex; align-items: center; justify-content: center;
  font-size: 11px; font-weight: 700; color: #777; background: #1a1a2a; flex-shrink: 0;
}
.sink.lit { border-color: #60a5fa; color: #60a5fa; background: #0c1a3a; }
.wire { flex: 1; height: 2px; background: #3a3a3a; transition: background 0.3s; }
.wire.lit { background: #60a5fa; }
.sw {
  width: 44px; height: 44px; border-radius: 9px; border: 2px solid #555;
  display: flex; align-items: center; justify-content: center;
  font-size: 13px; font-weight: 700; background: #1a1a2e; color: #666;
  transition: all 0.3s; flex-shrink: 0;
}
.sw.active { border-color: #60a5fa; background: #1e3a8a; color: #bfdbfe; box-shadow: 0 0 12px rgba(96,165,250,0.4); transform: scale(1.1); }

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
}
.init-body {
  color: #4b5563;
  font-size: 13px;
  font-style: italic;
  line-height: 1.6;
}
.init-example {
  margin-top: 8px;
  font-style: normal;
  color: #bfdbfe;
  font-size: 13px;
  background: #0c1a3a;
  border: 1px solid #1d4ed8;
  border-radius: 6px;
  padding: 4px 12px;
  display: inline-block;
}

/* 2-column dashboard */
.dash-content {
  flex: 1;
  display: flex;
  gap: 12px;
  align-items: stretch;
}

/* Left card */
.dash-left {
  flex: 1;
  background: #0c1a3a;
  border: 1px solid #1d4ed8;
  border-radius: 8px;
  padding: 10px 14px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.left-label {
  font-size: 10px;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.06em;
}
.left-name {
  font-size: 14px;
  font-weight: 700;
  color: #93c5fd;
}
.left-digest {
  font-size: 16px;
  font-weight: 700;
  color: #fff;
  margin: 2px 0;
}
.left-note {
  font-size: 12px;
  color: #94a3b8;
  line-height: 1.4;
}
.left-note.note-success { color: #4ade80; font-weight: 700; }
.left-known {
  margin-top: 4px;
  font-size: 12px;
  color: #6b7280;
}
.known-val { color: #93c5fd; }

/* Right card: equations */
.dash-right {
  flex: 2;
  background: #130a2a;
  border: 1px solid #5b21b6;
  border-radius: 8px;
  padding: 10px 14px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.eq-title {
  font-size: 11px;
  color: #9ca3af;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  border-bottom: 1px solid #1f2937;
  padding-bottom: 4px;
}
.eq-list {
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.eq-line {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 8px;
  padding: 4px 6px;
  border-radius: 5px;
}
.eq-line.pending { background: #1a1028; }
.eq-line.done    { background: #071a0e; }

.eq-text {
  font-size: 13px;
  color: #c4b5fd;
  flex-shrink: 0;
}
.eq-line.done .eq-text { color: #86efac; }
.eq-text :deep(s) { color: #ef4444; text-decoration-color: #ef4444; opacity: 0.8; }

.eq-status {
  font-size: 11px;
  color: #6b7280;
  text-align: right;
  flex: 1;
}
.eq-line.done .eq-status { color: #4ade80; font-weight: 700; }

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
.btn-next { background: #1e3a8a; border-color: #3b82f6; color: #bfdbfe; }
.btn-next:hover:not(:disabled) { background: #1e40af; border-color: #60a5fa; }

.fade-enter-active { transition: opacity 0.2s ease; }
.fade-leave-active { transition: opacity 0.15s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'

const PACKETS = [
  { name: 'p₁', switches: [0, 1, 0, 1, 0], digest: 'S₂ ⊕ S₄', known: '{}', 
    eqs: [{ text: 'p₁: S₂ ⊕ S₄', status: '(2 unknowns)' }], 
    note: '2 unknowns, wait...' },

  { name: 'p₂', switches: [0, 0, 0, 1, 0], digest: 'S₄', known: '{ S₄ }', 
    eqs: [
      { text: 'p₁: S₂ ⊕ S₄', status: '(2 unknowns)' }, 
      { text: 'p₂: S₄', status: '→ decode S₄ ✓', done: true }
    ], 
    note: '1 unknown → decode S₄!' },

  { name: 'p₁ (back-sub)', switches: [0, 1, 0, 0, 0], digest: 'sub S₄', known: '{ S₂, S₄ }', 
    eqs: [
      { text: 'p₁: S₂ ⊕ <s>S₄</s>', status: '→ decode S₂ ✓', done: true }, 
      { text: 'p₂: S₄', status: '✓', done: true }
    ], 
    note: 'Back-substitute S₄ into p₁ to decode S₂!' },

  { name: 'p₃', switches: [1, 1, 0, 1, 0], digest: 'S₁ ⊕ S₂ ⊕ S₄', known: '{ S₁, S₂, S₄ }', 
    eqs: [
      { text: 'p₁: S₂', status: '✓', done: true }, 
      { text: 'p₂: S₄', status: '✓', done: true },
      { text: 'p₃: S₁ ⊕ <s>S₂</s> ⊕ <s>S₄</s>', status: '→ decode S₁ ✓', done: true }
    ], 
    note: 'Known S₂, S₄ allows decoding S₁' },
    
  { name: 'Skip to end', switches: [0, 0, 1, 0, 1], digest: 'S₃ ⊕ S₅...', known: '{ S₁, S₂, S₃, S₄, S₅ }', 
    eqs: [
      { text: 'p₄: S₃ ⊕ <s>S₅</s>', status: '→ decode S₃ ✓', done: true },
      { text: 'p₅: S₅', status: '→ decode S₅ ✓', done: true }
    ], 
    note: '✓ All 5 switches recovered' }
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
</script>

<template>
  <div class="trace" ref="rootEl">
    <div class="net">
      <div class="src-node">SRC</div>
      <div class="wire"></div>
      <template v-for="i in 5" :key="i">
        <div class="sw" :class="{ active: current?.switches[i-1] }">S{{ i }}</div>
        <div class="wire" :class="{ lit: current?.switches[i-1] }"></div>
      </template>
      <div class="sink lit">RCV</div>
    </div>

    <div class="dashboard xor-dash">

      <Transition name="fade" mode="out-in">
        <div v-if="!current" :key="-1" class="init-state">
          <div class="init-title">XOR Encoding</div>
          <div class="init-desc">Each packet's digest = XOR of the IDs of switches it traversed</div>
          <div class="init-example">e.g. &nbsp; packet through S₂, S₄ &nbsp;→&nbsp; digest = <b>S₂ ⊕ S₄</b></div>
          <div class="init-desc" style="margin-top:6px; opacity:0.6">Collect enough equations → solve for all switch IDs</div>
        </div>
        <div v-else :key="step" class="dash-content">
          <div class="pkt-card">
            <div class="p-name">Packet {{ current.name }}</div>
            <div class="p-dig" v-html="current.digest"></div>
            <div class="p-note" :class="{ success: current.note.includes('✓') }">{{ current.note }}</div>
            <div class="s-known mt-4 text-sm">{{ current.known }}</div>
          </div>
          <div class="state-card eq-card">
            <div class="s-title">Equations Map</div>
            <div class="eq-list">
              <div v-for="(eq, idx) in current.eqs" :key="idx" class="eq-line" :class="{ done: eq.done }">
                <span class="eq-left" v-html="eq.text"></span>
                <span class="eq-right">{{ eq.status }}</span>
              </div>
            </div>
          </div>
        </div>
      </Transition>
    </div>

    <div class="ctrl">
      <button class="btn-back" :disabled="step <= -1" @click="back">← Back</button>
      <button class="btn-reset" :disabled="step < 0" @click="reset">↺ Reset</button>
      <button class="btn-next" :disabled="step >= PACKETS.length - 1" @click="advance">Next Step →</button>
    </div>
  </div>
</template>

<style scoped>
/* Inherit core CSS from baseline trace, inject XOR specific tweaks below */
*, *::before, *::after { box-sizing: border-box; }
.trace { display: flex; flex-direction: column; gap: 16px; font-family: 'JetBrains Mono', monospace; font-size: 13px; width: 100%; padding: 10px 0; }
.net { display: flex; align-items: center; height: 50px; }
.src-node, .sink { width: 44px; height: 36px; border-radius: 18px; border: 2px solid #555; display: flex; align-items: center; justify-content: center; font-size: 11px; font-weight: 700; color: #777; background: #1a1a2a; flex-shrink: 0; }
.sink.lit { border-color: #60a5fa; color: #60a5fa; background: #1e3a8a; }
.wire { flex: 1; height: 2px; background: #3a3a3a; transition: background 0.3s; }
.wire.lit { background: #60a5fa; }
.sw { width: 44px; height: 44px; border-radius: 9px; border: 2px solid #555; display: flex; align-items: center; justify-content: center; font-size: 13px; font-weight: 700; background: #1a1a2e; color: #666; transition: all 0.3s; flex-shrink: 0; }
.sw.active { border-color: #60a5fa; background: #1e3a8a; color: #bfdbfe; box-shadow: 0 0 12px rgba(96,165,250,0.4); transform: scale(1.1); }

.dashboard { min-height: 140px; background: #111118; border: 1px solid #333; border-radius: 8px; padding: 16px; display: flex; align-items: stretch; gap: 16px; }
.placeholder { margin: auto; color: #666; font-style: italic; }
.init-state { margin: auto; text-align: center; padding: 8px; }
.init-title { color: #60a5fa; font-size: 14px; font-weight: 700; margin-bottom: 6px; }
.init-desc  { color: #94a3b8; font-size: 12px; }
.init-example { color: #bfdbfe; font-size: 13px; margin-top: 4px; background: #1e3a8a44; border: 1px solid #3b82f6; border-radius: 6px; padding: 4px 12px; display: inline-block; }
.dash-content { display: flex; width: 100%; gap: 16px; }
.pkt-card { flex: 1; background: #1a1a2a; border-radius: 6px; padding: 12px; border-left: 3px solid #60a5fa; }
.p-name { color: #a1a1aa; font-size: 12px; margin-bottom: 4px; }
.p-dig { color: #fff; font-size: 16px; margin-bottom: 8px; font-weight: bold; }
.p-note { font-size: 12px; color: #94a3b8; }
.p-note.success { color: #4ade80; font-weight: bold; }
.s-known { font-size: 14px; color: #bfdbfe; font-weight: bold; margin-top: 12px; }

.eq-card { flex: 2; background: #1a1a2a; border-radius: 6px; padding: 12px; border-left: 3px solid #f472b6; }
.s-title { color: #a1a1aa; font-size: 12px; margin-bottom: 8px; border-bottom: 1px solid #333; padding-bottom: 4px; }
.eq-line { display: flex; justify-content: space-between; padding: 4px 0; color: #94a3b8; }
.eq-line.done .eq-right { color: #4ade80; font-weight: bold; }
.eq-left :deep(s) { color: #ef4444; text-decoration-color: #ef4444; opacity: 0.8; }

.ctrl { display: flex; gap: 10px; justify-content: center; margin-top: 4px; }
button { padding: 5px 18px; border-radius: 7px; border: 1px solid; cursor: pointer; font-family: inherit; font-size: 12px; transition: all 0.15s; }
button:disabled { opacity: 0.3; cursor: not-allowed; }
.btn-back, .btn-reset { background: #1a1a2e; border-color: #374151; color: #6b7280; }
.btn-back:hover:not(:disabled), .btn-reset:hover:not(:disabled) { background: #252540; border-color: #555; color: #aaa; }
.btn-next { background: #1e3a8a; border-color: #3b82f6; color: #bfdbfe; }
.btn-next:hover:not(:disabled) { background: #1e40af; border-color: #60a5fa; }

.fade-enter-active, .fade-leave-active { transition: opacity 0.2s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>

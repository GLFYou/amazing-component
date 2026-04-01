<template>
  <section
    ref="containerRef"
    class="cyber-snake"
    @pointermove="handlePointerMove"
    @pointerleave="handlePointerLeave"
    @pointerdown="handlePointerDown"
  >
    <div class="cyber-snake__bg"></div>
    <div class="cyber-snake__grid"></div>
    <div class="cyber-snake__aurora"></div>

    <header class="cyber-snake__hud">
      <p class="eyebrow">Bio-Digital Motion Interface</p>
      <h2>CYBER SNAKE</h2>
      <p class="desc">移动鼠标引导蛇形能量体巡航，点击任意区域触发脉冲爆裂与神经激活特效。</p>
    </header>

    <div class="cyber-snake__stage">
      <div
        v-for="segment in renderSegments"
        :key="segment.id"
        class="snake-segment"
        :class="{ 'snake-segment--head': segment.id === 0 }"
        :style="segment.style"
      >
        <div v-if="segment.id === 0" class="snake-head__face">
          <span class="snout-core"></span>
          <span class="eye eye--left"></span>
          <span class="eye eye--right"></span>
          <span class="jaw jaw--top"></span>
          <span class="jaw jaw--bottom"></span>
          <span class="tongue">
            <i class="tongue-stem"></i>
            <i class="tongue-fork tongue-fork--left"></i>
            <i class="tongue-fork tongue-fork--right"></i>
          </span>
        </div>
      </div>

      <div
        v-for="burst in bursts"
        :key="burst.id"
        class="pulse-burst"
        :style="{
          left: `${burst.x}px`,
          top: `${burst.y}px`,
          '--burst-hue': burst.hue
        }"
      ></div>

      <div
        v-for="callout in callouts"
        :key="callout.id"
        class="signal-callout"
        :style="{
          left: `${callout.x}px`,
          top: `${callout.y}px`
        }"
      >
        呼叫黄清华~
      </div>

      <div class="target-reticle" :style="reticleStyle">
        <span></span>
        <span></span>
      </div>
    </div>

    <footer class="cyber-snake__tips">
      <span>Velocity {{ displayVelocity }}</span>
      <span>Charge {{ chargeLevel }}%</span>
      <span>Status Neural Link Stable</span>
    </footer>
  </section>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref } from 'vue'

const containerRef = ref(null)
const segmentCount = 22
const baseHue = 188
const chargeBoost = ref(0)
const velocity = ref(0)
const bursts = ref([])
const callouts = ref([])

const pointer = reactive({
  x: 0,
  y: 0,
  active: false
})

const segments = reactive(
  Array.from({ length: segmentCount }, (_, index) => ({
    x: 0,
    y: 0,
    angle: 0,
    size: 74 - index * 2.2
  }))
)

let rafId = 0
let burstId = 0
let calloutId = 0
let autoMotionStart = Date.now()

const centerPointer = () => {
  const el = containerRef.value
  if (!el) return
  const rect = el.getBoundingClientRect()
  pointer.x = rect.width / 2
  pointer.y = rect.height / 2
}

const clamp = (value, min, max) => Math.min(Math.max(value, min), max)

const handlePointerMove = (event) => {
  const rect = containerRef.value?.getBoundingClientRect()
  if (!rect) return
  pointer.x = event.clientX - rect.left
  pointer.y = event.clientY - rect.top
  pointer.active = true
}

const handlePointerLeave = () => {
  pointer.active = false
  autoMotionStart = Date.now()
}

const spawnBurst = (x, y) => {
  const hue = baseHue + Math.round(Math.random() * 60)
  const id = burstId++
  bursts.value = [...bursts.value, { id, x, y, hue }]
  window.setTimeout(() => {
    bursts.value = bursts.value.filter((item) => item.id !== id)
  }, 900)
}

const spawnCallout = (x, y) => {
  const id = calloutId++
  callouts.value = [...callouts.value, { id, x, y }]
  window.setTimeout(() => {
    callouts.value = callouts.value.filter((item) => item.id !== id)
  }, 1400)
}

const handlePointerDown = (event) => {
  const rect = containerRef.value?.getBoundingClientRect()
  if (!rect) return
  const x = event.clientX - rect.left
  const y = event.clientY - rect.top
  pointer.x = x
  pointer.y = y
  pointer.active = true
  autoMotionStart = Date.now()
  chargeBoost.value = 34
  spawnBurst(x, y)
  spawnCallout(x, y - 18)
}

const getAutoPilotTarget = () => {
  const el = containerRef.value
  const head = segments[0]
  if (!el || !head) {
    return { x: pointer.x, y: pointer.y }
  }

  const rect = el.getBoundingClientRect()
  const elapsed = (Date.now() - autoMotionStart) / 1000
  const centerX = rect.width / 2
  const centerY = rect.height / 2
  const orbitX = Math.sin(elapsed * 0.85) * rect.width * 0.26
  const orbitY = Math.cos(elapsed * 1.15) * rect.height * 0.18
  const wobbleX = Math.cos(elapsed * 2.4) * 40
  const wobbleY = Math.sin(elapsed * 2.9) * 28

  return {
    x: clamp(centerX + orbitX + wobbleX, 60, rect.width - 60),
    y: clamp(centerY + orbitY + wobbleY, 70, rect.height - 70)
  }
}

const updateSnake = () => {
  const head = segments[0]
  const autoTarget = getAutoPilotTarget()
  const followX = pointer.active ? pointer.x : autoTarget.x
  const followY = pointer.active ? pointer.y : autoTarget.y

  head.x += (followX - head.x) * 0.18
  head.y += (followY - head.y) * 0.18

  let totalVelocity = Math.hypot(followX - head.x, followY - head.y)

  for (let i = 1; i < segments.length; i += 1) {
    const prev = segments[i - 1]
    const current = segments[i]
    const dx = prev.x - current.x
    const dy = prev.y - current.y
    current.x += dx * 0.28
    current.y += dy * 0.28
    current.angle = Math.atan2(dy, dx) * (180 / Math.PI)
    totalVelocity += Math.abs(dx) + Math.abs(dy)
  }

  head.angle = Math.atan2(followY - head.y, followX - head.x) * (180 / Math.PI)
  velocity.value = clamp(totalVelocity / 18, 8, 99)
  chargeBoost.value = Math.max(0, chargeBoost.value - 0.55)
  rafId = window.requestAnimationFrame(updateSnake)
}

const renderSegments = computed(() =>
  segments.map((segment, index) => {
    const intensity = 1 - index / segments.length
    const scale = 1 - index * 0.026 + chargeBoost.value * 0.002
    const glow = 18 + intensity * 30 + chargeBoost.value * 0.6
    const hue = baseHue + index * 6 + chargeBoost.value * 0.4

    return {
      id: index,
      style: {
        width: `${segment.size}px`,
        height: `${segment.size * (index === 0 ? 0.88 : 0.72)}px`,
        transform: `translate3d(${segment.x}px, ${segment.y}px, 0) translate(-50%, -50%) rotate(${segment.angle}deg) scale(${scale})`,
        opacity: `${0.18 + intensity * 0.82}`,
        zIndex: `${segments.length - index}`,
        '--segment-hue': hue,
        '--segment-glow': `${glow}px`
      }
    }
  })
)

const reticleStyle = computed(() => ({
  left: `${pointer.x}px`,
  top: `${pointer.y}px`,
  opacity: pointer.active ? 1 : 0.45
}))

const chargeLevel = computed(() => Math.round(clamp(52 + chargeBoost.value + velocity.value / 3, 0, 100)))
const displayVelocity = computed(() => velocity.value.toFixed(0).padStart(2, '0'))

onMounted(() => {
  centerPointer()
  autoMotionStart = Date.now()
  segments.forEach((segment) => {
    segment.x = pointer.x
    segment.y = pointer.y
  })
  updateSnake()
  window.addEventListener('resize', centerPointer)
})

onBeforeUnmount(() => {
  window.cancelAnimationFrame(rafId)
  window.removeEventListener('resize', centerPointer)
})
</script>

<style scoped lang="scss">
.cyber-snake {
  --panel-border: rgba(122, 247, 255, 0.28);
  position: relative;
  width: 100vw;
  min-height: 100vh;
  overflow: hidden;
  background:
    radial-gradient(circle at 20% 20%, rgba(42, 194, 255, 0.22), transparent 34%),
    radial-gradient(circle at 80% 18%, rgba(98, 84, 255, 0.18), transparent 26%),
    radial-gradient(circle at 50% 78%, rgba(0, 255, 187, 0.16), transparent 28%),
    linear-gradient(135deg, #04101d 0%, #071827 30%, #02070d 100%);
  cursor: crosshair;
}

.cyber-snake__bg,
.cyber-snake__grid,
.cyber-snake__aurora,
.cyber-snake__stage {
  position: absolute;
  inset: 0;
}

.cyber-snake__bg {
  background:
    radial-gradient(circle at center, rgba(0, 255, 238, 0.08), transparent 42%),
    linear-gradient(90deg, rgba(0, 209, 255, 0.03) 1px, transparent 1px),
    linear-gradient(rgba(0, 209, 255, 0.03) 1px, transparent 1px);
  background-size: auto, 56px 56px, 56px 56px;
  filter: saturate(130%);
}

.cyber-snake__grid {
  background:
    repeating-linear-gradient(
      90deg,
      transparent 0,
      transparent 48px,
      rgba(102, 226, 255, 0.06) 49px,
      transparent 50px
    ),
    repeating-linear-gradient(
      180deg,
      transparent 0,
      transparent 48px,
      rgba(102, 226, 255, 0.06) 49px,
      transparent 50px
    );
  mask-image: linear-gradient(to bottom, rgba(0, 0, 0, 0.9), transparent 120%);
}

.cyber-snake__aurora {
  background:
    conic-gradient(from 180deg at 50% 50%, rgba(0, 255, 213, 0.18), transparent 30%, rgba(26, 120, 255, 0.2), transparent 65%, rgba(0, 255, 213, 0.18));
  filter: blur(90px);
  opacity: 0.65;
  animation: drift 16s linear infinite;
}

.cyber-snake__hud,
.cyber-snake__tips {
  position: absolute;
  z-index: 20;
  backdrop-filter: blur(18px);
  background: rgba(2, 12, 24, 0.52);
  border: 1px solid var(--panel-border);
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.04), 0 0 40px rgba(23, 209, 255, 0.15);
}

.cyber-snake__hud {
  top: 32px;
  left: 32px;
  width: min(420px, calc(100vw - 40px));
  padding: 24px 26px;
  border-radius: 24px;

  .eyebrow {
    margin: 0 0 8px;
    font-size: 12px;
    letter-spacing: 0.32em;
    text-transform: uppercase;
    color: rgba(140, 240, 255, 0.78);
  }

  h2 {
    margin: 0;
    font-size: clamp(34px, 4vw, 56px);
    line-height: 1;
    letter-spacing: 0.08em;
    color: #eaffff;
    text-shadow: 0 0 18px rgba(91, 234, 255, 0.38);
  }

  .desc {
    margin: 14px 0 0;
    font-size: 14px;
    line-height: 1.7;
    color: rgba(216, 245, 255, 0.78);
  }
}

.cyber-snake__tips {
  right: 32px;
  bottom: 30px;
  display: flex;
  gap: 18px;
  padding: 14px 18px;
  border-radius: 18px;
  color: rgba(223, 251, 255, 0.82);
  font-size: 12px;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.snake-segment {
  position: absolute;
  top: 0;
  left: 0;
  border-radius: 999px;
  background:
    radial-gradient(circle at 35% 32%, rgba(255, 255, 255, 0.92), rgba(255, 255, 255, 0.2) 18%, transparent 28%),
    linear-gradient(90deg, rgba(255, 255, 255, 0.28), rgba(255, 255, 255, 0.04)),
    linear-gradient(135deg, hsla(var(--segment-hue), 100%, 74%, 0.96), hsla(calc(var(--segment-hue) + 36), 90%, 52%, 0.88));
  box-shadow:
    0 0 var(--segment-glow) hsla(var(--segment-hue), 100%, 68%, 0.46),
    0 0 calc(var(--segment-glow) * 1.8) hsla(calc(var(--segment-hue) + 40), 100%, 50%, 0.18),
    inset 0 0 12px rgba(255, 255, 255, 0.4);
  transform-origin: center center;
  transition: width 0.25s ease, height 0.25s ease;

  &::after {
    content: '';
    position: absolute;
    inset: 22% 16%;
    border-radius: inherit;
    border: 1px solid rgba(255, 255, 255, 0.14);
    opacity: 0.65;
  }
}

.snake-segment--head {
  overflow: hidden;
  border-radius: 20% 46% 48% 20% / 36% 42% 42% 36%;
  clip-path: polygon(0% 18%, 28% 2%, 78% 18%, 100% 50%, 78% 82%, 28% 98%, 0% 82%, 16% 50%);
  box-shadow:
    0 0 calc(var(--segment-glow) * 1.3) hsla(var(--segment-hue), 100%, 70%, 0.72),
    0 0 calc(var(--segment-glow) * 2) rgba(95, 255, 239, 0.22),
    inset 0 0 18px rgba(255, 255, 255, 0.45);

  &::before {
    content: '';
    position: absolute;
    inset: 7% 4%;
    clip-path: polygon(0% 18%, 30% 4%, 77% 18%, 100% 50%, 77% 82%, 30% 96%, 0% 82%, 13% 50%);
    background: linear-gradient(135deg, rgba(255, 255, 255, 0.2), transparent 45%, rgba(255, 255, 255, 0.08));
    opacity: 0.45;
    pointer-events: none;
  }
}

.snake-head__face {
  position: absolute;
  inset: 0;
  filter: drop-shadow(0 0 10px rgba(52, 244, 255, 0.45));

  .snout-core {
    position: absolute;
    left: 74%;
    top: 50%;
    width: 18%;
    height: 18%;
    border-radius: 50% 50% 48% 48%;
    transform: translate(-50%, -50%);
    background: radial-gradient(circle, rgba(232, 253, 255, 0.95), rgba(0, 247, 255, 0.35) 58%, transparent 72%);
    box-shadow: 0 0 18px rgba(90, 255, 245, 0.42);
  }

  .eye {
    position: absolute;
    top: 34%;
    width: 11px;
    height: 16px;
    border-radius: 50% 50% 45% 45%;
    background: #f5ffff;
    box-shadow: 0 0 12px rgba(255, 255, 255, 0.8);
    transform: translate(-50%, -50%) rotate(8deg);

    &::after {
      content: '';
      position: absolute;
      left: 50%;
      top: 36%;
      width: 5px;
      height: 7px;
      border-radius: 50%;
      transform: translate(-50%, -50%);
      background: #00f7ff;
      box-shadow: 0 0 12px #00f7ff;
    }
  }

  .eye--left {
    left: 55%;
    transform: translate(-50%, -50%) rotate(-16deg);
  }

  .eye--right {
    left: 68%;
    transform: translate(-50%, -50%) rotate(14deg);
  }

  .jaw {
    position: absolute;
    left: 79%;
    width: 15%;
    height: 2px;
    transform: translateX(-50%);
    background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.85), transparent);
    box-shadow: 0 0 16px rgba(0, 255, 247, 0.8);
  }

  .jaw--top {
    top: 46%;
    transform: translate(-50%, -50%) rotate(12deg);
  }

  .jaw--bottom {
    top: 56%;
    transform: translate(-50%, -50%) rotate(-10deg);
  }

  .tongue {
    position: absolute;
    left: 90%;
    top: 51%;
    width: 36px;
    height: 14px;
    transform: translate(-8px, -50%);
    transform-origin: left center;
    animation: tongue-flick 1.8s ease-in-out infinite;
    opacity: 0.9;
  }

  .tongue-stem,
  .tongue-fork {
    position: absolute;
    display: block;
    background: linear-gradient(90deg, rgba(117, 255, 252, 0.18), rgba(111, 250, 255, 0.92));
    box-shadow: 0 0 12px rgba(82, 247, 255, 0.75);
  }

  .tongue-stem {
    left: 0;
    top: 50%;
    width: 24px;
    height: 2px;
    transform: translateY(-50%);
    border-radius: 999px;
  }

  .tongue-fork {
    right: 2px;
    width: 11px;
    height: 2px;
    transform-origin: left center;
    border-radius: 999px;
  }

  .tongue-fork--left {
    top: 4px;
    transform: rotate(-34deg);
  }

  .tongue-fork--right {
    bottom: 4px;
    transform: rotate(34deg);
  }
}

.pulse-burst {
  position: absolute;
  width: 26px;
  height: 26px;
  margin-left: -13px;
  margin-top: -13px;
  border-radius: 50%;
  pointer-events: none;
  border: 1px solid hsla(var(--burst-hue), 100%, 70%, 0.75);
  box-shadow:
    0 0 25px hsla(var(--burst-hue), 100%, 70%, 0.55),
    inset 0 0 24px hsla(var(--burst-hue), 100%, 74%, 0.35);
  animation: burst 0.9s ease-out forwards;

  &::before,
  &::after {
    content: '';
    position: absolute;
    inset: -1px;
    border-radius: inherit;
    border: inherit;
    opacity: 0.65;
  }

  &::before {
    animation: burst 0.9s ease-out 0.08s forwards;
  }

  &::after {
    animation: burst 0.9s ease-out 0.16s forwards;
  }
}

.signal-callout {
  position: absolute;
  z-index: 18;
  margin-left: -72px;
  margin-top: -18px;
  width: 144px;
  text-align: center;
  pointer-events: none;
  font-size: 18px;
  font-weight: 700;
  letter-spacing: 0.08em;
  color: #d8ffff;
  text-shadow:
    0 0 10px rgba(95, 255, 239, 0.95),
    0 0 24px rgba(95, 181, 255, 0.7);
  animation: signal-rise 1.4s ease-out forwards;
}

.target-reticle {
  position: absolute;
  width: 62px;
  height: 62px;
  margin-left: -31px;
  margin-top: -31px;
  border-radius: 50%;
  border: 1px solid rgba(123, 246, 255, 0.45);
  box-shadow: 0 0 20px rgba(70, 237, 255, 0.25), inset 0 0 14px rgba(90, 255, 237, 0.08);
  pointer-events: none;
  transition: opacity 0.25s ease;

  span {
    position: absolute;
    background: rgba(138, 246, 255, 0.78);
  }

  span:first-child {
    left: 50%;
    top: 10px;
    width: 1px;
    height: calc(100% - 20px);
    transform: translateX(-50%);
  }

  span:last-child {
    top: 50%;
    left: 10px;
    width: calc(100% - 20px);
    height: 1px;
    transform: translateY(-50%);
  }
}

@keyframes drift {
  0% {
    transform: rotate(0deg) scale(1);
  }
  50% {
    transform: rotate(180deg) scale(1.12);
  }
  100% {
    transform: rotate(360deg) scale(1);
  }
}

@keyframes burst {
  0% {
    transform: scale(0.3);
    opacity: 1;
  }
  100% {
    transform: scale(6.2);
    opacity: 0;
  }
}

@keyframes tongue-flick {
  0%,
  100% {
    transform: translate(-10px, -50%) scaleX(0.55);
    opacity: 0.2;
  }
  18% {
    transform: translate(-6px, -50%) scaleX(0.82);
    opacity: 0.55;
  }
  28% {
    transform: translate(0, -50%) scaleX(1.12);
    opacity: 1;
  }
  42% {
    transform: translate(-3px, -50%) scaleX(0.92);
    opacity: 0.8;
  }
  58% {
    transform: translate(-10px, -50%) scaleX(0.4);
    opacity: 0.1;
  }
}

@keyframes signal-rise {
  0% {
    transform: translate3d(0, 8px, 0) scale(0.82);
    opacity: 0;
    filter: blur(6px);
  }
  18% {
    opacity: 1;
    filter: blur(0);
  }
  100% {
    transform: translate3d(0, -46px, 0) scale(1.06);
    opacity: 0;
    filter: blur(2px);
  }
}

@media (max-width: 768px) {
  .cyber-snake__hud {
    top: 18px;
    left: 18px;
    right: 18px;
    width: auto;
    padding: 18px;
  }

  .cyber-snake__tips {
    left: 18px;
    right: 18px;
    bottom: 18px;
    justify-content: space-between;
    gap: 10px;
    flex-wrap: wrap;
  }
}
</style>

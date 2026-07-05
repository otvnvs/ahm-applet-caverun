<template>
  <div class="a" @touchstart.prevent="ts" @touchend.prevent="te" @mousedown.prevent="ts" @mouseup.prevent="te">
    <header class="b">
      <h1>Cave Run</h1>
      <p>Score: <b>{{ s }}</b> | Best: <b>{{ h }}</b></p>
    </header>

    <main class="c">
      <div v-if="o" class="o" @click.stop="r">
        <h2>SYSTEM CRASH</h2>
        <p>Tap inside grid to reboot thrusters</p>
      </div>
      <canvas ref="g" width="320" height="380"></canvas>
    </main>

    <footer class="f">
      <p class="h">{{ u ? 'THRUSTERS ACTIVE' : 'HOLD SCREEN TO FLY UP' }}</p>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const s = ref(0), h = ref(0), o = ref(false), g = ref(null), u = ref(false)
let cn, ctx, t, py = 190, pv = 0, cv = [], au = null, count = 0

const initAudio = () => {
  if (!au) au = new (window.AudioContext || window.webkitAudioContext)()
}

const playSound = (type) => {
  if (!au) return
  const osc = au.createOscillator()
  const gain = au.createGain()
  osc.connect(gain)
  gain.connect(au.destination)

  if (type === 'thrust') {
    osc.type = 'sine'
    osc.frequency.setValueAtTime(120, au.currentTime)
    gain.gain.setValueAtTime(0.03, au.currentTime)
    gain.gain.linearRampToValueAtTime(0.01, au.currentTime + 0.05)
    osc.start(); osc.stop(au.currentTime + 0.05)
  } else if (type === 'boom') {
    osc.type = 'sawtooth'
    osc.frequency.setValueAtTime(100, au.currentTime)
    osc.frequency.linearRampToValueAtTime(10, au.currentTime + 0.4)
    gain.gain.setValueAtTime(0.3, au.currentTime)
    gain.gain.linearRampToValueAtTime(0.01, au.currentTime + 0.4)
    osc.start(); osc.stop(au.currentTime + 0.4)
  }
}

const r = () => {
  initAudio()
  s.value = 0; o.value = false; py = 190; pv = 0; cv = []; count = 0
  // Build initial safe starting tunnel cavern structures
  for (let i = 0; i < 34; i++) {
    cv.push({ x: i * 10, top: 40, bot: 340 })
  }
}

const ts = () => { initAudio(); if (!o.value) u.value = true }
const te = () => { u.value = false }

const tick = () => {
  if (o.value) return
  ctx.fillStyle = '#111112'
  ctx.fillRect(0, 0, 320, 380)

  // Fluid physics engine calculation loops
  if (u.value) {
    pv -= 0.35
    if (count % 3 === 0) playSound('thrust')
  } else {
    pv += 0.30
  }
  
  // Constrain limits
  pv = Math.max(-5, Math.min(5, pv))
  py += pv
  count++

  // Shift column points and add procedural variations
  cv.forEach(p => { p.x -= 2 })
  if (cv[0].x < -10) {
    cv.shift()
    s.value += 1
    const last = cv[cv.length - 1]
    
    // Smooth random terrain step parameters matching bounds limits
    let change = Math.floor(Math.random() * 7) - 3
    let nextTop = Math.max(10, Math.min(120, last.top + change))
    let nextBot = Math.max(260, Math.min(370, last.bot + change))
    
    // Ensure strict minimal passage gap spacing width rules
    if (nextBot - nextTop < 130) {
      nextTop -= 10
      nextBot += 10
    }
    
    cv.push({ x: 320, top: nextTop, bot: nextBot })
  }

  // Draw structural cavern contours line segments
  ctx.fillStyle = '#1c1c1e'
  ctx.beginPath()
  ctx.moveTo(cv[0].x, 0)
  cv.forEach(p => ctx.lineTo(p.x, p.top))
  ctx.lineTo(320, 0)
  ctx.fill()

  ctx.beginPath()
  ctx.moveTo(cv[0].x, 380)
  cv.forEach(p => ctx.lineTo(p.x, p.bot))
  ctx.lineTo(320, 380)
  ctx.fill()

  // Track overlapping vector coordinates bounding collisions profiles
  const targetColumn = cv.find(p => p.x >= 40 && p.x <= 60)
  if (targetColumn) {
    if (py < targetColumn.top || (py + 12) > targetColumn.bot) {
      o.value = true
      playSound('boom')
      if (s.value > h.value) h.value = s.value
    }
  }

  // Draw core minimalist user flight square node block asset
  ctx.fillStyle = u.value ? '#fff' : '#3a3a3c'
  ctx.fillRect(50, py, 12, 12)
}

onMounted(() => {
  cn = g.value; ctx = cn.getContext('2d')
  r(); t = setInterval(tick, 1000 / 60)
  
  // Desktop keyboard spacebar listeners binding configurations
  window.addEventListener('keydown', e => { if (e.key === ' ') ts() })
  window.addEventListener('keyup', e => { if (e.key === ' ') te() })
})
onUnmounted(() => clearInterval(t))
</script>

<style scoped>
.a { width: 100vw; height: 100vh; background: #000; color: #fff; display: flex; flex-direction: column; align-items: center; justify-content: space-between; padding: 24px; font-family: sans-serif; box-sizing: border-box; user-select: none; overflow: hidden; }
.b { text-align: center; margin-bottom: 5px; width: 100%; }
h1 { margin: 0; font-size: 20px; text-transform: uppercase; color: #e2e8f0; letter-spacing: 1.5px; }
p { margin: 4px 0; color: #71717a; font-size: 13px; }
b { color: #fff; }
.c { width: 100%; max-width: 320px; aspect-ratio: 320 / 380; position: relative; border: 2px solid #1c1c1e; box-shadow: 0 20px 40px rgba(0,0,0,0.6); background: #111112; cursor: pointer; }
canvas { display: block; width: 100%; height: 100%; }
.o { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.95); display: flex; flex-direction: column; align-items: center; justify-content: center; z-index: 10; }
.o h2 { color: #ef4444; margin: 0; font-size: 20px; letter-spacing: 1px; text-align: center; }
.o p { color: #71717a; font-size: 12px; margin-top: 6px; }
.f { width: 100%; text-align: center; padding-top: 10px; }
.h { font-size: 11px; text-transform: uppercase; letter-spacing: 0.5px; color: #3a3a3c; font-weight: 700; margin: 0; }
@media (max-width: 360px) { .c { max-width: 90vw; } }
</style>

<script setup>
import { ref, watch, onMounted, onBeforeUnmount } from 'vue'

const props = defineProps({
  digit: String,
})

const canvas = ref(null)
const width = 200
const height = 300
const FONT_SIZE = 180
const PARTICLE_RADIUS = 1.5

let ctx = null

class Particle {
  constructor(x, y) {
    this.x = x
    this.y = y
  }
  draw(ctx) {
    ctx.beginPath()
    ctx.arc(this.x, this.y, PARTICLE_RADIUS, 0, Math.PI * 2)
    ctx.fill()
  }
}

// 轮廓波纹粒子，每个粒子存自己的起点(x,y)，方向(normalX, normalY)，扩散半径(r)，透明度(alpha)
class RippleParticle {
  constructor(x, y, normalX, normalY) {
    this.baseX = x
    this.baseY = y
    this.normalX = normalX
    this.normalY = normalY
    this.radius = 0
    this.alpha = 1
  }
  update() {
    this.radius += 1.5
    this.alpha -= 0.02
  }
  draw(ctx) {
    if (this.alpha <= 0) return
    const x = this.baseX + this.normalX * this.radius
    const y = this.baseY + this.normalY * this.radius
    ctx.beginPath()
    ctx.fillStyle = `rgba(255,255,255,${this.alpha.toFixed(2)})`
    ctx.arc(x, y, PARTICLE_RADIUS, 0, Math.PI * 2)
    ctx.fill()
  }
}

let fixedParticles = []
let rippleParticles = []
let animationId = null
let currentDigit = null

function sampleNumberPositions(ctx, char) {
  ctx.clearRect(0, 0, width, height)
  ctx.font = `bold ${FONT_SIZE}px monospace`
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillStyle = 'white'
  ctx.fillText(char, width / 2, height / 2 + FONT_SIZE * 0.15)

  const imgData = ctx.getImageData(0, 0, width, height)
  const points = []

  for (let y = 0; y < height; y += 3) {
    for (let x = 0; x < width; x += 3) {
      const idx = (y * width + x) * 4
      const alpha = imgData.data[idx + 3]
      if (alpha > 128) {
        const jitterX = x + (Math.random() - 0.5) * 3
        const jitterY = y + (Math.random() - 0.5) * 3
        points.push(new Particle(jitterX, jitterY))
      }
    }
  }
  return points
}

// 从数字内部粒子中，找出轮廓点（边缘粒子）
// 简单边缘检测：判断该点周围是否有透明点，有则是轮廓点
function findContourPoints(imgData, width, height, step = 3) {
  const contourPoints = []
  for (let y = step; y < height - step; y += step) {
    for (let x = step; x < width - step; x += step) {
      const idx = (y * width + x) * 4 + 3
      const alpha = imgData.data[idx]
      if (alpha > 128) {
        // 检查上下左右像素是否有透明（alpha<=128）
        const neighbors = [
          ((y - step) * width + x) * 4 + 3,
          ((y + step) * width + x) * 4 + 3,
          (y * width + (x - step)) * 4 + 3,
          (y * width + (x + step)) * 4 + 3,
        ]
        if (neighbors.some(nidx => imgData.data[nidx] <= 128)) {
          contourPoints.push({ x, y })
        }
      }
    }
  }
  return contourPoints
}

// 计算某点法线（近似）：取轮廓点上下左右透明距离差，得出法线方向（向外）
// 这里简单用上下左右透明像素差来估算法线方向，归一化
function calcNormal(imgData, width, height, x, y, step = 3) {
  function alphaAt(xx, yy) {
    if (xx < 0 || xx >= width || yy < 0 || yy >= height) return 0
    return imgData.data[(yy * width + xx) * 4 + 3]
  }
  const leftAlpha = alphaAt(x - step, y)
  const rightAlpha = alphaAt(x + step, y)
  const upAlpha = alphaAt(x, y - step)
  const downAlpha = alphaAt(x, y + step)

  // 水平方向，透明差代表边缘方向
  let nx = leftAlpha - rightAlpha
  let ny = upAlpha - downAlpha

  // 归一化向量
  const len = Math.sqrt(nx * nx + ny * ny)
  if (len === 0) return { nx: 0, ny: 0 }
  return { nx: nx / len, ny: ny / len }
}

function createRippleParticles(imgData, contourPoints) {
  rippleParticles = []
  contourPoints.forEach(({ x, y }) => {
    const { nx, ny } = calcNormal(imgData, width, height, x, y)
    if (nx !== 0 || ny !== 0) {
      rippleParticles.push(new RippleParticle(x, y, nx, ny))
    }
  })
}

function draw() {
  ctx.clearRect(0, 0, width, height)
  ctx.fillStyle = 'white'

  fixedParticles.forEach(p => p.draw(ctx))
  rippleParticles.forEach(rp => rp.draw(ctx))
}

function animate() {
  rippleParticles.forEach(rp => rp.update())
  rippleParticles = rippleParticles.filter(rp => rp.alpha > 0)
  draw()
  if (rippleParticles.length > 0) {
    animationId = requestAnimationFrame(animate)
  } else {
    animationId = null
  }
}

function redraw(char) {
  ctx.clearRect(0, 0, width, height)
  ctx.font = `bold ${FONT_SIZE}px monospace`
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillStyle = 'white'
  ctx.fillText(char, width / 2, height / 2 + FONT_SIZE * 0.15)

  const imgData = ctx.getImageData(0, 0, width, height)
  fixedParticles = []

  for (let y = 0; y < height; y += 3) {
    for (let x = 0; x < width; x += 3) {
      const idx = (y * width + x) * 4 + 3
      if (imgData.data[idx] > 128) {
        const jitterX = x + (Math.random() - 0.5) * 3
        const jitterY = y + (Math.random() - 0.5) * 3
        fixedParticles.push(new Particle(jitterX, jitterY))
      }
    }
  }

  // 找轮廓点，创建涟漪粒子
  const contourPoints = findContourPoints(imgData, width, height)
  createRippleParticles(imgData, contourPoints)

  draw()
}

onMounted(() => {
  const cvs = canvas.value
  ctx = cvs.getContext('2d')
  cvs.width = width
  cvs.height = height
  currentDigit = props.digit
  redraw(currentDigit)
})

watch(() => props.digit, (newVal) => {
  if (!ctx || !newVal) return
  if (newVal !== currentDigit) {
    currentDigit = newVal
    redraw(newVal)
    if (!animationId) {
      animationId = requestAnimationFrame(animate)
    }
  }
})

onBeforeUnmount(() => {
  if (animationId) cancelAnimationFrame(animationId)
})
</script>

<template>
  <canvas ref="canvas" class="particle-canvas"></canvas>
</template>

<style scoped>
.particle-canvas {
  display: block;
  background: transparent;
  width: 200px;
  height: 300px;
}
</style>

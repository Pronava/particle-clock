<template>
  <div
    ref="container"
    class="ripple-container"
    @mousemove="onMouseMove"
    @click="onClick"
  >
    <canvas ref="glCanvas" class="gl-canvas"></canvas>
    <canvas ref="contentCanvas" class="content-canvas" style="display:none;"></canvas>

    <!-- DOM 结构可交互元素 -->
    <div class="interactive-ui">
      <input type="text" placeholder="输入点什么..." />
      <button>按钮</button>
    </div>

    <!-- 用于截图的 DOM -->
    <div ref="contentDom" class="content-dom" aria-hidden="true">
      <h2>水波涟漪扭曲内容示范</h2>
      <input placeholder="输入点什么..." />
      <p>鼠标移动和点击会产生水波折射效果</p>
      <img src="https://picsum.photos/300/150" alt="示例图片" />
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'

const container = ref(null)
const glCanvas = ref(null)
const contentCanvas = ref(null)
const contentDom = ref(null)

let gl, program, animationFrameId
let startTime = 0
const MAX_RIPPLES = 30
const DURATION = 1.5
let ripples = []

const vertexShaderSource = `
attribute vec2 a_position;
varying vec2 v_uv;
void main() {
  v_uv = a_position * 0.5 + 0.5;
  gl_Position = vec4(a_position, 0, 1);
}
`

const fragmentShaderSource = `
precision highp float;
uniform float u_time;
uniform vec2 u_resolution;
uniform sampler2D u_contentTex;
uniform int u_rippleCount;
uniform vec2 u_ripples[${MAX_RIPPLES}];
uniform float u_startTimes[${MAX_RIPPLES}];
uniform float u_strengths[${MAX_RIPPLES}];
varying vec2 v_uv;

float ripple(vec2 uv, vec2 center, float progress, float strength, float aspect) {
  if(progress < 0.0 || progress > 1.0) return 0.0;
  float maxRadius = 0.4;
  float radius = progress * maxRadius;
  vec2 diff = uv - center;
  diff.x /= aspect;
  float dist = length(diff);
  float width = 0.02;
  float diffDist = dist - radius;
  float wave = exp(-diffDist * diffDist / (width * width));
  float amplitude = strength * pow(1.0 - progress, 3.0);
  return wave * amplitude;
}

void main() {
  vec2 uv = v_uv;
  float aspect = u_resolution.x / u_resolution.y;
  uv.x *= aspect;

  float totalDistortion = 0.0;
  for(int i=0; i<${MAX_RIPPLES}; i++) {
    if(i >= u_rippleCount) break;
    float progress = u_time - u_startTimes[i];
    if(progress >= 0.0 && progress <= 1.0) {
      totalDistortion += ripple(uv, u_ripples[i] * vec2(aspect, 1.0), progress, u_strengths[i], aspect);
    }
  }

  vec2 displacedUV = v_uv + vec2(totalDistortion * 0.03);
  vec4 color = texture2D(u_contentTex, displacedUV);
  gl_FragColor = color;
}
`

function createShader(gl, type, source) {
  const shader = gl.createShader(type)
  gl.shaderSource(shader, source)
  gl.compileShader(shader)
  if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
    console.error(gl.getShaderInfoLog(shader))
    return null
  }
  return shader
}

function createProgram(gl, vs, fs) {
  const program = gl.createProgram()
  gl.attachShader(program, vs)
  gl.attachShader(program, fs)
  gl.linkProgram(program)
  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    console.error(gl.getProgramInfoLog(program))
    return null
  }
  return program
}

let positionBuffer, positionLoc
let u_time, u_resolution, u_rippleCount
let u_ripples = [], u_startTimes = [], u_strengths = []
let u_contentTex
let contentTexture

function setupWebGL(canvasEl) {
  gl = canvasEl.getContext('webgl')
  const vs = createShader(gl, gl.VERTEX_SHADER, vertexShaderSource)
  const fs = createShader(gl, gl.FRAGMENT_SHADER, fragmentShaderSource)
  program = createProgram(gl, vs, fs)
  gl.useProgram(program)

  positionBuffer = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer)
  gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([
    -1, -1, 1, -1, -1, 1,
    -1, 1, 1, -1, 1, 1
  ]), gl.STATIC_DRAW)

  positionLoc = gl.getAttribLocation(program, 'a_position')
  gl.enableVertexAttribArray(positionLoc)
  gl.vertexAttribPointer(positionLoc, 2, gl.FLOAT, false, 0, 0)

  u_time = gl.getUniformLocation(program, 'u_time')
  u_resolution = gl.getUniformLocation(program, 'u_resolution')
  u_rippleCount = gl.getUniformLocation(program, 'u_rippleCount')

  for (let i = 0; i < MAX_RIPPLES; i++) {
    u_ripples.push(gl.getUniformLocation(program, `u_ripples[${i}]`))
    u_startTimes.push(gl.getUniformLocation(program, `u_startTimes[${i}]`))
    u_strengths.push(gl.getUniformLocation(program, `u_strengths[${i}]`))
  }

  u_contentTex = gl.getUniformLocation(program, 'u_contentTex')
  contentTexture = gl.createTexture()
  gl.bindTexture(gl.TEXTURE_2D, contentTexture)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE)
}

function drawContentToCanvas() {
  const ctx = contentCanvas.value.getContext('2d')
  const w = contentCanvas.value.width
  const h = contentCanvas.value.height
  ctx.clearRect(0, 0, w, h)

  ctx.save()
  ctx.scale(1, -1)
  ctx.translate(0, -h)

  const centerX = w / 2

  ctx.fillStyle = 'white'
  ctx.font = 'bold 28px sans-serif'
  const title = '水波涟漪扭曲内容示范'
  const titleWidth = ctx.measureText(title).width
  ctx.fillText(title, centerX - titleWidth / 2, 50)

  ctx.font = '20px sans-serif'
  const inputText = '输入点什么...'
  const inputWidth = ctx.measureText(inputText).width
  ctx.fillText(inputText, centerX - inputWidth / 2, 100)

  const desc = '鼠标移动和点击会产生水波折射效果'
  const descWidth = ctx.measureText(desc).width
  ctx.fillText(desc, centerX - descWidth / 2, 140)

  const img = new Image()
  img.crossOrigin = 'anonymous'
  img.src = 'https://picsum.photos/300/150'
  img.onload = () => {
    ctx.drawImage(img, centerX - 150, 160, 300, 150)
    ctx.restore()
    updateTexture()
  }
}

function updateTexture() {
  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, contentTexture)
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, contentCanvas.value)
  gl.uniform1i(u_contentTex, 0)
}

function render(time) {
  if (!gl) return
  time /= 1000
  if (!startTime) startTime = time
  const elapsed = time - startTime

  ripples = ripples.filter(r => elapsed - r.startTime < DURATION)

  gl.viewport(0, 0, gl.canvas.width, gl.canvas.height)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  gl.uniform1f(u_time, elapsed)
  gl.uniform2f(u_resolution, gl.canvas.width, gl.canvas.height)
  gl.uniform1i(u_rippleCount, ripples.length)

  for (let i = 0; i < MAX_RIPPLES; i++) {
    if (i < ripples.length) {
      const r = ripples[i]
      gl.uniform2f(u_ripples[i], r.x, r.y)
      gl.uniform1f(u_startTimes[i], r.startTime)
      gl.uniform1f(u_strengths[i], r.strength)
    } else {
      gl.uniform2f(u_ripples[i], 0, 0)
      gl.uniform1f(u_startTimes[i], 0)
      gl.uniform1f(u_strengths[i], 0)
    }
  }

  gl.drawArrays(gl.TRIANGLES, 0, 6)
  animationFrameId = requestAnimationFrame(render)
}

function addRipple(x, y, strength) {
  if (ripples.length >= MAX_RIPPLES) ripples.shift()
  ripples.push({ x, y, startTime: performance.now() / 1000 - startTime, strength })
}

function onMouseMove(e) {
  const rect = glCanvas.value.getBoundingClientRect()
  const x = (e.clientX - rect.left) / rect.width
  const y = 1 - (e.clientY - rect.top) / rect.height
  addRipple(x, y, 0.4)
}

function onClick(e) {
  const rect = glCanvas.value.getBoundingClientRect()
  const x = (e.clientX - rect.left) / rect.width
  const y = 1 - (e.clientY - rect.top) / rect.height
  addRipple(x, y, 1.0)
}

function resize() {
  const cw = container.value.clientWidth
  const ch = 400
  glCanvas.value.width = cw * devicePixelRatio
  glCanvas.value.height = ch * devicePixelRatio
  glCanvas.value.style.width = cw + 'px'
  glCanvas.value.style.height = ch + 'px'

  contentCanvas.value.width = cw * devicePixelRatio
  contentCanvas.value.height = ch * devicePixelRatio
  drawContentToCanvas()
  updateTexture()
}

onMounted(async () => {
  setupWebGL(glCanvas.value)
  await nextTick()
  resize()
  window.addEventListener('resize', resize)
  animationFrameId = requestAnimationFrame(render)
})

onBeforeUnmount(() => {
  if (animationFrameId) cancelAnimationFrame(animationFrameId)
  window.removeEventListener('resize', resize)
})
</script>

<style scoped>
.ripple-container {
  position: relative;
  width: 100%;
  height: 400px;
  overflow: hidden;
  background: black;
}

.gl-canvas {
  position: absolute;
  top: 0; left: 0;
  width: 100%;
  height: 100%;
  z-index: 0;
  pointer-events: none;
}

.interactive-ui {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 2;
  display: flex;
  gap: 12px;
}

input, button {
  padding: 6px 12px;
  font-size: 16px;
}

.content-dom {
  display: none;
}
</style>

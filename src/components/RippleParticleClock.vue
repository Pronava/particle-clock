<template>
  <canvas ref="canvas" style="width: 600px; height: 150px; background: black;"></canvas>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

const canvas = ref(null)
let gl = null
let program = null
let animationFrameId = null
let startTime = 0

let currentTimeStr = '00:00:00'

const MAX_RIPPLES = 10
const rippleDuration = 1.2

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
uniform sampler2D u_textTex;

uniform int u_rippleCount;
uniform vec2 u_ripples[${MAX_RIPPLES}];
uniform float u_startTimes[${MAX_RIPPLES}];

varying vec2 v_uv;

// 单个波纹函数，宽度和振幅可参数化
float singleRipple(vec2 uv, vec2 center, float progress, float aspect, float maxRadius, float minWidth, float maxWidth) {
  if (progress <= 0.0 || progress >= 1.0) return 0.0;

  vec2 diffVec = uv - center;
  diffVec.x /= aspect;
  float dist = length(diffVec);

  float width = mix(minWidth, maxWidth, progress);

  float radius = progress * maxRadius;
  float diff = dist - radius;

  float rippleValue = exp(- (diff * diff) / (width * width));
  float amplitude = pow(1.0 - progress, 3.0);

  return rippleValue * amplitude;
}

// 主波纹 + 两个细波纹环绕主波前后
float ripple(vec2 uv, vec2 center, float progress, float aspect) {
  float mainMaxRadius = 1.0;

  // 主波纹，宽度较大
  float main = singleRipple(uv, center, progress, aspect, mainMaxRadius, 0.01, 0.06);

  // 细波纹半径围绕主波半径前后环绕
  float smallMaxRadius = 0.85 * mainMaxRadius;
  float largeMaxRadius = 1.15 * mainMaxRadius;

  float fineAmplitude = 0.25;

  float smallRipple = singleRipple(uv, center, progress, aspect, smallMaxRadius, 0.003, 0.02) * fineAmplitude;
  float largeRipple = singleRipple(uv, center, progress, aspect, largeMaxRadius, 0.003, 0.02) * fineAmplitude;

  return main + smallRipple + largeRipple;
}

void main() {
  vec2 uv = v_uv;

  float aspect = u_resolution.x / u_resolution.y;
  uv.x *= aspect;

  float distortion = 0.0;

  float speedFactor = 0.5;

  for (int i = 0; i < ${MAX_RIPPLES}; i++) {
    if(i >= u_rippleCount) break;

    float rawProgress = (u_time - u_startTimes[i]) * speedFactor;
    if(rawProgress > 0.0 && rawProgress < 1.0) {
      vec2 rippleCenter = u_ripples[i] * vec2(aspect, 1.0);
      distortion += ripple(uv, rippleCenter, rawProgress, aspect);
    }
  }

  vec2 distortedUV = v_uv + vec2(distortion * 0.05, distortion * 0.05);

  vec4 texColor = texture2D(u_textTex, distortedUV);
  float brightness = dot(texColor.rgb, vec3(0.299, 0.587, 0.114));

  vec3 rippleColor = vec3(1.0, 0.7, 0.4) * distortion * step(0.3, brightness);

  vec3 color = texColor.rgb + rippleColor;

  gl_FragColor = vec4(clamp(color, 0.0, 1.0), 1.0);
}
`

function createShader(gl, type, source) {
  const shader = gl.createShader(type)
  gl.shaderSource(shader, source)
  gl.compileShader(shader)
  if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
    console.error('Shader compile failed:', gl.getShaderInfoLog(shader))
    gl.deleteShader(shader)
    return null
  }
  return shader
}

function createProgram(gl, vShader, fShader) {
  const prog = gl.createProgram()
  gl.attachShader(prog, vShader)
  gl.attachShader(prog, fShader)
  gl.linkProgram(prog)
  if (!gl.getProgramParameter(prog, gl.LINK_STATUS)) {
    console.error('Program link failed:', gl.getProgramInfoLog(prog))
    gl.deleteProgram(prog)
    return null
  }
  return prog
}

let textTexture = null

function createTextTexture(gl, timeStr, width, height) {
  const textCanvas = document.createElement('canvas')
  textCanvas.width = width
  textCanvas.height = height
  const ctx = textCanvas.getContext('2d')

  ctx.clearRect(0, 0, width, height)

  ctx.save()
  ctx.scale(1, -1)
  ctx.translate(0, -height)

  ctx.fillStyle = 'black'
  ctx.fillRect(0, 0, width, height)

  ctx.fillStyle = 'white'
  ctx.font = `${height * 0.75}px monospace`
  ctx.textBaseline = 'middle'
  ctx.textAlign = 'center'

  for (let i = 0; i < 8; i++) {
    const x = (i + 0.5) * (width / 8)
    ctx.fillText(timeStr[i], x, height / 2)
  }

  ctx.restore()

  if (!textTexture) textTexture = gl.createTexture()
  gl.bindTexture(gl.TEXTURE_2D, textTexture)
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, textCanvas)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR)

  return textTexture
}

function setupWebGL(canvasEl) {
  gl = canvasEl.getContext('webgl')
  if (!gl) {
    alert('WebGL not supported')
    return
  }

  const vertexShader = createShader(gl, gl.VERTEX_SHADER, vertexShaderSource)
  const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fragmentShaderSource)
  program = createProgram(gl, vertexShader, fragmentShader)
  gl.useProgram(program)

  const positionBuffer = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer)
  const positions = new Float32Array([
    -1, -1,
    1, -1,
    -1, 1,
    -1, 1,
    1, -1,
    1, 1,
  ])
  gl.bufferData(gl.ARRAY_BUFFER, positions, gl.STATIC_DRAW)
  const positionLoc = gl.getAttribLocation(program, 'a_position')
  gl.enableVertexAttribArray(positionLoc)
  gl.vertexAttribPointer(positionLoc, 2, gl.FLOAT, false, 0, 0)

  program.u_time = gl.getUniformLocation(program, 'u_time')
  program.u_resolution = gl.getUniformLocation(program, 'u_resolution')
  program.u_textTex = gl.getUniformLocation(program, 'u_textTex')
  program.u_rippleCount = gl.getUniformLocation(program, 'u_rippleCount')
  program.u_ripples = []
  program.u_startTimes = []
  for (let i = 0; i < MAX_RIPPLES; i++) {
    program.u_ripples.push(gl.getUniformLocation(program, `u_ripples[${i}]`))
    program.u_startTimes.push(gl.getUniformLocation(program, `u_startTimes[${i}]`))
  }
}

function render(time) {
  if (!gl) return
  time /= 1000
  if (!startTime) startTime = time
  const elapsed = time - startTime

  ripples = ripples.filter(r => elapsed - r.startTime < rippleDuration)

  gl.viewport(0, 0, gl.canvas.width, gl.canvas.height)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  gl.uniform1f(program.u_time, elapsed)
  gl.uniform2f(program.u_resolution, gl.canvas.width, gl.canvas.height)

  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textTexture)
  gl.uniform1i(program.u_textTex, 0)

  gl.uniform1i(program.u_rippleCount, ripples.length)

  for (let i = 0; i < MAX_RIPPLES; i++) {
    if (i < ripples.length) {
      gl.uniform2f(program.u_ripples[i], ripples[i].x, ripples[i].y)
      gl.uniform1f(program.u_startTimes[i], ripples[i].startTime)
    } else {
      gl.uniform2f(program.u_ripples[i], 0, 0)
      gl.uniform1f(program.u_startTimes[i], 0)
    }
  }

  gl.drawArrays(gl.TRIANGLES, 0, 6)

  animationFrameId = requestAnimationFrame(render)
}

function charIndexToUV(i) {
  return { x: (i + 0.5) / 8, y: 0.5 }
}

function checkTimeChange() {
  const now = new Date()
  const timeStr =
    String(now.getHours()).padStart(2, '0') + ':' +
    String(now.getMinutes()).padStart(2, '0') + ':' +
    String(now.getSeconds()).padStart(2, '0')

  if (timeStr !== currentTimeStr) {
    for (let i = 0; i < 8; i++) {
      if (timeStr[i] !== currentTimeStr[i]) {
        if (ripples.length >= MAX_RIPPLES) ripples.shift()
        const uv = charIndexToUV(i)
        ripples.push({
          x: uv.x,
          y: uv.y,
          startTime: performance.now() / 1000 - startTime,
        })
        break
      }
    }
    currentTimeStr = timeStr
    createTextTexture(gl, timeStr, gl.canvas.width, gl.canvas.height)
  }
}

onMounted(() => {
  const canvasEl = canvas.value
  canvasEl.width = canvasEl.clientWidth * devicePixelRatio
  canvasEl.height = canvasEl.clientHeight * devicePixelRatio

  setupWebGL(canvasEl)
  createTextTexture(gl, currentTimeStr, gl.canvas.width, gl.canvas.height)

  animationFrameId = requestAnimationFrame(render)

  setInterval(checkTimeChange, 1000)
})

onBeforeUnmount(() => {
  if (animationFrameId) cancelAnimationFrame(animationFrameId)
})
</script>

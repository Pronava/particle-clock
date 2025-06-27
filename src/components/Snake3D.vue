<template>
  <div class="wrapper">
    <div class="controls">
      <button @click="startGame" :disabled="running">开始游戏</button>
      <div>分数: {{ score }}</div>
      <div v-if="gameOver" class="game-over">游戏结束！请点击开始重玩</div>
      <div class="instructions">
        <p>控制说明：</p>
        <ul>
          <li>箭头 ← →：左右移动 (X 轴)</li>
          <li>箭头 ↑ ↓：前后移动 (Z 轴)</li>
          <li>Q：向上移动 (Y+)</li>
          <li>E：向下移动 (Y-)</li>
        </ul>
      </div>
    </div>
    <div ref="container" class="snake-container" tabindex="0" @keydown="handleKeydown"></div>
  </div>
</template>

<script setup>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { ref, reactive, onMounted, onBeforeUnmount } from 'vue'

const container = ref(null)

const GRID_SIZE = 30
const BLOCK_SIZE = 1
const MOVE_INTERVAL_MS = 300

let scene, camera, renderer, controls
let snakeGroup = null
let animationId = null
let moveInterval = null

const state = reactive({
  snake: {
    body: [],
    direction: new THREE.Vector3(1, 0, 0),
    grow: false,
  },
  food: null,
  running: false,
  score: 0,
  gameOver: false,
})

function initThree() {
  scene = new THREE.Scene()

  const width = container.value.clientWidth
  const height = container.value.clientHeight

  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000)
  camera.position.set(GRID_SIZE * 1.5, GRID_SIZE * 1.5, GRID_SIZE * 2)
  camera.lookAt(GRID_SIZE / 2, GRID_SIZE / 2, GRID_SIZE / 2)

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(width, height)
  renderer.shadowMap.enabled = true
  container.value.appendChild(renderer.domElement)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.target.set(GRID_SIZE / 2, GRID_SIZE / 2, GRID_SIZE / 2)
  controls.enableDamping = true
  controls.dampingFactor = 0.1
  controls.update()

  scene.add(new THREE.AmbientLight(0x606060))
  const dirLight = new THREE.DirectionalLight(0xffffff, 1)
  dirLight.position.set(1, 2, 1)
  scene.add(dirLight)

  // 坐标轴辅助线
  const axesHelper = new THREE.AxesHelper(GRID_SIZE * 1.5)
  scene.add(axesHelper)

  addGridBox()

  snakeGroup = new THREE.Group()
  scene.add(snakeGroup)
}

function addGridBox() {
  const boxGeometry = new THREE.BoxGeometry(
    GRID_SIZE * BLOCK_SIZE,
    GRID_SIZE * BLOCK_SIZE,
    GRID_SIZE * BLOCK_SIZE
  )
  const edges = new THREE.EdgesGeometry(boxGeometry)
  const lineMaterial = new THREE.LineBasicMaterial({ color: 0x888888, opacity: 0.5, transparent: true })
  const boxWireframe = new THREE.LineSegments(edges, lineMaterial)
  boxWireframe.position.set(
    (GRID_SIZE * BLOCK_SIZE) / 2 - BLOCK_SIZE / 2,
    (GRID_SIZE * BLOCK_SIZE) / 2 - BLOCK_SIZE / 2,
    (GRID_SIZE * BLOCK_SIZE) / 2 - BLOCK_SIZE / 2
  )
  scene.add(boxWireframe)
}

function createCube(x, y, z, color) {
  const geometry = new THREE.BoxGeometry(BLOCK_SIZE * 0.9, BLOCK_SIZE * 0.9, BLOCK_SIZE * 0.9)
  const material = new THREE.MeshStandardMaterial({ color })
  const cube = new THREE.Mesh(geometry, material)
  cube.position.set(x * BLOCK_SIZE, y * BLOCK_SIZE, z * BLOCK_SIZE)

  // 边框线
  const edgesGeometry = new THREE.EdgesGeometry(geometry)
  const edgesMaterial = new THREE.LineBasicMaterial({ color: 0x222222 })
  const edges = new THREE.LineSegments(edgesGeometry, edgesMaterial)
  cube.add(edges)

  return cube
}

function renderSnake() {
  snakeGroup.clear()

  // 蛇身
  state.snake.body.forEach((pos, idx) => {
    const color = idx === 0 ? 0x0066ff : 0x00cc00
    snakeGroup.add(createCube(pos.x, pos.y, pos.z, color))
  })

  // 食物
  if (state.food) {
    snakeGroup.add(createCube(state.food.x, state.food.y, state.food.z, 0xff3300))
  }
}

function generateFood() {
  let pos
  do {
    pos = new THREE.Vector3(
      Math.floor(Math.random() * GRID_SIZE),
      Math.floor(Math.random() * GRID_SIZE),
      Math.floor(Math.random() * GRID_SIZE)
    )
  } while (state.snake.body.some(s => s.equals(pos)))
  state.food = pos
}

function moveSnake() {
  if (!state.running) return

  const head = state.snake.body[0].clone()
  head.add(state.snake.direction)

  // 碰撞检测：撞墙
  if (
    head.x < 0 || head.x >= GRID_SIZE ||
    head.y < 0 || head.y >= GRID_SIZE ||
    head.z < 0 || head.z >= GRID_SIZE
  ) {
    gameOver()
    return
  }

  // 撞自己
  if (state.snake.body.some(pos => pos.equals(head))) {
    gameOver()
    return
  }

  state.snake.body.unshift(head)

  // 吃到食物
  if (state.food && head.equals(state.food)) {
    state.snake.grow = true
    state.score++
    generateFood()
  }

  if (!state.snake.grow) {
    state.snake.body.pop()
  } else {
    state.snake.grow = false
  }

  renderSnake()
}

function gameOver() {
  state.running = false
  state.gameOver = true
}

function resetGame() {
  state.snake.body = []
  const center = Math.floor(GRID_SIZE / 2)
  for (let i = 0; i < 4; i++) {
    // 初始蛇身体4格，头在中心，向X负方向排列
    state.snake.body.push(new THREE.Vector3(center - i, center, center))
  }
  state.snake.direction = new THREE.Vector3(1, 0, 0)
  state.snake.grow = false
  state.score = 0
  state.gameOver = false
  generateFood()
  renderSnake()
}

function startGame() {
  if(state.running) return
  resetGame()
  state.running = true
  if(moveInterval) clearInterval(moveInterval)
  moveInterval = setInterval(moveSnake, MOVE_INTERVAL_MS)
  container.value.focus()
}

function handleKeydown(event) {
  if(!state.running) return
  const key = event.key.toLowerCase()
  const dir = state.snake.direction.clone()

  function setDirection(newDir) {
    if (dir.clone().add(newDir).length() === 0) return
    state.snake.direction = newDir
  }

  switch (key) {
    case 'arrowleft':   // X-
      setDirection(new THREE.Vector3(-1, 0, 0))
      break
    case 'arrowright':  // X+
      setDirection(new THREE.Vector3(1, 0, 0))
      break
    case 'arrowup':     // Z-
      setDirection(new THREE.Vector3(0, 0, -1))
      break
    case 'arrowdown':   // Z+
      setDirection(new THREE.Vector3(0, 0, 1))
      break
    case 'q':           // Y+
      setDirection(new THREE.Vector3(0, 1, 0))
      break
    case 'e':           // Y-
      setDirection(new THREE.Vector3(0, -1, 0))
      break
  }
}

function animate() {
  animationId = requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}

function onResize() {
  if (!container.value) return
  const width = container.value.clientWidth
  const height = container.value.clientHeight
  camera.aspect = width / height
  camera.updateProjectionMatrix()
  renderer.setSize(width, height)
}

onMounted(() => {
  initThree()
  window.addEventListener('resize', onResize)
  animate()
  container.value.focus()
})

onBeforeUnmount(() => {
  if (moveInterval) clearInterval(moveInterval)
  if (animationId) cancelAnimationFrame(animationId)
  window.removeEventListener('resize', onResize)
})
</script>

<style scoped>
.wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px;
  padding: 15px;
  background: #111;
  color: #eee;
  height: 100vh;
  box-sizing: border-box;
}

.snake-container {
  width: 100%;
  max-width: 900px;
  height: 600px;
  background-color: #222;
  outline: none;
  user-select: none;
  border-radius: 8px;
  cursor: grab;
}

.controls {
  text-align: center;
  font-size: 16px;
}

.game-over {
  margin-top: 8px;
  color: #ff5555;
  font-weight: bold;
}

.instructions {
  margin-top: 12px;
  font-size: 14px;
  color: #ccc;
  max-width: 400px;
  margin-left: auto;
  margin-right: auto;
  text-align: left;
}
</style>

<template>
  <div class="game-wrapper">
    <div ref="container" class="tetris-container" tabindex="0" @keydown="handleKeydown"></div>
    <div class="info-panel">
      <h2>3D 俄罗斯方块 (Tetris 3D)</h2>
      <p>分数: {{ score }}</p>
      <p v-if="isGameOver" class="game-over">游戏结束！按 R 重启</p>
      <p>控制说明：</p>
      <ul>
        <li>← → (A D)：左右移动</li>
        <li>W S：前后移动</li>
        <li>↓ ：加速下落</li>
        <li>↑ ：绕Y轴旋转</li>
        <li>Q / E ：绕X轴旋转</li>
        <li>Z / C ：绕Z轴旋转</li>
        <li>空格 ：硬降</li>
        <li>R ：重启游戏</li>
        <li>鼠标拖动视角，滚轮缩放</li>
      </ul>
      
      <!-- 添加开始和暂停按钮 -->
      <div>
        <button @click="toggleGame">{{ isGamePaused ? '继续游戏' : '暂停游戏' }}</button>
        <button @click="startGame" v-if="isGameOver">开始新游戏</button>
      </div>
    </div>
  </div>
</template>


<script setup>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { ref, reactive, onMounted, onBeforeUnmount } from 'vue'

const container = ref(null)
const score = ref(0)
const isGameOver = ref(false)

// 游戏尺寸
const COLS = 5
const ROWS = 5
const HEIGHT = 20
const BLOCK_SIZE = 1

// 7种经典3D俄罗斯方块形状及对应颜色（3x3x3数组）
const SHAPES = [
  { // I形
    color: 0x00ffff,
    blocks: [
      [[0,0,0],[0,0,0],[0,0,0]],
      [[1,1,1],[0,0,0],[0,0,0]],
      [[0,0,0],[0,0,0],[0,0,0]],
    ]
  },
  { // O形 (正方体2x2x2)
    color: 0xffff00,
    blocks: [
      [[0,0,0],[0,0,0],[0,0,0]],
      [[0,1,1],[0,1,1],[0,0,0]],
      [[0,0,0],[0,0,0],[0,0,0]],
    ]
  },
  { // T形
    color: 0xff00ff,
    blocks: [
      [[0,0,0],[0,0,0],[0,0,0]],
      [[0,1,0],[1,1,1],[0,0,0]],
      [[0,0,0],[0,0,0],[0,0,0]],
    ]
  },
  { // L形
    color: 0xff7700,
    blocks: [
      [[0,0,0],[0,0,0],[0,0,0]],
      [[1,0,0],[1,1,1],[0,0,0]],
      [[0,0,0],[0,0,0],[0,0,0]],
    ]
  },
  { // J形
    color: 0x0000ff,
    blocks: [
      [[0,0,0],[0,0,0],[0,0,0]],
      [[0,0,1],[1,1,1],[0,0,0]],
      [[0,0,0],[0,0,0],[0,0,0]],
    ]
  },
  { // S形
    color: 0x00ff00,
    blocks: [
      [[0,0,0],[0,0,0],[0,0,0]],
      [[0,1,1],[1,1,0],[0,0,0]],
      [[0,0,0],[0,0,0],[0,0,0]],
    ]
  },
  { // Z形
    color: 0xff0000,
    blocks: [
      [[0,0,0],[0,0,0],[0,0,0]],
      [[1,1,0],[0,1,1],[0,0,0]],
      [[0,0,0],[0,0,0],[0,0,0]],
    ]
  },
]

// 3D数组初始化
function createEmptyBoard() {
  const board = []
  for(let x=0; x<COLS; x++){
    board[x] = []
    for(let y=0; y<HEIGHT; y++){
      board[x][y] = []
      for(let z=0; z<ROWS; z++){
        board[x][y][z] = null
      }
    }
  }
  return board
}

// 深拷贝形状数组
function cloneBlocks(blocks) {
  return blocks.map(layer => layer.map(row => [...row]))
}

// 旋转函数改正，确保旋转正确
function rotateX(blocks) {
  const size = blocks.length
  const newBlocks = Array(size).fill(0).map(() => Array(size).fill(0).map(() => Array(size).fill(0)))
  for(let y=0; y<size; y++){
    for(let x=0; x<size; x++){
      for(let z=0; z<size; z++){
        newBlocks[y][x][z] = blocks[size-1 - z][x][y]
      }
    }
  }
  return newBlocks
}
function rotateY(blocks) {
  const size = blocks.length
  const newBlocks = Array(size).fill(0).map(() => Array(size).fill(0).map(() => Array(size).fill(0)))
  for(let y=0; y<size; y++){
    for(let x=0; x<size; x++){
      for(let z=0; z<size; z++){
        newBlocks[y][x][z] = blocks[y][size-1 - z][x]
      }
    }
  }
  return newBlocks
}
function rotateZ(blocks) {
  const size = blocks.length
  const newBlocks = Array(size).fill(0).map(() => Array(size).fill(0).map(() => Array(size).fill(0)))
  for(let y=0; y<size; y++){
    for(let x=0; x<size; x++){
      for(let z=0; z<size; z++){
        newBlocks[y][x][z] = blocks[size-1 - x][y][z]
      }
    }
  }
  return newBlocks
}

// 判断能否放置
function canPlace(blocks, pos, board) {
  const size = blocks.length
  for(let y=0; y<size; y++){
    for(let x=0; x<size; x++){
      for(let z=0; z<size; z++){
        if(blocks[y][x][z]){
          const bx = pos.x + x
          const by = pos.y + y
          const bz = pos.z + z
          if(bx < 0 || bx >= COLS || by < 0 || by >= HEIGHT || bz < 0 || bz >= ROWS) return false
          if(board[bx][by][bz]) return false
        }
      }
    }
  }
  return true
}

// 固定方块到board
function placeBlocks(blocks, pos, board, color) {
  const size = blocks.length
  for(let y=0; y<size; y++){
    for(let x=0; x<size; x++){
      for(let z=0; z<size; z++){
        if(blocks[y][x][z]){
          const bx = pos.x + x
          const by = pos.y + y
          const bz = pos.z + z
          board[bx][by][bz] = color
        }
      }
    }
  }
}

// 清理满层
function clearFullLayers(board) {
  let cleared = 0
  for(let y=0; y<HEIGHT; y++){
    let full = true
    outer: for(let x=0; x<COLS; x++){
      for(let z=0; z<ROWS; z++){
        if(!board[x][y][z]){
          full = false
          break outer
        }
      }
    }
    if(full){
      for(let yy = y; yy < HEIGHT-1; yy++){
        for(let x=0; x<COLS; x++){
          for(let z=0; z<ROWS; z++){
            board[x][yy][z] = board[x][yy+1][z]
          }
        }
      }
      for(let x=0; x<COLS; x++){
        for(let z=0; z<ROWS; z++){
          board[x][HEIGHT-1][z] = null
        }
      }
      cleared++
      y--
    }
  }
  return cleared
}

const state = reactive({
  board: createEmptyBoard(),
  currentPiece: null,
  currentPos: {x: 0, y: 0, z: 0},
  currentColor: 0xffffff,
  isGameOver: false,
})

// Three.js 相关
let scene, camera, renderer, blocksGroup, controls
let animationId = null
let dropTimer = null

function initThree() {
  if(renderer){
    renderer.dispose()
    container.value.innerHTML = ''
  }
  scene = new THREE.Scene()
  const width = container.value.clientWidth
  const height = container.value.clientHeight
  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000)
  camera.position.set(COLS * 1.5, HEIGHT * 1.5, ROWS * 2)
  camera.lookAt(COLS / 2, HEIGHT / 2, ROWS / 2)

  renderer = new THREE.WebGLRenderer({antialias:true})
  renderer.setSize(width, height)
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  container.value.appendChild(renderer.domElement)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.target.set(COLS / 2, HEIGHT / 2, ROWS / 2)
  controls.enableDamping = true
  controls.dampingFactor = 0.1
  controls.update()


  scene.add(new THREE.AmbientLight(0x606060))
  
  const pointLight1 = new THREE.PointLight(0xffffff, 0.6)
  pointLight1.position.set(COLS, HEIGHT * 1.5, ROWS)
  pointLight1.castShadow = true
  scene.add(pointLight1)

  const pointLight2 = new THREE.PointLight(0xffffff, 0.4)
  pointLight2.position.set(-COLS, HEIGHT, -ROWS)
  pointLight2.castShadow = true
  scene.add(pointLight2)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(1, 2, 1)
  directionalLight.castShadow = true
  directionalLight.shadow.mapSize.width = 1024
  directionalLight.shadow.mapSize.height = 1024
  scene.add(directionalLight)

  // 地面阴影接收平面
  const floorGeometry = new THREE.PlaneGeometry(COLS * BLOCK_SIZE, ROWS * BLOCK_SIZE)
  const floorMaterial = new THREE.ShadowMaterial({opacity: 0.3})
  const floor = new THREE.Mesh(floorGeometry, floorMaterial)
  floor.rotation.x = -Math.PI / 2
  floor.position.set(
    (COLS * BLOCK_SIZE) / 2 - BLOCK_SIZE / 2,
    -0.5,
    (ROWS * BLOCK_SIZE) / 2 - BLOCK_SIZE / 2
  )
  floor.receiveShadow = true
  scene.add(floor)

  // 边框线框盒
  const boxGeometry = new THREE.BoxGeometry(
    COLS * BLOCK_SIZE,
    HEIGHT * BLOCK_SIZE,
    ROWS * BLOCK_SIZE
  )
  const edges = new THREE.EdgesGeometry(boxGeometry)
  const lineMaterial = new THREE.LineBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.2 })
  const boxWireframe = new THREE.LineSegments(edges, lineMaterial)
  boxWireframe.position.set(
    (COLS * BLOCK_SIZE) / 2 - BLOCK_SIZE / 2,
    (HEIGHT * BLOCK_SIZE) / 2 - BLOCK_SIZE / 2,
    (ROWS * BLOCK_SIZE) / 2 - BLOCK_SIZE / 2
  )
  scene.add(boxWireframe)

  addContainerGridLines()

  blocksGroup = new THREE.Group()
  scene.add(blocksGroup)
}

// 给容器6面添加网格线
function addContainerGridLines() {
  const gridColor = 0x444444
  const gridMaterial = new THREE.LineBasicMaterial({ color: gridColor, opacity: 0.3, transparent: true })

  const step = BLOCK_SIZE
  const gridSizeX = COLS * BLOCK_SIZE
  const gridSizeY = HEIGHT * BLOCK_SIZE
  const gridSizeZ = ROWS * BLOCK_SIZE

  const createGrid = (width, height, step, position, rotation) => {
    const geometry = new THREE.BufferGeometry()
    const vertices = []

    for (let i = 0; i <= height; i += step) {
      vertices.push(0, i, 0, width, i, 0)
    }
    for (let j = 0; j <= width; j += step) {
      vertices.push(j, 0, 0, j, height, 0)
    }

    geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3))
    const lines = new THREE.LineSegments(geometry, gridMaterial)
    lines.position.copy(position)
    lines.rotation.set(rotation.x, rotation.y, rotation.z)
    scene.add(lines)
  }

  // 底面
  createGrid(
    gridSizeX, gridSizeZ, step,
    new THREE.Vector3(0, -0.5, 0),
    new THREE.Euler(-Math.PI / 2, 0, 0)
  )

  // 顶面
  createGrid(
    gridSizeX, gridSizeZ, step,
    new THREE.Vector3(0, gridSizeY - 0.5, 10),
    new THREE.Euler(-Math.PI / 2, 0, 0)
  )

  // 前面 (+Z)
  createGrid(
    gridSizeX, gridSizeY, step,
    new THREE.Vector3(0, 0, gridSizeZ - 0.5),
    new THREE.Euler(0, 0, 0)
  )

  // 后面 (-Z)
  createGrid(
    gridSizeX, gridSizeY, step,
    new THREE.Vector3(0, 0, -0.5),
    new THREE.Euler(0, Math.PI, 0)
  )

  // 左面 (-X)
  createGrid(
    gridSizeZ, gridSizeY, step,
    new THREE.Vector3(-0.5, 0, 0),
    new THREE.Euler(0, Math.PI / 2, 0)
  )

  // 右面 (+X)
  createGrid(
    gridSizeZ, gridSizeY, step,
    new THREE.Vector3(gridSizeX - 0.5, 0, 0),
    new THREE.Euler(0, -Math.PI / 2, 0)
  )
}

// 生成小方块，带边框线
function createCube(x, y, z, color) {
  const geometry = new THREE.BoxGeometry(BLOCK_SIZE * 0.95, BLOCK_SIZE * 0.95, BLOCK_SIZE * 0.95)
  const material = new THREE.MeshPhysicalMaterial({
    color,
    metalness: 0.3,
    roughness: 0.2,
    clearcoat: 0.5,
    clearcoatRoughness: 0.1,
    emissive: new THREE.Color(color).multiplyScalar(0.15)
  })
  const cube = new THREE.Mesh(geometry, material)
  cube.position.set(x * BLOCK_SIZE, y * BLOCK_SIZE, z * BLOCK_SIZE)
  cube.castShadow = true
  cube.receiveShadow = true

  const edgesGeometry = new THREE.EdgesGeometry(geometry)
  const edgesMaterial = new THREE.LineBasicMaterial({ color: 0x222222, linewidth: 1 })
  const edges = new THREE.LineSegments(edgesGeometry, edgesMaterial)
  cube.add(edges)

  return cube
}

// 更新场景
function updateScene() {
  if(!blocksGroup) return
  blocksGroup.clear()

  for(let x=0; x<COLS; x++){
    for(let y=0; y<HEIGHT; y++){
      for(let z=0; z<ROWS; z++){
        const color = state.board[x][y][z]
        if(color){
          blocksGroup.add(createCube(x, y, z, color))
        }
      }
    }
  }

  if(state.currentPiece){
    const blocks = state.currentPiece
    const size = blocks.length
    for(let y=0; y<size; y++){
      for(let x=0; x<size; x++){
        for(let z=0; z<size; z++){
          if(blocks[y][x][z]){
            const bx = state.currentPos.x + x
            const by = state.currentPos.y + y
            const bz = state.currentPos.z + z
            blocksGroup.add(createCube(bx, by, bz, state.currentColor))
          }
        }
      }
    }
  }
}

// 动画循环
function animate() {
  animationId = requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}

// 产生新方块
function spawnPiece() {
  const idx = Math.floor(Math.random() * SHAPES.length)
  const shapeObj = SHAPES[idx]
  const blocks = cloneBlocks(shapeObj.blocks)
  const startPos = {
    x: Math.floor(COLS / 2) - 1,
    y: HEIGHT - 3,
    z: Math.floor(ROWS / 2) - 1
  }
  if(!canPlace(blocks, startPos, state.board)){
    state.isGameOver = true
    isGameOver.value = true
    stopGame()
    return
  }
  state.currentPiece = blocks
  state.currentPos = startPos
  state.currentColor = shapeObj.color
}

// 方块下落
function dropPiece() {
  if(state.isGameOver) return
  const newPos = { ...state.currentPos, y: state.currentPos.y - 1 }
  if(canPlace(state.currentPiece, newPos, state.board)){
    state.currentPos = newPos
  } else {
    // 固定
    placeBlocks(state.currentPiece, state.currentPos, state.board, state.currentColor)
    const cleared = clearFullLayers(state.board)
    if(cleared > 0) {
      score.value += cleared * 100
    }
    spawnPiece()
  }
  updateScene()
}

// 游戏主循环
function startGame() {
  state.board = createEmptyBoard()
  state.isGameOver = false
  isGameOver.value = false
  score.value = 0
  spawnPiece()
  updateScene()
  if(dropTimer) clearInterval(dropTimer)
  dropTimer = setInterval(dropPiece, 800)
  animate()
}

function stopGame() {
  if(dropTimer){
    clearInterval(dropTimer)
    dropTimer = null
  }
  if(animationId){
    cancelAnimationFrame(animationId)
    animationId = null
  }
}

function hardDrop() {
  if(state.isGameOver) return
  let newPos = {...state.currentPos}
  while(canPlace(state.currentPiece, {x:newPos.x, y:newPos.y-1, z:newPos.z}, state.board)){
    newPos.y -= 1
  }
  state.currentPos = newPos
  dropPiece()
}

function movePiece(dx, dz) {
  if(state.isGameOver) return
  const newPos = {x: state.currentPos.x + dx, y: state.currentPos.y, z: state.currentPos.z + dz}
  if(canPlace(state.currentPiece, newPos, state.board)){
    state.currentPos = newPos
    updateScene()
  }
}

function moveLeft() { movePiece(-1, 0) }
function moveRight() { movePiece(1, 0) }
function moveForward() { movePiece(0, -1) }
function moveBackward() { movePiece(0, 1) }

function rotateYPiece() {
  if(state.isGameOver) return
  const rotated = rotateY(state.currentPiece)
  if(canPlace(rotated, state.currentPos, state.board)){
    state.currentPiece = rotated
    updateScene()
  }
}

function rotateXPiece() {
  if(state.isGameOver) return
  const rotated = rotateX(state.currentPiece)
  if(canPlace(rotated, state.currentPos, state.board)){
    state.currentPiece = rotated
    updateScene()
  }
}

function rotateZPiece() {
  if(state.isGameOver) return
  const rotated = rotateZ(state.currentPiece)
  if(canPlace(rotated, state.currentPos, state.board)){
    state.currentPiece = rotated
    updateScene()
  }
}

function handleKeydown(e) {
  if(state.isGameOver && e.key.toLowerCase() === 'r'){
    startGame()
    return
  }
  switch(e.key.toLowerCase()){
    case 'arrowleft':
    case 'a': moveLeft(); break
    case 'arrowright':
    case 'd': moveRight(); break
    case 'w': moveForward(); break
    case 's': moveBackward(); break
    case 'arrowdown': dropPiece(); break
    case 'arrowup': rotateYPiece(); break
    case 'q': rotateXPiece(); break
    case 'e': rotateZPiece(); break
    case ' ': hardDrop(); break
  }
}

onMounted(() => {
  initThree()
  startGame()
  window.addEventListener('resize', () => {
    if(!container.value) return
    const width = container.value.clientWidth
    const height = container.value.clientHeight
    camera.aspect = width / height
    camera.updateProjectionMatrix()
    renderer.setSize(width, height)
  })
})

onBeforeUnmount(() => {
  stopGame()
  window.removeEventListener('resize', () => {})
})

</script>

<style scoped>
.game-wrapper {
  display: flex;
  gap: 20px;
  padding: 20px;
  background: #121212;
  color: #eee;
  font-family: "Microsoft YaHei", sans-serif;
  height: 100vh;
  box-sizing: border-box;
}
.tetris-container {
  flex: 1;
  background: #1e1e1e;
  border-radius: 8px;
  outline: none;
  user-select: none;
  cursor: grab;
  min-width: 320px;
  min-height: 600px;
}
.info-panel {
  width: 280px;
  padding: 15px;
  background: #222;
  border-radius: 8px;
  overflow-y: auto;
  font-size: 14px;
  line-height: 1.4;
}
.info-panel h2 {
  margin-top: 0;
  margin-bottom: 10px;
  color: #aaccff;
}
.info-panel ul {
  padding-left: 16px;
  margin: 8px 0;
}
.game-over {
  color: #ff5555;
  font-weight: bold;
}
</style>

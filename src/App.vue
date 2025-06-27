<script setup>
import ParticleClock from './components/ParticleClock.vue'
import RippleParticleClock from './components/RippleParticleClock.vue'
import RippleContainer from './components/RippleContainer.vue'
import TetrisGame from './components/TetrisGame.vue'
import Snake3D from './components/Snake3D.vue'  // 新增3D贪吃蛇组件
</script>

<template>
  <div class="wrapper">
    <section>
      <h2>粒子数字时钟</h2>
      <ParticleClock />
    </section>

    <section>
      <h2>水波涟漪时钟</h2>
      <RippleParticleClock />
    </section>

    <section>
      <h2>鼠标交互水波涟漪容器</h2>
      <RippleContainer />
    </section>

    <section>
      <h2>俄罗斯方块小游戏</h2>
      <TetrisGame />
    </section>

    <!-- 新增3D贪吃蛇区域 -->
    <section>
      <h2>3D 贪吃蛇小游戏</h2>
      <Snake3D />
    </section>
  </div>
</template>

<style scoped>
.wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 60px;
  padding: 40px;
  width: 100%;
  box-sizing: border-box;
}

section {
  width: 100%;
  max-width: 1600px;
  margin: 0 auto;
  text-align: center;
}
</style>

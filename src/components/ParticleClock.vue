<template>
  <div class="clock-container">
    <ParticleNumber
      v-for="(char, i) in timeArray"
      :key="i"
      :digit="char"
    />
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import ParticleNumber from './ParticleNumber.vue'

const timeArray = ref([])

function updateTime() {
  const now = new Date()
  const h = now.getHours().toString().padStart(2, '0')
  const m = now.getMinutes().toString().padStart(2, '0')
  const s = now.getSeconds().toString().padStart(2, '0')

  timeArray.value = (h + ':' + m + ':' + s).split('')
}

onMounted(() => {
  updateTime()
  const timer = setInterval(updateTime, 1000)
  onBeforeUnmount(() => clearInterval(timer))
})
</script>

<style scoped>
.clock-container {
  display: flex;
  align-items: center;
  gap: 10px;
  background: black;
  padding: 20px;
  justify-content: center;
}
</style>

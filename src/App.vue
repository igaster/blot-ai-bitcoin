<script setup lang="ts">
import BitcoinChart from './components/BitcoinChart.vue'
import { ref, onMounted } from 'vue'

const isDark = ref(false)

function toggleDark() {
  isDark.value = !isDark.value
  document.documentElement.classList.toggle('dark')
}

onMounted(() => {
  // Check system preference
  if (window.matchMedia('(prefers-color-scheme: dark)').matches) {
    isDark.value = true
    document.documentElement.classList.add('dark')
  }
})
</script>

<template>
  <div class="min-h-screen bg-gray-100 dark:bg-gray-900 py-8 transition-colors duration-200">
    <div class="container mx-auto">
      <div class="flex justify-between items-center mb-8 px-4">
        <h1 class="text-3xl font-bold text-center text-gray-900 dark:text-white">Bitcoin Price Chart</h1>
        <button 
          @click="toggleDark" 
          class="px-4 py-2 rounded-lg bg-gray-200 dark:bg-gray-700 text-gray-800 dark:text-white hover:bg-gray-300 dark:hover:bg-gray-600 transition-colors"
        >
          {{ isDark ? '☀️ Light' : '🌙 Dark' }}
        </button>
      </div>
      <BitcoinChart />
    </div>
  </div>
</template>
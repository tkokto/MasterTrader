<script setup lang="ts">
import { ref } from 'vue'

defineProps<{ msg: string }>()

const count = ref(0)
</script>

<template>
  <h1 class="text-4xl font-bold text-blue-600 mb-6">{{ msg }}</h1>

  <div class="card bg-gray-100 dark:bg-gray-800 rounded-lg p-6 shadow-lg">
    <button 
      type="button" 
      @click="count++" 
      class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded transition-colors duration-300"
    >
      count is {{ count }}
    </button>
    <p class="mt-4 text-gray-600 dark:text-gray-300">
      Edit
      <code class="bg-gray-200 dark:bg-gray-700 px-2 py-1 rounded">components/HelloWorld.vue</code> to test HMR
    </p>
  </div>

  <p>
    Check out
    <a href="https://vuejs.org/guide/quick-start.html#local" target="_blank"
      >create-vue</a
    >, the official Vue + Vite starter
  </p>
  <p>
    Learn more about IDE Support for Vue in the
    <a
      href="https://vuejs.org/guide/scaling-up/tooling.html#ide-support"
      target="_blank"
      >Vue Docs Scaling up Guide</a
    >.
  </p>
  <p class="read-the-docs">Click on the Vite and Vue logos to learn more</p>
</template>

<style scoped>
.read-the-docs {
  color: #888;
}
</style>

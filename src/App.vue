<script setup>
import GameCanvas from './components/GameCanvas.vue'
</script>

<template>
  <div class="min-h-screen bg-gray-900 text-white flex flex-col items-center justify-center py-8">
    <h1 class="text-4xl font-bold mb-2 text-transparent bg-clip-text bg-gradient-to-r from-green-400 to-blue-500">
      Turbo Racer
    </h1>
    <GameCanvas />
  </div>
</template>
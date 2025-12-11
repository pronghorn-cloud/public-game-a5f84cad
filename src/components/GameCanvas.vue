<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const canvasRef = ref(null);
const score = ref(0);
const gameOver = ref(false);
const gameStarted = ref(false);
const finalScore = ref(0);

// Game constants
const CAR_WIDTH = 50;
const CAR_HEIGHT = 80;
const ROAD_WIDTH = 400;
const LANE_WIDTH = ROAD_WIDTH / 3;

// Game state
let ctx = null;
let animationFrameId = null;
let playerX = ROAD_WIDTH / 2 - CAR_WIDTH / 2;
let playerY = 0; // Set in init
let obstacles = [];
let roadOffset = 0;
let speed = 5;
let lastTime = 0;
let keys = {
  ArrowLeft: false,
  ArrowRight: false,
  a: false,
  d: false
};

const initGame = () => {
  if (!canvasRef.value) return;
  const canvas = canvasRef.value;
  ctx = canvas.getContext('2d');
  
  // Reset state
  playerY = canvas.height - CAR_HEIGHT - 20;
  playerX = canvas.width / 2 - CAR_WIDTH / 2;
  obstacles = [];
  score.value = 0;
  speed = 5;
  gameOver.value = false;
  gameStarted.value = true;
  
  lastTime = performance.now();
  loop();
};

const spawnObstacle = () => {
  const lane = Math.floor(Math.random() * 3);
  const obstacleX = (canvasRef.value.width - ROAD_WIDTH) / 2 + lane * LANE_WIDTH + (LANE_WIDTH - CAR_WIDTH) / 2;
  
  // Ensure we don't spawn too close to another obstacle
  const tooClose = obstacles.some(obs => obs.y < -CAR_HEIGHT * 2);
  if (!tooClose) {
    obstacles.push({
      x: obstacleX,
      y: -100,
      speed: speed * 0.8,
      color: getRandomColor()
    });
  }
};

const getRandomColor = () => {
  const colors = ['#FF5733', '#33FF57', '#3357FF', '#F333FF', '#FFFF33'];
  return colors[Math.floor(Math.random() * colors.length)];
};

const update = (deltaTime) => {
  if (gameOver.value) return;

  // Move player
  const moveSpeed = speed * 1.5;
  if (keys.ArrowLeft || keys.a) playerX -= moveSpeed;
  if (keys.ArrowRight || keys.d) playerX += moveSpeed;

  // Constrain player to road
  const roadLeft = (canvasRef.value.width - ROAD_WIDTH) / 2;
  const roadRight = roadLeft + ROAD_WIDTH;
  
  if (playerX < roadLeft) playerX = roadLeft;
  if (playerX + CAR_WIDTH > roadRight) playerX = roadRight - CAR_WIDTH;

  // Update road offset for animation
  roadOffset = (roadOffset + speed) % 40;

  // Update obstacles
  if (Math.random() < 0.02) spawnObstacle();

  obstacles.forEach((obs, index) => {
    obs.y += speed;
    
    // Collision detection
    if (
      playerX < obs.x + CAR_WIDTH &&
      playerX + CAR_WIDTH > obs.x &&
      playerY < obs.y + CAR_HEIGHT &&
      playerY + CAR_HEIGHT > obs.y
    ) {
      endGame();
    }

    // Remove off-screen obstacles and increase score
    if (obs.y > canvasRef.value.height) {
      obstacles.splice(index, 1);
      score.value += 10;
      // Increase difficulty
      if (score.value % 100 === 0) speed += 0.5;
    }
  });
};

const draw = () => {
  if (!ctx || !canvasRef.value) return;
  const canvas = canvasRef.value;
  
  // Clear canvas
  ctx.fillStyle = '#2d3748'; // Dark gray bg
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // Draw Road
  const roadLeft = (canvas.width - ROAD_WIDTH) / 2;
  ctx.fillStyle = '#4a5568'; // Road color
  ctx.fillRect(roadLeft, 0, ROAD_WIDTH, canvas.height);

  // Draw Lane Markers
  ctx.strokeStyle = '#ffffff';
  ctx.setLineDash([20, 20]);
  ctx.lineDashOffset = -roadOffset;
  ctx.lineWidth = 4;

  ctx.beginPath();
  ctx.moveTo(roadLeft + LANE_WIDTH, 0);
  ctx.lineTo(roadLeft + LANE_WIDTH, canvas.height);
  ctx.moveTo(roadLeft + LANE_WIDTH * 2, 0);
  ctx.lineTo(roadLeft + LANE_WIDTH * 2, canvas.height);
  ctx.stroke();

  // Draw Borders
  ctx.strokeStyle = '#f56565'; // Red borders
  ctx.setLineDash([]);
  ctx.lineWidth = 6;
  ctx.beginPath();
  ctx.moveTo(roadLeft, 0);
  ctx.lineTo(roadLeft, canvas.height);
  ctx.moveTo(roadLeft + ROAD_WIDTH, 0);
  ctx.lineTo(roadLeft + ROAD_WIDTH, canvas.height);
  ctx.stroke();

  // Draw Player Car
  drawCar(playerX, playerY, '#4299e1'); // Blue car

  // Draw Obstacles
  obstacles.forEach(obs => {
    drawCar(obs.x, obs.y, obs.color);
  });
};

const drawCar = (x, y, color) => {
  ctx.fillStyle = color;
  // Main body
  ctx.fillRect(x, y, CAR_WIDTH, CAR_HEIGHT);
  
  // Roof
  ctx.fillStyle = '#00000040';
  ctx.fillRect(x + 5, y + 20, CAR_WIDTH - 10, CAR_HEIGHT - 40);
  
  // Headlights
  ctx.fillStyle = '#ffff00';
  ctx.fillRect(x + 5, y, 10, 5);
  ctx.fillRect(x + CAR_WIDTH - 15, y, 10, 5);
};

const loop = (timestamp) => {
  if (!gameStarted.value || gameOver.value) return;
  
  const deltaTime = timestamp - lastTime;
  lastTime = timestamp;

  update(deltaTime);
  draw();
  
  animationFrameId = requestAnimationFrame(loop);
};

const endGame = () => {
  gameOver.value = true;
  gameStarted.value = false;
  finalScore.value = score.value;
  cancelAnimationFrame(animationFrameId);
};

const handleKeydown = (e) => {
  if (keys.hasOwnProperty(e.key)) {
    keys[e.key] = true;
  }
};

const handleKeyup = (e) => {
  if (keys.hasOwnProperty(e.key)) {
    keys[e.key] = false;
  }
};

onMounted(() => {
  window.addEventListener('keydown', handleKeydown);
  window.addEventListener('keyup', handleKeyup);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown);
  window.removeEventListener('keyup', handleKeyup);
  cancelAnimationFrame(animationFrameId);
});
</script>

<template>
  <div class="relative flex flex-col items-center justify-center p-4">
    <div class="mb-4 text-2xl font-bold text-white flex gap-8">
      <span>Score: {{ score }}</span>
      <span v-if="gameStarted" class="text-sm opacity-75">Speed: {{ Math.floor(speed * 10) }} km/h</span>
    </div>
    
    <div class="relative">
      <canvas 
        ref="canvasRef"
        width="600"
        height="800"
        class="bg-gray-800 rounded-lg shadow-2xl border-4 border-gray-700"
      ></canvas>

      <!-- Start Screen -->
      <div 
        v-if="!gameStarted && !gameOver"
        class="absolute inset-0 flex flex-col items-center justify-center bg-black/70 rounded-lg"
      >
        <h2 class="text-4xl font-bold text-white mb-4">Ready to Race?</h2>
        <p class="text-gray-300 mb-8">Use Left/Right Arrows or A/D to move</p>
        <button 
          @click="initGame"
          class="px-8 py-3 bg-green-500 hover:bg-green-600 text-white font-bold rounded-full text-xl transition transform hover:scale-105"
        >
          START ENGINE
        </button>
      </div>

      <!-- Game Over Screen -->
      <div 
        v-if="gameOver"
        class="absolute inset-0 flex flex-col items-center justify-center bg-red-900/80 rounded-lg"
      >
        <h2 class="text-5xl font-bold text-white mb-2">CRASHED!</h2>
        <p class="text-2xl text-gray-200 mb-8">Final Score: {{ finalScore }}</p>
        <button 
          @click="initGame"
          class="px-8 py-3 bg-white text-red-600 hover:bg-gray-100 font-bold rounded-full text-xl transition transform hover:scale-105"
        >
          TRY AGAIN
        </button>
      </div>
    </div>
    
    <div class="mt-4 text-gray-400 text-sm">
      <p>Avoid the other cars! They speed up as you go further.</p>
    </div>
  </div>
</template>

<style scoped>
canvas {
  max-width: 100%;
  max-height: 80vh;
}
</style>
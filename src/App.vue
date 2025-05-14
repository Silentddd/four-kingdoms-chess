<template>
  <div class="app-container">
    <h1>四国军棋</h1>
    <div class="game-board">
      <ChessBoard />
    </div>
  </div>
</template>

<script setup>
import ChessBoard from './components/ChessBoard.vue';
</script>

<style scoped>
.app-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  text-align: center;
}

.game-board {
  margin-top: 20px;
}

h1 {
  color: #333;
  margin-bottom: 20px;
}
</style> 
<template>
  <div class="chess-board-container">
    <div class="game-info">
      <h2>当前回合: {{ currentPlayer }}</h2>
      <div class="player-status">
        <div class="player red" :class="{ active: currentPlayer === '红方' }">红方</div>
        <div class="player blue" :class="{ active: currentPlayer === '蓝方' }">蓝方</div>
        <div class="player green" :class="{ active: currentPlayer === '绿方' }">绿方</div>
        <div class="player yellow" :class="{ active: currentPlayer === '黄方' }">黄方</div>
      </div>
      <el-button type="primary" @click="startGame">开始新游戏</el-button>
    </div>
    
    <div class="chess-board">
      <div v-for="(row, rowIndex) in board" :key="'row-' + rowIndex" class="board-row">
        <div 
          v-for="(cell, colIndex) in row" 
          :key="'cell-' + rowIndex + '-' + colIndex" 
          class="board-cell"
          :class="getCellClass(rowIndex, colIndex, cell)"
          @click="handleCellClick(rowIndex, colIndex)"
        >
          <div v-if="cell.piece" class="chess-piece" :class="cell.piece.color">
            {{ cell.piece.rank }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import { ElMessage } from 'element-plus';

// 棋盘尺寸
const BOARD_SIZE = 17;
const CAMP_SIZE = 5;

// 玩家颜色
const PLAYERS = ['红方', '蓝方', '绿方', '黄方'];
const COLORS = ['red', 'blue', 'green', 'yellow'];

// 棋子等级
const RANKS = {
  '司令': 9,
  '军长': 8,
  '师长': 7,
  '旅长': 6,
  '团长': 5,
  '营长': 4,
  '连长': 3,
  '排长': 2,
  '工兵': 1,
  '炸弹': 'Z',
  '地雷': 'L',
  '军旗': 'F'
};

// 棋子初始配置 (每种棋子的数量)
const PIECE_CONFIG = {
  '司令': 1,
  '军长': 1,
  '师长': 2,
  '旅长': 2,
  '团长': 2,
  '营长': 2,
  '连长': 3,
  '排长': 3,
  '工兵': 3,
  '炸弹': 2,
  '地雷': 3,
  '军旗': 1
};

// 游戏状态
const board = ref([]);
const currentPlayer = ref('');
const selectedCell = ref(null);
const gameStarted = ref(false);

// 初始化棋盘
const initializeBoard = () => {
  const newBoard = [];
  
  // 创建空棋盘
  for (let i = 0; i < BOARD_SIZE; i++) {
    const row = [];
    for (let j = 0; j < BOARD_SIZE; j++) {
      row.push({
        type: 'normal',
        piece: null,
        available: false
      });
    }
    newBoard.push(row);
  }
  
  // 标记铁路轨道
  markRailwayTracks(newBoard);
  
  // 标记大本营
  markCamps(newBoard);
  
  // 标记行营
  markSpecialFields(newBoard);
  
  board.value = newBoard;
};

// 标记铁路轨道
const markRailwayTracks = (board) => {
  // 横向铁路
  for (let i = 0; i < BOARD_SIZE; i++) {
    if (i === 0 || i === 5 || i === 6 || i === 10 || i === 11 || i === 16) {
      for (let j = 0; j < BOARD_SIZE; j++) {
        if (j >= 1 && j <= 15) {
          board[i][j].type = 'railway';
        }
      }
    }
  }
  
  // 纵向铁路
  for (let j = 0; j < BOARD_SIZE; j++) {
    if (j === 0 || j === 5 || j === 6 || j === 10 || j === 11 || j === 16) {
      for (let i = 0; i < BOARD_SIZE; i++) {
        if (i >= 1 && i <= 15) {
          board[i][j].type = 'railway';
        }
      }
    }
  }
};

// 标记大本营
const markCamps = (board) => {
  // 红方大本营 (左上)
  for (let i = 0; i < CAMP_SIZE; i++) {
    for (let j = 0; j < CAMP_SIZE; j++) {
      board[i][j].type = 'camp';
      board[i][j].color = 'red';
    }
  }
  
  // 蓝方大本营 (右上)
  for (let i = 0; i < CAMP_SIZE; i++) {
    for (let j = BOARD_SIZE - CAMP_SIZE; j < BOARD_SIZE; j++) {
      board[i][j].type = 'camp';
      board[i][j].color = 'blue';
    }
  }
  
  // 绿方大本营 (右下)
  for (let i = BOARD_SIZE - CAMP_SIZE; i < BOARD_SIZE; i++) {
    for (let j = BOARD_SIZE - CAMP_SIZE; j < BOARD_SIZE; j++) {
      board[i][j].type = 'camp';
      board[i][j].color = 'green';
    }
  }
  
  // 黄方大本营 (左下)
  for (let i = BOARD_SIZE - CAMP_SIZE; i < BOARD_SIZE; i++) {
    for (let j = 0; j < CAMP_SIZE; j++) {
      board[i][j].type = 'camp';
      board[i][j].color = 'yellow';
    }
  }
};

// 标记行营和障碍物
const markSpecialFields = (board) => {
  // 行营位置
  const campPositions = [
    [2, 7], [3, 8], [4, 7], [3, 6],  // 上方行营
    [7, 2], [8, 3], [7, 4], [6, 3],  // 左方行营
    [12, 7], [13, 8], [14, 7], [13, 6],  // 下方行营
    [7, 12], [8, 13], [7, 14], [6, 13],  // 右方行营
    [8, 8]  // 中央行营
  ];
  
  // 标记行营
  for (const [row, col] of campPositions) {
    board[row][col].type = 'fieldcamp';
  }
  
  // 标记中央障碍物
  board[8][8].type = 'obstacle';
};

// 随机放置棋子
const placeRandomPieces = () => {
  const piecePool = [];
  
  // 创建棋子池
  Object.entries(PIECE_CONFIG).forEach(([rank, count]) => {
    for (let i = 0; i < count; i++) {
      piecePool.push(rank);
    }
  });
  
  // 为每个玩家随机分配棋子
  COLORS.forEach((color, playerIndex) => {
    const campArea = getCampArea(playerIndex);
    const shuffledPieces = [...piecePool].sort(() => Math.random() - 0.5);
    
    let pieceIndex = 0;
    for (const [row, col] of campArea) {
      if (pieceIndex < shuffledPieces.length) {
        board.value[row][col].piece = {
          rank: shuffledPieces[pieceIndex],
          color: color,
          revealed: true  // 开发阶段设为true，实际游戏中应为false
        };
        pieceIndex++;
      }
    }
  });
};

// 获取玩家大本营区域坐标
const getCampArea = (playerIndex) => {
  const campCells = [];
  
  if (playerIndex === 0) {  // 红方 (左上)
    for (let i = 0; i < CAMP_SIZE; i++) {
      for (let j = 0; j < CAMP_SIZE; j++) {
        campCells.push([i, j]);
      }
    }
  } else if (playerIndex === 1) {  // 蓝方 (右上)
    for (let i = 0; i < CAMP_SIZE; i++) {
      for (let j = BOARD_SIZE - CAMP_SIZE; j < BOARD_SIZE; j++) {
        campCells.push([i, j]);
      }
    }
  } else if (playerIndex === 2) {  // 绿方 (右下)
    for (let i = BOARD_SIZE - CAMP_SIZE; i < BOARD_SIZE; i++) {
      for (let j = BOARD_SIZE - CAMP_SIZE; j < BOARD_SIZE; j++) {
        campCells.push([i, j]);
      }
    }
  } else if (playerIndex === 3) {  // 黄方 (左下)
    for (let i = BOARD_SIZE - CAMP_SIZE; i < BOARD_SIZE; i++) {
      for (let j = 0; j < CAMP_SIZE; j++) {
        campCells.push([i, j]);
      }
    }
  }
  
  return campCells;
};

// 开始游戏
const startGame = () => {
  initializeBoard();
  placeRandomPieces();
  currentPlayer.value = PLAYERS[0];  // 红方先行
  gameStarted.value = true;
  selectedCell.value = null;
  ElMessage.success('游戏开始！红方先行');
};

// 获取单元格CSS类
const getCellClass = (row, col, cell) => {
  const classes = [cell.type];
  
  if (cell.color) {
    classes.push(`camp-${cell.color}`);
  }
  
  if (selectedCell.value && selectedCell.value.row === row && selectedCell.value.col === col) {
    classes.push('selected');
  }
  
  if (cell.available) {
    classes.push('available');
  }
  
  return classes;
};

// 处理单元格点击
const handleCellClick = (row, col) => {
  if (!gameStarted.value) {
    ElMessage.warning('请先开始游戏');
    return;
  }
  
  const cell = board.value[row][col];
  
  // 如果没有选中棋子并且点击了有棋子的格子
  if (!selectedCell.value && cell.piece) {
    // 只能选择当前玩家的棋子
    const playerColor = COLORS[PLAYERS.indexOf(currentPlayer.value)];
    if (cell.piece.color !== playerColor) {
      ElMessage.warning('只能移动自己的棋子');
      return;
    }
    
    // 选中棋子
    selectedCell.value = { row, col };
    highlightAvailableMoves(row, col);
  } 
  // 如果已经选中了棋子
  else if (selectedCell.value) {
    // 如果点击了可移动的位置
    if (cell.available) {
      movePiece(selectedCell.value.row, selectedCell.value.col, row, col);
      clearHighlights();
      selectedCell.value = null;
      switchPlayer();
    } 
    // 如果点击了自己的另一个棋子，切换选择
    else if (cell.piece && cell.piece.color === COLORS[PLAYERS.indexOf(currentPlayer.value)]) {
      clearHighlights();
      selectedCell.value = { row, col };
      highlightAvailableMoves(row, col);
    } 
    // 取消选择
    else {
      clearHighlights();
      selectedCell.value = null;
    }
  }
};

// 高亮可移动位置
const highlightAvailableMoves = (row, col) => {
  clearHighlights();
  
  const piece = board.value[row][col].piece;
  if (!piece) return;
  
  // 获取可移动的位置
  const availableMoves = getAvailableMoves(row, col, piece);
  
  // 标记可移动的位置
  for (const [r, c] of availableMoves) {
    board.value[r][c].available = true;
  }
};

// 清除高亮
const clearHighlights = () => {
  for (let i = 0; i < BOARD_SIZE; i++) {
    for (let j = 0; j < BOARD_SIZE; j++) {
      board.value[i][j].available = false;
    }
  }
};

// 获取可移动的位置
const getAvailableMoves = (row, col, piece) => {
  const moves = [];
  
  // 军旗不能移动
  if (piece.rank === 'F') {
    return moves;
  }
  
  // 检查四个方向 (上、右、下、左)
  const directions = [[-1, 0], [0, 1], [1, 0], [0, -1]];
  
  for (const [dr, dc] of directions) {
    const newRow = row + dr;
    const newCol = col + dc;
    
    // 检查边界
    if (newRow < 0 || newRow >= BOARD_SIZE || newCol < 0 || newCol >= BOARD_SIZE) {
      continue;
    }
    
    const targetCell = board.value[newRow][newCol];
    
    // 不能走入障碍物
    if (targetCell.type === 'obstacle') {
      continue;
    }
    
    // 不能走入自己的军旗
    if (targetCell.piece && targetCell.piece.color === piece.color && targetCell.piece.rank === 'F') {
      continue;
    }
    
    // 不能走入自己的棋子
    if (targetCell.piece && targetCell.piece.color === piece.color) {
      continue;
    }
    
    // 行营特殊规则：可以进入空行营，但不能吃行营中的棋子
    if (targetCell.type === 'fieldcamp') {
      if (!targetCell.piece) {
        moves.push([newRow, newCol]);
      }
      continue;
    }
    
    // 普通移动或吃子
    moves.push([newRow, newCol]);
  }
  
  // 工兵可以沿铁路无限移动
  if (piece.rank === '工兵' && board.value[row][col].type === 'railway') {
    addRailwayMoves(row, col, moves, piece.color);
  }
  
  return moves;
};

// 添加铁路移动
const addRailwayMoves = (row, col, moves, pieceColor) => {
  // 铁路方向 (上、右、下、左)
  const directions = [[-1, 0], [0, 1], [1, 0], [0, -1]];
  
  for (const [dr, dc] of directions) {
    let currentRow = row;
    let currentCol = col;
    
    while (true) {
      currentRow += dr;
      currentCol += dc;
      
      // 检查边界
      if (currentRow < 0 || currentRow >= BOARD_SIZE || currentCol < 0 || currentCol >= BOARD_SIZE) {
        break;
      }
      
      const targetCell = board.value[currentRow][currentCol];
      
      // 只能沿铁路移动
      if (targetCell.type !== 'railway') {
        break;
      }
      
      // 遇到棋子停止
      if (targetCell.piece) {
        // 如果是敌方棋子，可以吃
        if (targetCell.piece.color !== pieceColor) {
          moves.push([currentRow, currentCol]);
        }
        break;
      }
      
      // 空格可以移动
      moves.push([currentRow, currentCol]);
    }
  }
};

// 移动棋子
const movePiece = (fromRow, fromCol, toRow, toCol) => {
  const fromCell = board.value[fromRow][fromCol];
  const toCell = board.value[toRow][toCol];
  
  // 如果目标位置有敌方棋子，进行战斗
  if (toCell.piece) {
    const result = battle(fromCell.piece, toCell.piece);
    
    if (result === 'win') {
      // 攻击方获胜，移动到目标位置
      toCell.piece = fromCell.piece;
      fromCell.piece = null;
      ElMessage.success('战斗胜利！');
      
      // 检查是否胜利（吃掉军旗）
      if (toCell.piece.rank === 'F') {
        announceWinner();
      }
    } else if (result === 'lose') {
      // 攻击方失败，移除攻击棋子
      fromCell.piece = null;
      ElMessage.error('战斗失败！');
    } else {
      // 同归于尽
      fromCell.piece = null;
      toCell.piece = null;
      ElMessage.warning('同归于尽！');
    }
  } else {
    // 目标位置为空，直接移动
    toCell.piece = fromCell.piece;
    fromCell.piece = null;
  }
};

// 战斗规则
const battle = (attacker, defender) => {
  // 显示双方棋子
  attacker.revealed = true;
  defender.revealed = true;
  
  // 工兵可以拆地雷
  if (attacker.rank === '工兵' && defender.rank === 'L') {
    return 'win';
  }
  
  // 任何棋子碰到军旗都获胜
  if (defender.rank === 'F') {
    return 'win';
  }
  
  // 炸弹与任何棋子同归于尽（包括地雷）
  if (attacker.rank === 'Z' || defender.rank === 'Z') {
    return 'draw';
  }
  
  // 所有棋子碰到地雷都失败（除了工兵）
  if (defender.rank === 'L') {
    return 'lose';
  }
  
  // 常规战斗规则
  const attackerRank = RANKS[attacker.rank];
  const defenderRank = RANKS[defender.rank];
  
  if (attackerRank > defenderRank) {
    return 'win';
  } else if (attackerRank < defenderRank) {
    return 'lose';
  } else {
    return 'draw';
  }
};

// 切换玩家
const switchPlayer = () => {
  const currentIndex = PLAYERS.indexOf(currentPlayer.value);
  const nextIndex = (currentIndex + 1) % PLAYERS.length;
  currentPlayer.value = PLAYERS[nextIndex];
};

// 宣布胜利者
const announceWinner = () => {
  ElMessage({
    message: `游戏结束！${currentPlayer.value}获胜！`,
    type: 'success',
    duration: 0
  });
  gameStarted.value = false;
};

// 初始化游戏
onMounted(() => {
  initializeBoard();
});
</script>

<style scoped>
.chess-board-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.game-info {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  margin-bottom: 20px;
}

.player-status {
  display: flex;
  gap: 20px;
  margin: 10px 0 20px;
}

.player {
  padding: 8px 16px;
  border-radius: 4px;
  font-weight: bold;
  opacity: 0.6;
}

.player.active {
  opacity: 1;
  box-shadow: 0 0 8px rgba(0, 0, 0, 0.3);
}

.player.red {
  background-color: #ff4d4f;
  color: white;
}

.player.blue {
  background-color: #1890ff;
  color: white;
}

.player.green {
  background-color: #52c41a;
  color: white;
}

.player.yellow {
  background-color: #faad14;
  color: white;
}

.chess-board {
  display: flex;
  flex-direction: column;
  border: 2px solid #333;
}

.board-row {
  display: flex;
}

.board-cell {
  width: 40px;
  height: 40px;
  border: 1px solid #ccc;
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  background-color: #f0f0f0;
}

.board-cell.railway {
  background-color: #e6f7ff;
}

.board-cell.camp {
  background-color: #f9f9f9;
}

.board-cell.camp-red {
  background-color: #fff1f0;
}

.board-cell.camp-blue {
  background-color: #e6f7ff;
}

.board-cell.camp-green {
  background-color: #f6ffed;
}

.board-cell.camp-yellow {
  background-color: #fffbe6;
}

.board-cell.fieldcamp {
  background-color: #f0f0f0;
  border: 1px dashed #666;
}

.board-cell.obstacle {
  background-color: #d9d9d9;
}

.board-cell.selected {
  box-shadow: inset 0 0 0 3px #40a9ff;
}

.board-cell.available {
  box-shadow: inset 0 0 0 3px #52c41a;
  cursor: pointer;
}

.chess-piece {
  width: 35px;
  height: 35px;
  border-radius: 50%;
  display: flex;
  justify-content: center;
  align-items: center;
  font-weight: bold;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
}

.chess-piece.red {
  background-color: #ff4d4f;
  color: white;
}

.chess-piece.blue {
  background-color: #1890ff;
  color: white;
}

.chess-piece.green {
  background-color: #52c41a;
  color: white;
}

.chess-piece.yellow {
  background-color: #faad14;
  color: white;
}
</style> 
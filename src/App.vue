<template>
  <div class="tetris">
    <div class="game-header">
      <div class="score">分数: {{ score }}</div>
      <div class="level">等级: {{ level }}</div>
    </div>

    <div class="game-container">
      <!-- 左侧控制区 -->
      <div class="left-controls">
        <div class="direction-pad">
          <button @click="moveLeft" class="dir-btn left-btn">←</button>
          <button @click="moveUp" class="dir-btn up-btn">↑</button>
          <button @click="moveDown" class="dir-btn down-btn">↓</button>
          <button @click="moveRight" class="dir-btn right-btn">→</button>
        </div>
      </div>

      <!-- 游戏主区域 -->
      <div class="board-container">
        <div class="board">
          <div v-for="(row, rowIndex) in board"
               :key="rowIndex"
               class="row">
            <div v-for="(cell, colIndex) in row"
                 :key="colIndex"
                 class="cell"
                 :class="{ filled: cell }">
            </div>
          </div>
        </div>
      </div>

      <!-- 右侧控制区 -->
      <div class="right-controls">
        <div class="next-shape-container">
          <h3>下一个:</h3>
          <div class="mini-board">
            <div v-for="(row, rowIndex) in nextShapeBoard"
                 :key="rowIndex"
                 class="row">
              <div v-for="(cell, colIndex) in row"
                   :key="colIndex"
                   class="cell"
                   :class="{ filled: cell }">
              </div>
            </div>
          </div>
        </div>
        <div class="game-controls">
          <button @click="togglePause" class="control-btn">
            {{ isPaused ? '继续' : '暂停' }}
          </button>
          <button @click="startNewGame" class="control-btn">新游戏</button>
        </div>
        <div class="action-buttons">
          <button @click="hardDrop" class="action-btn drop-btn">直落</button>
          <button @click="rotate" class="action-btn rotate-btn">旋转</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted } from 'vue';

export default {
  setup() {
    // 首先定义所有响应式状态
    const boardWidth = 10;
    const boardHeight = 20;
    const board = ref(Array.from({ length: boardHeight }, () => Array(boardWidth).fill(false)));
    const score = ref(0);
    const level = ref(1);
    const isPaused = ref(false);
    const isGameOver = ref(false);
    const nextShape = ref(null);
    const nextShapeBoard = ref(Array.from({ length: 4 }, () => Array(4).fill(false)));

    // 非响应式状态
    let currentShape = null;
    let shapePosition = { x: 0, y: 0 };
    let gameInterval = null;

    const shapes = [
      // I shape
      [
        [false, false, false, false],
        [true, true, true, true],
        [false, false, false, false],
        [false, false, false, false],
      ],
      // O shape
      [
        [false, false, false, false],
        [false, true, true, false],
        [false, true, true, false],
        [false, false, false, false],
      ],
      // T shape
      [
        [false, false, false, false],
        [false, true, true, true],
        [false, false, true, false],
        [false, false, false, false],
      ],
      // Z shape
      [
        [false, false, false, false],
        [true, true, false, false],
        [false, true, true, false],
        [false, false, false, false],
      ],
      // S shape (Z shape的镜像)
      [
        [false, false, false, false],
        [false, true, true, false],
        [true, true, false, false],
        [false, false, false, false],
      ],
      // L shape
      [
        [false, false, false, false],
        [true, true, true, false],
        [true, false, false, false],
        [false, false, false, false],
      ],
      // J shape (L shape的镜像)
      [
        [false, false, false, false],
        [true, true, true, false],
        [false, false, true, false],
        [false, false, false, false],
      ]
    ];

    const randomShape = () => shapes[Math.floor(Math.random() * shapes.length)];

    const drawShape = (shape, x, y) => {
      shape.forEach((row, rowIndex) => {
        row.forEach((cell, colIndex) => {
          if (cell && isValidPosition(x + colIndex, y + rowIndex)) {
            board.value[rowIndex + y][colIndex + x] = true;
          }
        });
      });
    };

    const clearShape = (shape, x, y) => {
      shape.forEach((row, rowIndex) => {
        row.forEach((cell, colIndex) => {
          if (cell && isValidPosition(x + colIndex, y + rowIndex)) {
            board.value[rowIndex + y][colIndex + x] = false;
          }
        });
      });
    };

    const moveLeft = () => {
      if (canMoveLeft(currentShape, shapePosition.x, shapePosition.y)) {
        clearShape(currentShape, shapePosition.x, shapePosition.y);
        shapePosition.x--;
        drawShape(currentShape, shapePosition.x, shapePosition.y);
      }
    };

    const moveRight = () => {
      if (canMoveRight(currentShape, shapePosition.x, shapePosition.y)) {
        clearShape(currentShape, shapePosition.x, shapePosition.y);
        shapePosition.x++;
        drawShape(currentShape, shapePosition.x, shapePosition.y);
      }
    };

    const moveDown = () => {
      // 检查下一个位置是否会碰撞
      if (!checkCollision(currentShape, shapePosition.x, shapePosition.y + 1)) {
        // 可以继续下落
        clearShape(currentShape, shapePosition.x, shapePosition.y);
        shapePosition.y++;
        drawShape(currentShape, shapePosition.x, shapePosition.y);
        return true; // 返回 true 表示方块还在下落
      } else {
        // 方块已经到达底部或碰到其他方块
        // 固定当前方块
        drawShape(currentShape, shapePosition.x, shapePosition.y);
        // 检查是否可以消行
        checkLines();
        // 生成新方块
        spawnNewShape();
        return false; // 返回 false 表示方块已经固定
      }
    };

    const rotate = () => {
      const rotatedShape = transpose(currentShape).map(row => row.reverse());
      if (canRotate(rotatedShape, shapePosition.x, shapePosition.y)) {
        clearShape(currentShape, shapePosition.x, shapePosition.y);
        currentShape = rotatedShape;
        drawShape(currentShape, shapePosition.x, shapePosition.y);
      }
    };

    const transpose = matrix => matrix[0].map((_, colIndex) => matrix.map(row => row[colIndex]));

    const isValidPosition = (x, y) => x >= 0 && x < boardWidth && y >= 0 && y < boardHeight;

    const canMoveLeft = (shape, x, y) => {
      for (let rowIndex = 0; rowIndex < shape.length; rowIndex++) {
        for (let colIndex = 0; colIndex < shape[rowIndex].length; colIndex++) {
          if (shape[rowIndex][colIndex] && !isValidPosition(x + colIndex - 1, y + rowIndex)) {
            return false;
          }
        }
      }
      return true;
    };

    const canMoveRight = (shape, x, y) => {
      for (let rowIndex = 0; rowIndex < shape.length; rowIndex++) {
        for (let colIndex = 0; colIndex < shape[rowIndex].length; colIndex++) {
          if (shape[rowIndex][colIndex] && !isValidPosition(x + colIndex + 1, y + rowIndex)) {
            return false;
          }
        }
      }
      return true;
    };

    // eslint-disable-next-line no-unused-vars
    const canMoveDown = (shape, x, y) => {
      for (let rowIndex = 0; rowIndex < shape.length; rowIndex++) {
        for (let colIndex = 0; colIndex < shape[rowIndex].length; colIndex++) {
          if (shape[rowIndex][colIndex] && !isValidPosition(x + colIndex, y + rowIndex + 1)) {
            return false;
          }
        }
      }
      return true;
    };

    const canRotate = (shape, x, y) => {
      for (let rowIndex = 0; rowIndex < shape.length; rowIndex++) {
        for (let colIndex = 0; colIndex < shape[rowIndex].length; colIndex++) {
          if (shape[rowIndex][colIndex] && !isValidPosition(x + colIndex, y + rowIndex)) {
            return false;
          }
        }
      }
      return true;
    };

    const checkLines = () => {
      let linesCleared = 0;
      for (let y = boardHeight - 1; y >= 0; y--) {
        if (board.value[y].every(cell => cell)) {
          board.value.splice(y, 1);
          board.value.unshift(Array(boardWidth).fill(false));
          linesCleared++;
          y++;
        }
      }
      if (linesCleared > 0) {
        const points = [0, 100, 300, 500, 800][linesCleared];
        score.value += points * level.value/10;
        level.value = Math.floor(score.value / 1000) + 1;
        updateGameSpeed();
      }
    };

    const updateGameSpeed = () => {
      if (gameInterval) {
        clearInterval(gameInterval);
      }
      const speed = Math.max(100, 1000 - (level.value - 1) * 100);
      gameInterval = setInterval(() => {
        if (!isPaused.value && !isGameOver.value) {
          moveDown();
        }
      }, speed);
    };

    const checkCollision = (shape, x, y) => {
      for (let row = 0; row < shape.length; row++) {
        for (let col = 0; col < shape[row].length; col++) {
          if (shape[row][col]) {
            const newX = x + col;
            const newY = y + row;

            // 检查是否超出边界
            if (newX < 0 || newX >= boardWidth || newY >= boardHeight) {
              return true;
            }

            // 检查是否与其他方块重叠
            if (newY >= 0 && board.value[newY][newX] &&
                !isPartOfCurrentShape(newX, newY)) {
              return true;
            }
          }
        }
      }
      return false;
    };

    const isPartOfCurrentShape = (x, y) => {
      if (!currentShape) return false;

      for (let row = 0; row < currentShape.length; row++) {
        for (let col = 0; col < currentShape[row].length; col++) {
          if (currentShape[row][col] &&
              shapePosition.x + col === x &&
              shapePosition.y + row === y) {
            return true;
          }
        }
      }
      return false;
    };

    const hardDrop = () => {
      while (!checkCollision(currentShape, shapePosition.x, shapePosition.y + 1)) {
        clearShape(currentShape, shapePosition.x, shapePosition.y);
        shapePosition.y++;
      }
      // 绘制最终位置并固定方块
      drawShape(currentShape, shapePosition.x, shapePosition.y);
      checkLines();
      spawnNewShape();
    };

    const updateNextShapePreview = () => {
      nextShapeBoard.value = Array.from({ length: 4 }, () => Array(4).fill(false));
      nextShape.value.forEach((row, rowIndex) => {
        row.forEach((cell, colIndex) => {
          if (cell) {
            nextShapeBoard.value[rowIndex][colIndex] = true;
          }
        });
      });
    };

    const togglePause = () => {
      isPaused.value = !isPaused.value;
    };

    const startNewGame = () => {
      board.value = Array.from({ length: boardHeight }, () => Array(boardWidth).fill(false));
      score.value = 0;
      level.value = 1;
      isPaused.value = false;
      isGameOver.value = false;
      nextShape.value = randomShape();
      updateNextShapePreview();
      startGame();
    };

    const startGame = () => {
      shapePosition = { x: Math.floor(boardWidth / 2) - 2, y: 0 };
      // 初始化第一个方块
      currentShape = randomShape();
      // 生成下一个方块
      nextShape.value = randomShape();
      updateNextShapePreview();
      updateGameSpeed();
    };

    const handleKeydown = (event) => {
      if (isPaused.value || isGameOver.value) return;

      switch (event.key) {
        case 'ArrowLeft':
          moveLeft();
          break;
        case 'ArrowRight':
          moveRight();
          break;
        case 'ArrowDown':
          moveDown();
          break;
        case 'ArrowUp':
          rotate();
          break;
        case ' ': // 空格键
          hardDrop();
          break;
        case 'p':
          togglePause();
          break;
      }
    };

    const spawnNewShape = () => {
      // 如果没有下一个方块，先生成一个
      if (!nextShape.value) {
        nextShape.value = randomShape();
      }

      // 设置新的当前方块
      currentShape = nextShape.value;
      shapePosition = { x: Math.floor(boardWidth / 2) - 2, y: 0 };

      // 生成下一个方块
      nextShape.value = randomShape();
      updateNextShapePreview();

      // 检查游戏是否结束
      if (checkCollision(currentShape, shapePosition.x, shapePosition.y)) {
        isGameOver.value = true;
        clearInterval(gameInterval);
        alert(`游戏结束！最终得分：${score.value}`);
        return;
      }

      // 立即绘制新方块
      drawShape(currentShape, shapePosition.x, shapePosition.y);
    };

    onMounted(() => {
      startGame();
      window.addEventListener('keydown', handleKeydown);
      // 阻止双击缩放
      document.addEventListener('dblclick', preventZoom, { passive: false });
      // 阻止多点触控缩放
      document.addEventListener('touchstart', preventZoom, { passive: false });
      document.addEventListener('touchmove', preventZoom, { passive: false });
    });

    onUnmounted(() => {
      if (gameInterval) {
        clearInterval(gameInterval);
      }
      window.removeEventListener('keydown', handleKeydown);
      // 移除事件监听器
      document.removeEventListener('dblclick', preventZoom);
      document.removeEventListener('touchstart', preventZoom);
      document.removeEventListener('touchmove', preventZoom);
    });

    const preventZoom = (event) => {
      if (event.touches && event.touches.length > 1) {
        event.preventDefault();
      }
    }

    return {
      board,
      score,
      level,
      isPaused,
      nextShapeBoard,
      moveLeft,
      moveRight,
      rotate,
      moveDown,
      hardDrop,
      togglePause,
      startNewGame,
    };
  },
};
</script>

<style>
/* 添加这些全局样式确保更好的响应式布局 */
html, body {
  margin: 0;
  padding: 0;
  overflow: hidden;
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  touch-action: manipulation;
}

*, *:before, *:after {
  box-sizing: inherit;
}

/* 禁用长按菜单 */
* {
  -webkit-touch-callout: none; /* iOS Safari */
  -webkit-user-select: none; /* Safari */
  -khtml-user-select: none; /* Konqueror HTML */
  -moz-user-select: none; /* Firefox */
  -ms-user-select: none; /* Internet Explorer/Edge */
  user-select: none; /* Non-prefixed version, currently supported by Chrome and Opera */
}

.tetris {
  display: flex;
  flex-direction: column;
  height: 100vh;
  width: 100vw;
  max-width: 100%;
  max-height: 100vh;
  overflow: hidden;
  background: #f0f0f0;
  font-family: 'PingFang SC', 'Helvetica Neue', Arial, sans-serif;
  margin: 0;
  padding: 0;
  position: relative; /* 改为相对定位而非绝对定位 */
}

.game-header {
  padding: min(15px, 2vh);
  text-align: center;
  background: #2c3e50;
  color: white;
  box-shadow: 0 2px 8px rgba(0,0,0,0.2);
  height: min(60px, 8vh); /* 固定头部高度 */
  display: flex;
  justify-content: center;
  align-items: center;
}

.score, .level {
  display: inline-block;
  margin: 0 20px;
  font-size: clamp(1rem, 2vw, 1.3em); /* 响应式字体大小 */
  font-weight: bold;
}

.game-container {
  display: flex;
  flex: 1;
  justify-content: space-between;
  height: calc(100vh - min(60px, 8vh)); /* 减去头部高度 */
  padding: min(10px, 1.5vw);
  box-sizing: border-box;
  position: relative; /* 确保子元素可以基于它定位 */
}

/* 左侧控制区 */
.left-controls {
  width: 25%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: min(15px, 2vw);
}

.direction-pad {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: repeat(3, 1fr);
  gap: min(16px, 2vw);
  width: min(360px, 25vw);
  height: min(360px, 25vw);
  max-width: 90%;
}

.dir-btn {
  width: 100%;
  height: 100%;
  font-size: min(56px, 4vw);
  border: none;
  border-radius: 50%;
  background: linear-gradient(135deg, #4CAF50, #2E7D32);
  color: white;
  touch-action: manipulation;
  box-shadow: 0 8px 16px rgba(0,0,0,0.2);
  transition: all 0.2s ease;
}

.dir-btn:active {
  transform: scale(0.8);
  box-shadow: 0 4px 8px rgba(0,0,0,0.2);
}

.left-btn { grid-column: 1; grid-row: 2; }
.up-btn { grid-column: 2; grid-row: 1; }
.down-btn { grid-column: 2; grid-row: 3; }
.right-btn { grid-column: 3; grid-row: 2; }

/* 游戏主区域 */
.board-container {
  flex: 0 0 50%;
  display: flex;
  justify-content: center;
  align-items: center;
  background: #343a40;
  padding: 0;
  border-radius: 10px;
  box-shadow: 0 5px 15px rgba(0,0,0,0.3);
  max-height: 95%; /* 确保不会溢出 */
  transform: none !important; /* 禁止缩放 */
}

.board {
  display: flex;
  flex-direction: column;
  background: #343a40;
  width: 100%;
  height: 100%;
  justify-content: space-between;
  padding: min(10px, 1.5vw);
  border-radius: 10px;
  aspect-ratio: 1/2; /* 保持俄罗斯方块传统的宽高比 */
  max-height: 95%;
  margin: auto;
}

.row {
  display: flex;
  flex: 1;
  justify-content: space-between;
}

.cell {
  flex: 1;
  aspect-ratio: 1;
  background-color: #f8f9fa;
  border-radius: 3px;
  margin: 1px;
  transition: background-color 0.1s ease;
}

.filled {
  background-color: #007bff;
  box-shadow: inset 0 0 5px rgba(0,0,0,0.3);
}

/* 右侧控制区 */
.right-controls {
  width: 25%;
  display: flex;
  flex-direction: column;
  justify-content: space-between; /* 改变为两端对齐 */
  align-items: center;
  padding: min(15px, 2vw);
  transform: none !important; /* 禁止缩放 */
}

.next-shape-container {
  text-align: center;
  background: #f8f9fa;
  border-radius: 10px;
  padding: min(15px, 2vw);
  box-shadow: 0 2px 10px rgba(0,0,0,0.05);
  width: 100%;
  margin-bottom: 0;
}

.next-shape-container h3 {
  margin-top: 0;
  color: #2c3e50;
  font-size: clamp(1rem, 1.5vw, 1.2em);
  margin-bottom: min(10px, 1.5vw);
}

.mini-board {
  display: flex;
  flex-direction: column;
  background: #ddd;
  padding: 8px;
  border-radius: 6px;
  margin: 0 auto;
  width: min(100px, 10vw);
  height: min(100px, 10vw);
  min-width: 60px;
  min-height: 60px;
}

.game-controls {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  gap: min(10px, 1.5vw);
  width: 100%;
  margin: min(20px, 3vh) 0;
}

.control-btn {
  flex: 1;
  padding: min(30px, 4vh) min(10px, 1.5vw);
  font-size: clamp(1rem, 1.8vw, 20px);
  border: none;
  border-radius: 25px;
  background: linear-gradient(135deg, #9C27B0, #673AB7);
  color: white;
  touch-action: manipulation;
  box-shadow: 0 5px 10px rgba(0,0,0,0.3);
  transition: all 0.2s ease;
  text-align: center;
  font-weight: bold;
}

.control-btn:active {
  transform: translateY(3px);
  box-shadow: 0 2px 5px rgba(0,0,0,0.2);
}

.action-buttons {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  gap: min(20px, 3vw);
  width: 100%;
  height: 50%;
  margin-top: auto; /* 将按钮移至底部 */
  margin-bottom: min(30px, 4vh);
}

.action-btn {
  width: min(120px, 10vw);
  height: min(120px, 10vw);
  min-width: 60px;
  min-height: 60px;
  font-size: clamp(0.8rem, 1.8vw, 20px);
  font-weight: bold;
  border: none;
  border-radius: 50%;
  color: white;
  touch-action: manipulation;
  box-shadow: 0 8px 16px rgba(0,0,0,0.3);
  transition: all 0.2s ease;
  display: flex;
  justify-content: center;
  align-items: center;
}

.drop-btn {
  background: linear-gradient(135deg, #2196F3, #0D47A1);
}

.rotate-btn {
  background: linear-gradient(135deg, #FF9800, #E65100);
}

.action-btn:active {
  transform: scale(0.95);
  box-shadow: 0 4px 8px rgba(0,0,0,0.2);
}

/* 移动设备适配 */
@media (max-width: 768px) {
  .game-container {
    flex-direction: column;
    height: auto;
    gap: min(20px, 3vh);
  }
  
  .left-controls,
  .board-container,
  .right-controls {
    width: 100%;
    margin-bottom: 0;
  }
  
  .left-controls {
    order: 2; /* 调整顺序 */
  }
  
  .board-container {
    order: 1; /* 让游戏区在最上面 */
    height: 50vh;
  }
  
  .right-controls {
    order: 3;
    flex-direction: row;
    flex-wrap: wrap;
    justify-content: center;
  }
  
  .next-shape-container {
    width: 40%;
    margin-right: 5%;
  }
  
  .game-controls {
    width: 55%;
    flex-direction: column;
    margin: 0;
  }
  
  .direction-pad {
    width: min(300px, 80vw);
    height: min(300px, 80vw);
  }
  
  .action-buttons {
    width: 100%;
    height: 50%;
    margin-top: min(20px, 3vh);
    justify-content: center;
  }
  
  .action-btn {
    width: min(100px, 30vw);
    height: min(100px, 30vw);
  }
}

/* 高度特别低的屏幕 */
@media (max-height: 500px) {
  .game-container {
    flex-direction: row;
  }
  
  .board {
    max-height: 100%;
  }
  
  .direction-pad {
    width: min(180px, 15vw);
    height: min(180px, 15vw);
  }
  
  .action-btn {
    width: min(80px, 8vw);
    height: min(80px, 8vw);
  }
  
  .control-btn {
    padding: min(15px, 3vh) min(10px, 1vw);
  }
}

/* 禁用双击缩放 */
html, body {
  touch-action: manipulation;
}

/* 添加这个专门针对华为折叠屏设备的媒体查询 */
@media screen and (min-width: 2000px) and (min-height: 2000px) {
  .tetris {
    height: 100vh; /* 确保不超出视口高度 */
    overflow: hidden;
  }
  
  .game-header {
    padding: 10px;
    height: 5vh; /* 减小头部高度 */
  }
  
  .game-container {
    height: 95vh; /* 增加容器高度 */
    padding: 5px; /* 减小内边距 */
  }
  
  .board-container {
    max-height: 90vh; /* 限制游戏区最大高度 */
  }
  
  .board {
    max-height: 88vh; /* 确保游戏板不会太大 */
    padding: 5px; /* 减小内边距 */
  }
  
  /* 调整控制区大小 */
  .direction-pad {
    width: min(300px, 18vw);
    height: min(300px, 18vw);
    gap: 10px; /* 减小间距 */
  }
  
  .action-btn {
    width: min(100px, 8vw);
    height: min(100px, 8vw);
  }
  
  /* 调整右侧控件布局 */
  .right-controls {
    justify-content: flex-start; /* 从顶部开始布局 */
    gap: 15px; /* 减小元素间距 */
  }
  
  .action-buttons {
    margin-top: 15px; /* 减小顶部边距 */
  }
  
  /* 缩小各种元素以适应屏幕 */
  .mini-board {
    width: min(80px, 8vw);
    height: min(80px, 8vw);
  }
  
  .control-btn {
    padding: 15px 8px;
  }
}

/* 针对华为 Mate X5 横屏模式的特别优化 */
@media screen and (min-width: 2400px) and (max-height: 2100px) {
  .game-container {
    flex-direction: row;
    height: calc(100vh - 5vh); /* 减去头部高度 */
  }
  
  .board-container {
    flex: 0 0 40%; /* 调整中间游戏区的宽度 */
  }
  
  .board {
    aspect-ratio: 1/2; /* 保持俄罗斯方块传统比例 */
    margin: auto;
    max-height: 85vh;
  }
  
  .left-controls, .right-controls {
    width: 20%;
  }
}

/* 针对华为 Mate X5 竖屏模式的特别优化 */
@media screen and (max-width: 2300px) and (min-height: 2200px) {
  .game-container {
    flex-direction: column;
    height: auto;
  }
  
  .board-container {
    height: 45vh; /* 确保游戏区域不会太高 */
    width: 90%;
    margin: 0 auto;
  }
  
  .left-controls, .right-controls {
    width: 100%;
    flex-direction: row;
    justify-content: space-around;
    padding: 10px 0;
  }
  
  .direction-pad {
    width: min(250px, 40vw);
    height: min(250px, 40vw);
  }
  
  .next-shape-container, .game-controls, .action-buttons {
    width: 30%;
  }
  
  .action-buttons {
    margin-top: 0;
  }
}

/* 通用响应式调整 */
.row, .cell {
  min-height: 2px; /* 确保单元格有最小高度 */
}

/* 针对非常高的设备，调整比例 */
@media screen and (max-aspect-ratio: 9/16) and (min-height: 2000px) {
  .board {
    aspect-ratio: 1/1.8; /* 稍微调整宽高比 */
    max-height: 70vh;
  }
}

/* 针对 iPad Air 2 横屏模式的左侧控制区优化 */
@media only screen 
  and (min-device-width: 2048px) 
  and (max-device-width: 2048px) 
  and (min-device-height: 1536px) 
  and (max-device-height: 1536px) 
  and (orientation: landscape) 
  and (-webkit-min-device-pixel-ratio: 2) {
  
  .left-controls {
    width: 20%; /* 缩小左侧控制区宽度 */
    transform: scale(0.8); /* 整体缩放 */
    transform-origin: left center; /* 从左侧中心缩放 */
  }

  .direction-pad {
    width: 70%; /* 调整方向键区域大小 */
    height: 70%;
    gap: 10px; /* 减小按钮间距 */
  }

  .dir-btn {
    font-size: 30px; /* 调整按钮字体大小 */
  }
}

/* 通用 iPad 横屏模式左侧控制区优化 */
@media only screen 
  and (min-device-width: 768px) 
  and (max-device-width: 1024px) 
  and (orientation: landscape) 
  and (-webkit-min-device-pixel-ratio: 2) {
  
  .left-controls {
    width: 20%;
    transform: scale(0.6);
    transform-origin: left center;
  }

  .direction-pad {
    width: 85%;
    height: 85%;
    gap: 12px;
  }

  .dir-btn {
    font-size: 36px;
  }
}
</style>

<template>
  <div class="game">
    <div class="grid">
      <div v-for="(row, rowIndex) in grid" :key="'row-' + rowIndex" class="grid-row">
        <div
          v-for="(tile, colIndex) in row"
          :key="'tile-' + rowIndex + '-' + colIndex"
          class="tile"
          :class="'tile-' + tile"
        >
          <span v-if="tile">{{ tile }}</span>
        </div>
      </div>
    </div>
    <div class="controls">
      <button @click="startNewGame">New Game</button>
      <div class="score">Score: {{ score }}</div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      grid: this.createEmptyGrid(),
      score: 0,
      gameOver: false,
    };
  },
  methods: {
    createEmptyGrid() {
      return Array(4).fill().map(() => Array(4).fill(0));
    },
    startNewGame() {
      this.grid = this.createEmptyGrid();
      this.score = 0;
      this.addRandomTile();
      this.addRandomTile();
    },
    addRandomTile() {
      const emptyTiles = [];
      for (let r = 0; r < 4; r++) {
        for (let c = 0; c < 4; c++) {
          if (this.grid[r][c] === 0) {
            emptyTiles.push({ row: r, col: c });
          }
        }
      }
      if (emptyTiles.length > 0) {
        const randomIndex = Math.floor(Math.random() * emptyTiles.length);
        const { row, col } = emptyTiles[randomIndex];
        this.grid[row][col] = Math.random() < 0.9 ? 2 : 4;
      }
    },
    move(direction) {
  if (this.gameOver) return;

  let moved = false;
  const oldGrid = JSON.parse(JSON.stringify(this.grid));

  switch (direction) {
    case 'left':
      for (let r = 0; r < 4; r++) {
        moved = this.moveRowLeft(r) || moved;
      }
      break;
    case 'right':
      for (let r = 0; r < 4; r++) {
        moved = this.moveRowRight(r) || moved;
      }
      break;
    case 'up':
      for (let c = 0; c < 4; c++) {
        moved = this.moveColUp(c) || moved;
      }
      break;
    case 'down':
      for (let c = 0; c < 4; c++) {
        moved = this.moveColDown(c) || moved;
      }
      break;
    default:
      console.error(`Invalid direction: ${direction}`);
  }

  // 如果发生了任何变化，才生成一个新的随机方块
  if (moved) {
    this.addRandomTile(); // 确保只调用一次
    this.checkGameOver();
    this.updateScore();
  }
},

    moveRowLeft(row) {
      let moved = false;

      if (!Array.isArray(this.grid[row])) {
        console.error(`Invalid row data: ${this.grid[row]}`);
        return moved;
      }

      let newRow = this.grid[row].filter(val => val !== 0); // 去除零元素
      console.log(`Row before moveLeft: ${newRow}`);

      for (let i = 0; i < newRow.length - 1; i++) {
        if (newRow[i] === newRow[i + 1]) {
          newRow[i] *= 2;
          this.score += newRow[i]; // 增加分数
          newRow[i + 1] = 0; // 合并后的下一个方块设为0
          moved = true;
          console.log(`Merged ${newRow[i]} at index ${i}`);
        }
      }

      newRow = newRow.filter(val => val !== 0); // 清除合并后多出的零
      while (newRow.length < 4) {
        newRow.push(0); // 补充零元素
      }

      console.log(`Row after moveLeft: ${newRow}`);

      // 如果有变化，更新 moved 标志
      if (!this.arraysAreEqual(this.grid[row], newRow)) {
        this.grid[row] = newRow;
        moved = true;
      }

      return moved;
    },

    moveRowRight(row) {
      if (!this.grid[row]) {
        console.error(`Invalid row data: ${this.grid[row]}`);
        return false;
      }
      this.grid[row].reverse();
      let moved = this.moveRowLeft(row);
      this.grid[row].reverse();
      return moved;
    },

    arraysAreEqual(arr1, arr2) {
      return arr1.length === arr2.length && arr1.every((val, index) => val === arr2[index]);
    },

    moveColUp(col) {
  let moved = false;

  // 抽取列并去除零元素
  let column = this.grid.map(row => row[col]).filter(val => val !== 0);

  // 合并方块
  for (let i = 0; i < column.length - 1; i++) {
    if (column[i] === column[i + 1]) {
      column[i] *= 2;
      this.score += column[i]; // 增加分数
      column[i + 1] = 0; // 合并后的下一个方块设为0
      moved = true;
    }
  }

  // 去除合并后的零元素
  column = column.filter(val => val !== 0);

  // 补充零元素，直到列的长度为4
  while (column.length < 4) {
    column.push(0);
  }

  // 更新 grid 中对应列的值
  for (let r = 0; r < 4; r++) {
    if (this.grid[r][col] !== column[r]) {
      moved = true; // 只要位置发生变化，设置 moved 为 true
    }
    this.grid[r][col] = column[r];
  }

  // 只在发生了移动后才生成一个新的随机方块
  return moved;
},

moveColDown(col) {
  let moved = false;

  // 抽取列并去除零元素
  let column = this.grid.map(row => row[col]).filter(val => val !== 0);

  // 反转列并合并方块
  column.reverse();
  for (let i = 0; i < column.length - 1; i++) {
    if (column[i] === column[i + 1]) {
      column[i] *= 2;
      this.score += column[i]; // 增加分数
      column[i + 1] = 0; // 合并后的下一个方块设为0
      moved = true;
    }
  }

  // 去除合并后的零元素
  column = column.filter(val => val !== 0);

  // 补充零元素，直到列的长度为4
  while (column.length < 4) {
    column.push(0);
  }

  // 再反转回来
  column.reverse();

  // 更新 grid 中对应列的值
  for (let r = 0; r < 4; r++) {
    if (this.grid[r][col] !== column[r]) {
      moved = true; // 只要位置发生变化，设置 moved 为 true
    }
    this.grid[r][col] = column[r];
  }

  return moved;
},

    checkGameOver() {
      let availableMove = false;
      for (let r = 0; r < 4; r++) {
        for (let c = 0; c < 4; c++) {
          if (this.grid[r][c] === 0) {
            availableMove = true;
            break;
          }
          if (r < 3 && this.grid[r][c] === this.grid[r + 1][c]) {
            availableMove = true;
            break;
          }
          if (c < 3 && this.grid[r][c] === this.grid[r][c + 1]) {
            availableMove = true;
            break;
          }
        }
      }
      if (!availableMove) {
        this.gameOver = true;
      }
    },

    updateScore() {
      this.score = this.grid.flat().reduce((acc, val) => acc + val, 0);
    },
  },

  mounted() {
    this.startNewGame();
    window.addEventListener('keydown', (e) => {
      e.preventDefault();

      switch (e.key) {
        case 'a':
          this.move('left');
          break;
        case 'd':
          this.move('right');
          break;
        case 'w':
          this.move('up');
          break;
        case 's':
          this.move('down');
          break;
        default:
          console.error(`Invalid key: ${e.key}`);
      }
    });
  }
};
</script>

<style scoped>
.game {
  max-width: 420px;
  margin: 50px auto;
}

.grid {
  display: flex;
  flex-wrap: wrap;
}

.grid-row {
  display: flex;
}

.tile {
  width: 100px;
  height: 100px;
  display: inline-block;
  margin: 10px;
  text-align: center;
  line-height: 100px;
  border-radius: 10px;
  background-color: #ccc;
  font-size: 2rem;
  font-weight: bold;
  color: white;
  transition: transform 0.2s ease;
}

.tile-2 {
  background-color: #eee4da;
}
.tile-4 {
  background-color: #ede0c8;
}
.tile-8 {
  background-color: #f2b179;
}
.tile-16 {
  background-color: #f59563;
}
.tile-32 {
  background-color: #f67c5f;
}
.tile-64 {
  background-color: #f65e3b;
}
.tile-128 {
  background-color: #edcf72;
}
.tile-256 {
  background-color: #edcc61;
}
.tile-512 {
  background-color: #edc850;
}
.tile-1024 {
  background-color: #edc53f;
}
.tile-2048 {
  background-color: #edc22e;
}
.tile-3072 {
  background-color: #b97b44;
}

.controls {
  display: flex;
  justify-content: space-between;
  margin-top: 20px;
}

button {
  padding: 10px 20px;
  font-size: 16px;
}

.score {
  font-size: 18px;
}
</style>
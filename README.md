<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>最適化版テトリス</title>
  <style>
    body {
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      margin: 0;
      background: #1a1a1a;
      font-family: Arial, sans-serif;
    }

    .game-container {
      display: flex;
      gap: 15px;
      padding: 10px;
      background: #2a2a2a;
      border-radius: 8px;
    }

    .game-board {
      display: grid;
      grid-template: repeat(20, 20px) / repeat(10, 20px);
      gap: 1px;
      background: #000;
      border: 2px solid #444;
    }

    .cell {
      width: 20px;
      height: 20px;
      background: rgba(255, 255, 255, 0.1);
    }

    .preview {
      opacity: 0.3 !important;
    }

    .info-panel {
      display: flex;
      flex-direction: column;
      gap: 10px;
      color: #fff;
    }

    .next-block {
      display: grid;
      grid-template: repeat(4, 15px) / repeat(4, 15px);
      gap: 1px;
      background: #000;
      padding: 5px;
    }

    .score {
      font-size: 18px;
    }

    .level {
      color: #ff9900;
    }

    .controls {
      margin-top: 10px;
      display: grid;
      grid-template-columns: repeat(2, 1fr);
      gap: 5px;
    }

    button {
      padding: 6px 12px;
      background: #444;
      border: none;
      color: white;
      cursor: pointer;
      border-radius: 4px;
    }

    button:hover {
      background: #555;
    }
  </style>
</head>
<body>
  <div class="game-container">
    <div class="game-board" id="board"></div>
    <div class="info-panel">
      <div class="score">SCORE: <span id="score">0</span></div>
      <div class="level">LEVEL: <span id="level">1</span></div>
      <div>NEXT:</div>
      <div class="next-block" id="next-block"></div>
      <div class="controls">
        <button id="left">←</button>
        <button id="right">→</button>
        <button id="rotate">↻</button>
        <button id="drop">↓</button>
        <button id="pause" style="grid-column: span 2;">⏸ PAUSE</button>
      </div>
    </div>
  </div>

<script>
const shapes = [
  [[1,1,1,1]], // I
  [[1,1,1],[0,1,0]], // T
  [[1,1],[1,1]], // O
  [[1,1,0],[0,1,1]], // Z
  [[0,1,1],[1,1,0]], // S
  [[1,0],[1,0],[1,1]], // L
  [[0,1],[0,1],[1,1]] // J
];

const colors = ['#00f0f0', '#f0a000', '#f0f000', '#00f000', '#f00000', '#a000f0', '#0000f0'];
let board = Array(20).fill().map(() => Array(10).fill(0));
let current = null, next = null;
let score = 0, level = 1, lines = 0, isPaused = false;
let lastUpdate = 0, dropInterval = 1000;

// パフォーマンス改善のための最適化
const boardCells = Array(200).fill().map((_, i) => {
  const cell = document.createElement('div');
  cell.className = 'cell';
  document.getElementById('board').appendChild(cell);
  return cell;
});

function createPiece() {
  if (!next) next = generateNewPiece();
  const piece = next;
  next = generateNewPiece();
  return piece;
}

function generateNewPiece() {
  const index = Math.floor(Math.random() * shapes.length);
  return {
    shape: shapes[index],
    color: colors[index],
    x: 3,
    y: 0
  };
}

function updateBoard() {
  // ボード全体をクリア
  boardCells.forEach(cell => {
    cell.style.background = '';
    cell.classList.remove('preview');
  });

  // 固定ブロックを描画
  board.forEach((row, y) => {
    row.forEach((color, x) => {
      if (color) {
        boardCells[y * 10 + x].style.background = color;
      }
    });
  });

  // 現在のピースを描画
  if (current) {
    const previewY = calculatePreviewY();
    current.shape.forEach((row, dy) => {
      row.forEach((cell, dx) => {
        if (cell) {
          // プレビュー表示
          const previewIndex = (previewY + dy) * 10 + (current.x + dx);
          if (previewIndex >= 0 && previewIndex < 200) {
            boardCells[previewIndex].style.background = current.color;
            boardCells[previewIndex].classList.add('preview');
          }

          // 現在位置表示
          const currentIndex = (current.y + dy) * 10 + (current.x + dx);
          if (currentIndex >= 0 && currentIndex < 200) {
            boardCells[currentIndex].style.background = current.color;
          }
        }
      });
    });
  }
}

function calculatePreviewY() {
  let y = current.y;
  while (!checkCollision(current.x, y + 1, current.shape)) y++;
  return y;
}

function checkCollision(x, y, shape) {
  return shape.some((row, dy) =>
    row.some((cell, dx) => {
      const px = x + dx;
      const py = y + dy;
      return cell && (
        px < 0 || px >= 10 ||
        py >= 20 ||
        (py >= 0 && board[py][px])
      );
    })
  );
}

function rotatePiece() {
  const originalShape = current.shape;
  const rotated = current.shape[0].map((_, i) =>
    current.shape.map(row => row[i]).reverse()
  );
  
  // 回転時の壁抜け防止
  const offsets = [[0,0], [1,0], [-1,0], [2,0], [-2,0]];
  for (const [dx, dy] of offsets) {
    current.x += dx;
    current.shape = rotated;
    if (!checkCollision(current.x, current.y, rotated)) break;
    current.x -= dx;
    current.shape = originalShape;
  }
}

function hardDrop() {
  while (!checkCollision(current.x, current.y + 1, current.shape)) {
    current.y++;
  }
  lockPiece();
}

function lockPiece() {
  current.shape.forEach((row, dy) => {
    row.forEach((cell, dx) => {
      if (cell) {
        board[current.y + dy][current.x + dx] = current.color;
      }
    });
  });

  checkLines();
  current = createPiece();
  updateNext();
  
  if (checkCollision(current.x, current.y, current.shape)) {
    alert(`GAME OVER! Score: ${score}`);
    resetGame();
  }
}

function checkLines() {
  let cleared = 0;
  board = board.filter(row => {
    if (row.every(cell => cell)) {
      cleared++;
      return false;
    }
    return true;
  });

  while (board.length < 20) board.unshift(Array(10).fill(0));

  if (cleared > 0) {
    lines += cleared;
    score += [100, 300, 500, 800][cleared - 1] * level;
    updateLevel();
  }
}

function updateLevel() {
  level = Math.floor(lines / 5) + 1;
  dropInterval = Math.max(50, 1000 - (level * 100));
  document.getElementById('level').textContent = level;
  document.getElementById('score').textContent = score;
}

function updateNext() {
  const container = document.getElementById('next-block');
  container.innerHTML = '';
  
  const offsetX = Math.floor((4 - next.shape[0].length)/2);
  const offsetY = Math.floor((4 - next.shape.length)/2);
  
  for(let y = 0; y < 4; y++) {
    for(let x = 0; x < 4; x++) {
      const cell = document.createElement('div');
      cell.className = 'cell';
      if (next.shape[y - offsetY]?.[x - offsetX]) {
        cell.style.background = next.color;
      }
      container.appendChild(cell);
    }
  }
}

function resetGame() {
  board = Array(20).fill().map(() => Array(10).fill(0));
  score = 0;
  lines = 0;
  level = 1;
  current = createPiece();
  next = createPiece();
  updateNext();
  updateLevel();
  updateBoard();
}

// ゲームループ
function gameLoop(timestamp) {
  if (!isPaused) {
    if (timestamp - lastUpdate > dropInterval) {
      if (!checkCollision(current.x, current.y + 1, current.shape)) {
        current.y++;
      } else {
        lockPiece();
      }
      lastUpdate = timestamp;
      updateBoard();
    }
  }
  requestAnimationFrame(gameLoop);
}

// 初期化
resetGame();
requestAnimationFrame(gameLoop);

// 操作イベント
document.addEventListener('keydown', e => {
  if (isPaused) return;

  switch(e.key) {
    case 'ArrowLeft':
      current.x--;
      if (checkCollision(current.x, current.y, current.shape)) current.x++;
      break;
    case 'ArrowRight':
      current.x++;
      if (checkCollision(current.x, current.y, current.shape)) current.x--;
      break;
    case 'ArrowUp':
      rotatePiece();
      break;
    case 'ArrowDown':
      current.y++;
      if (checkCollision(current.x, current.y, current.shape)) current.y--;
      break;
    case ' ':
      hardDrop();
      break;
  }
  updateBoard();
});

document.getElementById('pause').addEventListener('click', () => {
  isPaused = !isPaused;
  document.getElementById('pause').textContent = isPaused ? '▶ PLAY' : '⏸ PAUSE';
});
</script>
</body>
</html>

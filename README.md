<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <!-- スマホでの拡大縮小やスクロールを防ぐためのviewport設定 -->
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>テトリス</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            /* モバイルでのスワイプによるリロードやスクロールを防止 */
            touch-action: none;
            overflow: hidden;
            font-family: 'Courier New', Courier, monospace;
            height: 100vh;
            height: 100dvh; /* モダンブラウザ用：URLバーを除いた実際の画面高さ */
            width: 100vw;
        }
        canvas {
            display: block;
            margin: 0 auto;
            background-color: #111;
            box-shadow: 0 0 20px rgba(0, 0, 0, 0.5);
            border: 2px solid #333;
        }
        .btn-control {
            /* タッチデバイス用のボタン装飾 */
            background: rgba(255, 255, 255, 0.1);
            border: 1px solid rgba(255, 255, 255, 0.2);
            border-radius: 8px;
            color: white;
            font-weight: bold;
            display: flex;
            justify-content: center;
            align-items: center;
            user-select: none;
            cursor: pointer;
        }
        .btn-control:active {
            background: rgba(255, 255, 255, 0.3);
        }
        #tetris {
            /* 画面の高さに合わせてキャンバスを自動縮小し、下のボタンが隠れないようにする */
            max-height: calc(100vh - 240px);
            max-height: calc(100dvh - 240px);
            width: auto;
        }
    </style>
</head>
<body class="bg-gray-900 text-white flex flex-col items-center justify-center">

    <div class="text-center mb-2">
        <h1 class="text-3xl font-bold tracking-widest text-blue-400">TETRIS</h1>
    </div>

    <!-- ゲームエリアのコンテナ -->
    <div class="flex flex-row items-start gap-4">
        
        <!-- 左側：スコアなどの情報 -->
        <div class="flex flex-col gap-4 w-24 sm:w-32">
            <div class="bg-gray-800 p-2 rounded border border-gray-700 text-center">
                <div class="text-xs text-gray-400">SCORE</div>
                <div id="score" class="text-lg sm:text-xl font-bold">0</div>
            </div>
            <div class="bg-gray-800 p-2 rounded border border-gray-700 text-center">
                <div class="text-xs text-gray-400">LINES</div>
                <div id="lines" class="text-lg sm:text-xl font-bold">0</div>
            </div>
            <div class="bg-gray-800 p-2 rounded border border-gray-700 text-center">
                <div class="text-xs text-gray-400">LEVEL</div>
                <div id="level" class="text-lg sm:text-xl font-bold">1</div>
            </div>
        </div>

        <!-- 中央：メインキャンバス -->
        <div class="relative">
            <canvas id="tetris" width="300" height="600"></canvas>
            
            <!-- ゲームオーバー/スタート画面のオーバーレイ -->
            <div id="overlay" class="absolute inset-0 bg-black/75 flex flex-col items-center justify-center z-10">
                <h2 id="overlay-msg" class="text-2xl font-bold mb-4 text-red-500">ゲームオーバー</h2>
                <button id="start-btn" class="px-6 py-2 bg-blue-600 hover:bg-blue-500 rounded text-white font-bold transition">
                    スタート
                </button>
            </div>
        </div>

        <!-- 右側：ネクストブロック -->
        <div class="flex flex-col gap-4 w-24 sm:w-32">
            <div class="bg-gray-800 p-2 rounded border border-gray-700 flex flex-col items-center">
                <div class="text-xs text-gray-400 mb-2">NEXT</div>
                <canvas id="next" width="120" height="120" class="border-none bg-transparent shadow-none"></canvas>
            </div>
            
            <!-- 操作説明 (PC用) -->
            <div class="hidden sm:block text-xs text-gray-400 mt-4 leading-relaxed">
                [←][→]: 移動<br>
                [↑]: 回転<br>
                [↓]: ソフトドロップ<br>
                [Space]: ハードドロップ
            </div>
        </div>
    </div>

    <!-- モバイル用コントロールパネル (PCでは非表示) -->
    <div class="sm:hidden grid grid-cols-3 gap-2 mt-6 w-full max-w-[360px] px-4">
        <div class="btn-control h-12 col-start-1" id="btn-left">←</div>
        <div class="btn-control h-12 col-start-2" id="btn-rotate">↻</div>
        <div class="btn-control h-12 col-start-3" id="btn-right">→</div>
        <div class="btn-control h-12 col-start-1 col-span-2" id="btn-down">↓ (Soft)</div>
        <div class="btn-control h-12 col-start-3 bg-red-900/50" id="btn-drop">⇓ (Hard)</div>
    </div>

    <script>
        // キャンバスとコンテキストの取得
        const canvas = document.getElementById('tetris');
        const ctx = canvas.getContext('2d');
        const nextCanvas = document.getElementById('next');
        const nextCtx = nextCanvas.getContext('2d');

        // ゲームの定数
        const COLS = 10;
        const ROWS = 20;
        const BLOCK_SIZE = 30; // 300 / 10 = 30px per block

        // キャンバスのスケール調整（CSSサイズではなく実際の描画サイズ）
        ctx.scale(BLOCK_SIZE, BLOCK_SIZE);
        nextCtx.scale(BLOCK_SIZE, BLOCK_SIZE);

        // UI要素の取得
        const scoreEl = document.getElementById('score');
        const linesEl = document.getElementById('lines');
        const levelEl = document.getElementById('level');
        const overlay = document.getElementById('overlay');
        const overlayMsg = document.getElementById('overlay-msg');
        const startBtn = document.getElementById('start-btn');

        // テトリミノの色
        const COLORS = [
            null,
            '#00FFFF', // I - Cyan
            '#0000FF', // J - Blue
            '#FFA500', // L - Orange
            '#FFFF00', // O - Yellow
            '#00FF00', // S - Green
            '#800080', // T - Purple
            '#FF0000', // Z - Red
            '#FF69B4', // 8: 十字 - Pink
            '#8B4513', // 9: U字 - Brown
            '#00FA9A', // 10: W字 - MediumSpringGreen
            '#FFD700', // 11: 小さいL - Gold
            '#EEEEEE'  // 12: 5マス棒 - LightGray
        ];

        // テトリミノの形状（行列）
        const PIECES = [
            [], // 0は空
            // 1: I
            [
                [0, 1, 0, 0],
                [0, 1, 0, 0],
                [0, 1, 0, 0],
                [0, 1, 0, 0]
            ],
            // 2: J
            [
                [0, 2, 0],
                [0, 2, 0],
                [2, 2, 0]
            ],
            // 3: L
            [
                [0, 3, 0],
                [0, 3, 0],
                [0, 3, 3]
            ],
            // 4: O
            [
                [4, 4],
                [4, 4]
            ],
            // 5: S
            [
                [0, 5, 5],
                [5, 5, 0],
                [0, 0, 0]
            ],
            // 6: T
            [
                [0, 0, 0],
                [6, 6, 6],
                [0, 6, 0]
            ],
            // 7: Z
            [
                [7, 7, 0],
                [0, 7, 7],
                [0, 0, 0]
            ],
            // 8: 十字
            [
                [0, 8, 0],
                [8, 8, 8],
                [0, 8, 0]
            ],
            // 9: U字
            [
                [9, 0, 9],
                [9, 9, 9],
                [0, 0, 0]
            ],
            // 10: W字
            [
                [10,  0,  0],
                [10, 10,  0],
                [ 0, 10, 10]
            ],
            // 11: 小さいL(3マス)
            [
                [11, 11, 0],
                [11,  0, 0],
                [ 0,  0, 0]
            ],
            // 12: 5マスの長い棒
            [
                [0, 0, 12, 0, 0],
                [0, 0, 12, 0, 0],
                [0, 0, 12, 0, 0],
                [0, 0, 12, 0, 0],
                [0, 0, 12, 0, 0]
            ]
        ];

        // ゲーム状態変数
        let board = [];
        let player = { pos: {x: 0, y: 0}, matrix: null };
        let nextPiece = null;
        let dropCounter = 0;
        let dropInterval = 1000;
        let lastTime = 0;
        let score = 0;
        let lines = 0;
        let level = 1;
        let isGameOver = true;
        let animationId;

        // 行列（ボード）の作成
        function createMatrix(w, h) {
            const matrix = [];
            while (h--) {
                matrix.push(new Array(w).fill(0));
            }
            return matrix;
        }

        // ブロックの描画ヘルパー関数
        function drawMatrix(matrix, offset, context, isGhost = false) {
            matrix.forEach((row, y) => {
                row.forEach((value, x) => {
                    if (value !== 0) {
                        context.fillStyle = COLORS[value];
                        if (isGhost) {
                            // ゴーストブロックは半透明で枠線のみのような見た目に
                            context.globalAlpha = 0.3;
                            context.fillRect(x + offset.x, y + offset.y, 1, 1);
                            context.globalAlpha = 1.0;
                            context.lineWidth = 0.05;
                            context.strokeStyle = COLORS[value];
                            context.strokeRect(x + offset.x, y + offset.y, 1, 1);
                        } else {
                            // 通常のブロック
                            context.fillRect(x + offset.x, y + offset.y, 1, 1);
                            // 立体感を出すためのハイライトとシャドウ
                            context.fillStyle = 'rgba(255,255,255,0.3)';
                            context.fillRect(x + offset.x, y + offset.y, 1, 0.1);
                            context.fillRect(x + offset.x, y + offset.y, 0.1, 1);
                            context.fillStyle = 'rgba(0,0,0,0.3)';
                            context.fillRect(x + offset.x, y + offset.y + 0.9, 1, 0.1);
                            context.fillRect(x + offset.x + 0.9, y + offset.y, 0.1, 1);
                        }
                    }
                });
            });
        }

        // ランダムなテトリミノを生成
        function createRandomPiece() {
            // 元の7種類に加えて、特殊な形のミノを5種類追加（計12種類）
            const randomType = Math.floor(Math.random() * 12) + 1;
            return JSON.parse(JSON.stringify(PIECES[randomType]));
        }

        // プレイヤーのリセット（新しいブロックのセット）
        function playerReset() {
            if (!nextPiece) {
                nextPiece = createRandomPiece();
            }
            player.matrix = nextPiece;
            nextPiece = createRandomPiece();
            
            // 中央上部に配置
            player.pos.y = 0;
            player.pos.x = Math.floor(COLS / 2) - Math.floor(player.matrix[0].length / 2);

            // 配置した瞬間に衝突したらゲームオーバー
            if (collide(board, player)) {
                gameOver();
            }
            drawNextPiece();
        }

        // 次のブロックを描画
        function drawNextPiece() {
            nextCtx.fillStyle = '#111'; // 背景クリア
            nextCtx.fillRect(0, 0, nextCanvas.width, nextCanvas.height);
            
            // 中央に描画するためのオフセット計算
            const offsetX = (nextCanvas.width / BLOCK_SIZE - nextPiece[0].length) / 2;
            const offsetY = (nextCanvas.height / BLOCK_SIZE - nextPiece.length) / 2;
            
            drawMatrix(nextPiece, {x: offsetX, y: offsetY}, nextCtx);
        }

        // 衝突判定
        function collide(board, player) {
            const m = player.matrix;
            const o = player.pos;
            for (let y = 0; y < m.length; ++y) {
                for (let x = 0; x < m[y].length; ++x) {
                    if (m[y][x] !== 0 &&
                       (board[y + o.y] && board[y + o.y][x + o.x]) !== 0) {
                        return true;
                    }
                }
            }
            return false;
        }

        // ボードへの結合
        function merge(board, player) {
            player.matrix.forEach((row, y) => {
                row.forEach((value, x) => {
                    if (value !== 0) {
                        board[y + player.pos.y][x + player.pos.x] = value;
                    }
                });
            });
        }

        // 回転の実行
        function rotate(matrix, dir) {
            // 転置
            for (let y = 0; y < matrix.length; ++y) {
                for (let x = 0; x < y; ++x) {
                    [matrix[x][y], matrix[y][x]] = [matrix[y][x], matrix[x][y]];
                }
            }
            // 反転
            if (dir > 0) {
                matrix.forEach(row => row.reverse());
            } else {
                matrix.reverse();
            }
        }

        // プレイヤーの回転（壁蹴り対応）
        function playerRotate(dir) {
            const pos = player.pos.x;
            let offset = 1;
            rotate(player.matrix, dir);
            
            // 回転した結果、壁やブロックにめり込んだ場合の補正（壁蹴り）
            while (collide(board, player)) {
                player.pos.x += offset;
                offset = -(offset + (offset > 0 ? 1 : -1));
                if (offset > player.matrix[0].length) {
                    // 補正しきれない場合は回転を元に戻す
                    rotate(player.matrix, -dir);
                    player.pos.x = pos;
                    return;
                }
            }
        }

        // プレイヤーの移動
        function playerMove(offset) {
            player.pos.x += offset;
            if (collide(board, player)) {
                player.pos.x -= offset;
            }
        }

        // プレイヤーの落下
        function playerDrop() {
            player.pos.y++;
            if (collide(board, player)) {
                player.pos.y--;
                merge(board, player);
                playerReset();
                sweepLines();
                updateScore();
            }
            dropCounter = 0;
        }

        // ハードドロップ（一気に下まで落とす）
        function playerHardDrop() {
            while (!collide(board, player)) {
                player.pos.y++;
                score += 2; // ハードドロップのボーナススコア
            }
            player.pos.y--;
            merge(board, player);
            playerReset();
            sweepLines();
            updateScore();
            dropCounter = 0;
        }

        // ゴーストブロックの落下位置を計算
        function getGhostPos() {
            const ghostPos = { x: player.pos.x, y: player.pos.y };
            const ghostPlayer = { matrix: player.matrix, pos: ghostPos };
            
            while (!collide(board, ghostPlayer)) {
                ghostPlayer.pos.y++;
            }
            ghostPlayer.pos.y--;
            return ghostPlayer.pos;
        }

        // ラインが揃ったかチェックして消去
        function sweepLines() {
            let linesCleared = 0;
            outer: for (let y = board.length - 1; y >= 0; --y) {
                for (let x = 0; x < board[y].length; ++x) {
                    if (board[y][x] === 0) {
                        continue outer; // 0が含まれていれば次の行へ
                    }
                }
                
                // 行を削除して一番上に空の行を追加
                const row = board.splice(y, 1)[0].fill(0);
                board.unshift(row);
                ++y; // 同じyインデックスを再チェックするため
                linesCleared++;
            }

            if (linesCleared > 0) {
                // スコア計算（テトリス本来のスコアシステムに近似）
                const lineScores = [0, 40, 100, 300, 1200];
                score += lineScores[linesCleared] * level;
                lines += linesCleared;
                level = Math.floor(lines / 10) + 1;
                // レベルアップでスピードアップ（最大速度制限あり）
                dropInterval = Math.max(100, 1000 - (level - 1) * 100);
            }
        }

        // スコアUIの更新
        function updateScore() {
            scoreEl.innerText = score;
            linesEl.innerText = lines;
            levelEl.innerText = level;
        }

        // 全体の描画処理
        function draw() {
            // 背景クリア
            ctx.fillStyle = '#111';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // ボードの描画
            drawMatrix(board, {x: 0, y: 0}, ctx);

            // ブロックが存在する場合のみプレイヤーブロックとゴーストを描画
            if (player.matrix) {
                // ゴーストブロックの描画
                const ghostPos = getGhostPos();
                drawMatrix(player.matrix, ghostPos, ctx, true);

                // プレイヤーブロックの描画
                drawMatrix(player.matrix, player.pos, ctx);
            }
        }

        // メインループ
        function update(time = 0) {
            if (isGameOver) return;

            const deltaTime = time - lastTime;
            lastTime = time;
            dropCounter += deltaTime;

            if (dropCounter > dropInterval) {
                playerDrop();
            }

            draw();
            animationId = requestAnimationFrame(update);
        }

        // ゲームオーバー処理
        function gameOver() {
            isGameOver = true;
            cancelAnimationFrame(animationId);
            overlayMsg.innerText = "ゲームオーバー";
            overlay.classList.remove('hidden');
        }

        // ゲーム開始・リセット処理
        function startGame() {
            board = createMatrix(COLS, ROWS);
            score = 0;
            lines = 0;
            level = 1;
            dropInterval = 1000;
            updateScore();
            nextPiece = null;
            playerReset();
            
            isGameOver = false;
            overlay.classList.add('hidden');
            
            lastTime = performance.now();
            update();
        }

        // --- 入力制御 --- //

        // キーボード操作
        document.addEventListener('keydown', event => {
            if (isGameOver) return;
            
            switch(event.keyCode) {
                case 37: // Left
                    playerMove(-1);
                    break;
                case 39: // Right
                    playerMove(1);
                    break;
                case 40: // Down
                    playerDrop();
                    score += 1; // ソフトドロップボーナス
                    updateScore();
                    break;
                case 38: // Up
                    playerRotate(1);
                    break;
                case 32: // Space
                    playerHardDrop();
                    break;
            }
        });

        // モバイルタッチ操作用関数
        function addTouchControl(id, action) {
            const btn = document.getElementById(id);
            if (!btn) return;

            // 連打や長押しに対応するための変数
            let intervalId;

            const startAction = (e) => {
                e.preventDefault(); // スクロール等のデフォルト動作を防止
                if (isGameOver) return;
                action();
                
                // 左右と下移動は長押しで連続入力
                if (id === 'btn-left' || id === 'btn-right' || id === 'btn-down') {
                    // 最初は少し遅延させてから連続入力を開始する (DAS: Delayed Auto Shift)
                    setTimeout(() => {
                        intervalId = setInterval(() => {
                            if (!isGameOver) action();
                        }, 50); // 連続入力の間隔
                    }, 200); 
                }
            };

            const stopAction = (e) => {
                e.preventDefault();
                clearInterval(intervalId);
            };

            btn.addEventListener('touchstart', startAction, {passive: false});
            btn.addEventListener('touchend', stopAction, {passive: false});
            btn.addEventListener('touchcancel', stopAction, {passive: false});
            // マウス操作も対応させておく
            btn.addEventListener('mousedown', startAction);
            btn.addEventListener('mouseup', stopAction);
            btn.addEventListener('mouseleave', stopAction);
        }

        // タッチボタンのバインド
        addTouchControl('btn-left', () => playerMove(-1));
        addTouchControl('btn-right', () => playerMove(1));
        addTouchControl('btn-down', () => { playerDrop(); score++; updateScore(); });
        addTouchControl('btn-rotate', () => playerRotate(1));
        addTouchControl('btn-drop', () => playerHardDrop());

        // スタートボタン
        startBtn.addEventListener('click', startGame);

        // 初期表示用（まだ始まっていない状態）
        overlayMsg.innerText = "TETRIS";
        overlayMsg.classList.replace('text-red-500', 'text-blue-400');
        board = createMatrix(COLS, ROWS);
        draw();

    </script>
</body>
</html>

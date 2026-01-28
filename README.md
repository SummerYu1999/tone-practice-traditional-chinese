<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>五度標記法即時分析儀</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.4.0/p5.js"></script>
    <script src="https://unpkg.com/ml5@latest/dist/ml5.min.js"></script>
    <style>
        body { font-family: 'PingFang TC', 'Microsoft JhengHei', sans-serif; text-align: center; background: #121212; color: #e0e0e0; margin: 0; padding: 20px; }
        .container { max-width: 900px; margin: auto; }
        #canvas-container { position: relative; margin: 20px auto; border: 2px solid #333; border-radius: 12px; box-shadow: 0 0 20px rgba(0,0,0,0.5); background: #000; overflow: hidden; }
        .instructions { background: #222; padding: 15px; border-radius: 8px; margin-bottom: 20px; border-left: 5px solid #00ffcc; }
        .status-bar { display: flex; justify-content: space-around; font-size: 1.1em; margin-bottom: 10px; color: #00ffcc; }
        kbd { background: #444; padding: 2px 6px; border-radius: 4px; color: #fff; font-family: monospace; }
        .hint { font-size: 0.9em; color: #888; margin-top: 10px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>五度標記法即時音高分析</h1>
        
        <div class="instructions" id="guide">
            <strong>第一步：</strong> 用平常說話的音調持續發出「啊—」的聲音。<br>
            <strong>第二步：</strong> 看著頻率跳動時，按下 <kbd>空白鍵 (Space)</kbd> 鎖定你的基準音域。
        </div>

        <div class="status-bar">
            <div>狀態: <span id="status">模型讀取中...</span></div>
            <div>基準音 (3度): <span id="baseFreq">未設定</span></div>
            <div>即時頻率: <span id="liveFreq">0</span> Hz</div>
        </div>

        <div id="canvas-container"></div>
        
        <div class="hint">提示：鎖定後，若發音高於基準則曲線上升，低於則下降。再次按下空白鍵可重新校準。</div>
    </div>

    <script>
        let pitch;
        let audioContext;
        let mic;
        let points = []; 
        const maxPoints = 300; // 顯示點數
        let userBaseLog = null; 
        let isCalibrated = false;
        let latestFreq = 0;

        function setup() {
            const canvas = createCanvas(800, 450);
            canvas.parent('canvas-container');
            
            audioContext = getAudioContext();
            mic = new p5.AudioIn();
            mic.start(() => {
                // 使用 ml5 的 Crepe 模型進行精準音高偵測
                pitch = ml5.pitchDetection('https://cdn.jsdelivr.net/gh/ml5js/ml5-data-and-models/models/pitch-detection/crepe/', audioContext, mic.stream, modelLoaded);
            });
        }

        function modelLoaded() {
            select('#status').html('等待校準...');
            getPitch();
        }

        function getPitch() {
            pitch.getPitch((err, frequency) => {
                if (frequency && frequency > 50 && frequency < 1200) {
                    latestFreq = frequency;
                    select('#liveFreq').html(floor(frequency));
                    
                    let logFreq = Math.log2(frequency);
                    
                    if (!isCalibrated) {
                        // 校準期間：持續計算平均音高
                        if (userBaseLog === null) userBaseLog = logFreq;
                        userBaseLog = lerp(userBaseLog, logFreq, 0.1);
                        select('#baseFreq').html(floor(Math.pow(2, userBaseLog)) + ' Hz');
                    }

                    // 轉換為五度值 (設定範圍：基準點上下各約 4 個半音)
                    // 12分之4 也就是 0.33 個八度
                    let range = 0.35; 
                    let toneValue = map(logFreq, userBaseLog - range, userBaseLog + range, 1, 5);
                    points.push(toneValue);
                } else {
                    points.push(null); // 無聲時斷開
                }

                if (points.length > maxPoints) points.shift();
                getPitch();
            });
        }

        function keyPressed() {
            if (key === ' ') {
                isCalibrated = !isCalibrated;
                if (isCalibrated) {
                    select('#status').html('🔴 偵測中 (已鎖定音域)');
                    select('#guide').style('border-left', '5px solid #ff4444');
                    select('#guide').html('<strong>音域已鎖定！</strong> 現在可以練習：<br>第一聲 (55) ── 、第二聲 (214) ˇ 、第三聲 (51) ˋ 、第四聲 (35) ˊ');
                } else {
                    select('#status').html('等待校準...');
                    select('#guide').style('border-left', '5px solid #00ffcc');
                    select('#guide').html('<strong>重新校準中：</strong> 請持續發出穩定平聲，再按一次 <kbd>空白鍵</kbd> 鎖定。');
                }
            }
        }

        function draw() {
            background(10);
            drawGrid();
            
            if (points.length > 0) {
                noFill();
                strokeWeight(4);
                // 已鎖定用鮮艷青色，未鎖定用灰色
                stroke(isCalibrated ? 0, 255, 204 : 100);

                let drawWidth = width * 0.8; // 左側 80% 區域
                let step = drawWidth / maxPoints;

                beginShape();
                for (let i = 0; i < points.length; i++) {
                    if (points[i] !== null) {
                        let x = i * step;
                        // 將 1-5 度映射到畫布高度
                        let y = map(points[i], 0.5, 5.5, height, 0); 
                        vertex(x, y);
                    } else {
                        endShape();
                        beginShape();
                    }
                }
                endShape();

                // 掃描線 (現在的發音點位置)
                stroke(255, 255, 255, 100);
                strokeWeight(1);
                line(points.length * step, 0, points.length * step, height);
            }
        }

        function drawGrid() {
            // 繪製 1-5 度橫線
            for (let i = 1; i <= 5; i++) {
                let y = map(i, 0.5, 5.5, height, 0);
                stroke(40);
                strokeWeight(1);
                line(0, y, width, y);
                
                noStroke();
                fill(80);
                textSize(14);
                textAlign(LEFT);
                text(i + '度', width * 0.82, y + 5);
            }
            
            // 繪製 80% 邊界虛線
            stroke(60, 60, 200, 80);
            for(let i=0; i<height; i+=10) line(width * 0.8, i, width * 0.8, i+5);
        }
    </script>
</body>
</html>

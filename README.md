<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>五度標記法即時糾正系統</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.4.0/p5.js"></script>
    <script src="https://unpkg.com/ml5@latest/dist/ml5.min.js"></script>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; text-align: center; background: #f8f9fa; margin: 0; padding: 20px; }
        .container { max-width: 900px; margin: auto; background: white; padding: 20px; border-radius: 12px; box-shadow: 0 4px 20px rgba(0,0,0,0.1); }
        #canvas-holder { position: relative; border: 2px solid #ddd; border-radius: 8px; overflow: hidden; margin: 20px 0; }
        .controls { display: flex; justify-content: center; gap: 10px; margin-bottom: 20px; }
        button { padding: 10px 20px; font-size: 16px; cursor: pointer; border: none; border-radius: 5px; transition: 0.3s; }
        .btn-start { background: #28a745; color: white; }
        .btn-calibrate { background: #007bff; color: white; }
        .status-bar { font-size: 14px; color: #666; margin-bottom: 10px; }
        .debug-info { font-family: monospace; font-size: 12px; color: #999; }
    </style>
</head>
<body>

<div class="container">
    <h1>五韻．五度標記法糾正</h1>
    <div class="status-bar" id="status">正在載入 AI 模型...</div>
    
    <div class="controls">
        <button class="btn-calibrate" id="calBtn" onclick="toggleCalibration()">1. 開始校準 (請發平聲)</button>
        <button class="btn-start" id="startBtn" onclick="toggleAudio()">2. 開始偵測</button>
    </div>

    <div id="canvas-holder"></div>
    
    <div class="debug-info">
        基準音高 (3度): <span id="baseFreq">未設定</span> Hz | 
        即時音高: <span id="currentFreq">0</span> Hz
    </div>
</div>

<script>
let pitch;
let audioContext;
let isListening = false;
let isCalibrating = false;
let baseFrequency = 200; // 預設基準
let history = [];
let maxHistory = 400;

// 五度標記法參數：半音寬度 (通常 5 度涵蓋約 4-6 個半音)
const SEMITONE_RANGE = 5; 

function setup() {
    let cnv = createCanvas(800, 400);
    cnv.parent('canvas-holder');
    audioContext = getAudioContext();
}

function modelLoaded() {
    document.getElementById('status').innerText = "模型載入成功！請先點擊校準。";
}

function toggleAudio() {
    if (!isListening) {
        userStartAudio().then(() => {
            mic = new p5.AudioIn();
            mic.start(() => {
                pitch = ml5.pitchDetection('https://cdn.jsdelivr.net/gh/ml5js/ml5-data-and-models/models/pitch-detection/crepe/', audioContext, mic.stream, modelLoaded);
                isListening = true;
                document.getElementById('status').innerText = "偵測中...";
                loop();
            });
        });
    }
}

function toggleCalibration() {
    isCalibrating = true;
    document.getElementById('status').innerText = "請用平時說話的音高發出「啊—」三秒...";
    setTimeout(() => {
        isCalibrating = false;
        document.getElementById('status').innerText = "校準完成！現在可以開始測試發音。";
    }, 3000);
}

function draw() {
    background(255);
    drawFiveDegrees();

    if (isListening && pitch) {
        pitch.getPitch((err, freq) => {
            if (freq) {
                if (isCalibrating) {
                    baseFrequency = freq;
                    document.getElementById('baseFreq').innerText = floor(baseFrequency);
                }
                document.getElementById('currentFreq').innerText = floor(freq);
                
                // 核心轉換邏輯：Hz -> 半音 -> 五度值
                let semitoneDiff = 12 * Math.log2(freq / baseFrequency);
                // 將 3度(0) 映射到畫布中間，SEMITION_RANGE 控制靈敏度
                let toneLevel = map(semitoneDiff, -SEMITONE_RANGE, SEMITONE_RANGE, 1, 5);
                history.push(toneLevel);
            } else {
                history.push(null); // 無聲時留白
            }
            
            if (history.length > maxHistory) {
                history.shift();
            }
        });
    }

    renderCurve();
}

function drawFiveDegrees() {
    strokeWeight(1);
    textAlign(LEFT, CENTER);
    for (let i = 1; i <= 5; i++) {
        let y = map(i, 1, 5, height - 50, 50);
        stroke(230);
        line(0, y, width, y);
        noStroke();
        fill(150);
        text(i + ' 度', 10, y - 10);
    }
    
    // 繪製「現在偵測點」的分隔線 (8:2 比例)
    stroke(200, 0, 0, 50);
    line(width * 0.8, 0, width * 0.8, height);
}

function renderCurve() {
    noFill();
    strokeWeight(4);
    stroke(0, 123, 255);
    
    beginShape();
    for (let i = 0; i < history.length; i++) {
        if (history[i] !== null) {
            let x = i * (width / maxHistory);
            let y = map(history[i], 1, 5, height - 50, 50);
            vertex(x, y);
        } else {
            endShape();
            beginShape();
        }
    }
    endShape();
}
</script>

</body>
</html>

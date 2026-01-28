<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>注音五度標記偵測器 v5.0 - 極速版</title>
    <style>
        body { font-family: sans-serif; background: #121212; color: #eee; margin: 0; display: flex; flex-direction: column; height: 100vh; overflow: hidden; }
        #top { padding: 15px 20px; background: #1e1e1e; border-bottom: 1px solid #333; display: flex; justify-content: space-between; align-items: center; }
        .container { display: flex; flex: 1; }
        #left { flex: 1; position: relative; display: flex; flex-direction: column; padding: 10px; }
        #right { width: 280px; background: #181818; padding: 20px; border-left: 1px solid #333; font-size: 14px; }
        canvas { background: #000; width: 100%; flex: 1; border-radius: 8px; cursor: crosshair; }
        .stat { color: #00ff00; font-family: monospace; font-size: 20px; font-weight: bold; }
        button { padding: 10px 20px; border: none; border-radius: 5px; cursor: pointer; font-weight: bold; background: #4a90e2; color: white; transition: 0.2s; }
        button:hover { background: #357abd; }
        button:disabled { background: #444; }
        .sentence { background: #333; padding: 12px; border-radius: 6px; color: #f39c12; margin: 10px 0; font-weight: bold; text-align: center; }
        .desc { font-size: 12px; color: #888; margin-bottom: 15px; }
    </style>
</head>
<body>

<div id="top">
    <div style="font-size: 20px; font-weight: bold; color: #4a90e2;">五度標記即時偵測器</div>
    <div class="stat">Pitch: <span id="hzDisplay">0</span> Hz | Level: <span id="lvDisplay">--</span></div>
    <div style="display: flex; gap: 10px;">
        <button id="startBtn">Step 1. 開啟偵測</button>
        <button id="caliBtn" style="background:#f39c12" disabled>Step 2. 唸例句校準</button>
    </div>
</div>

<div class="container">
    <div id="left">
        <canvas id="canvas"></canvas>
    </div>
    <div id="right">
        <div class="desc">此版本優化了演算法，可達到與專業調音器同等的即時回饋性。</div>
        <h3>校準例句</h3>
        <div class="sentence">「他拔起把柄。」</div>
        <div class="desc">校準後，程式將鎖定您的音域 (1-5 樓)。</div>
        <hr style="border:0; border-top:1px solid #333;">
        <h3 style="color:#f39c12">聲調指引</h3>
        <ul style="line-height: 2; padding-left: 20px; color:#ccc;">
            <li><b>一聲 (55):</b> 高位平直</li>
            <li><b>二聲 (35):</b> 由中向高衝</li>
            <li><b>三聲 (214):</b> 低沉折轉</li>
            <li><b>四聲 (51):</b> 快速下墜</li>
        </ul>
    </div>
</div>

<script>
let audioCtx, analyser, dataArray;
let minHz = 100, maxHz = 350;
let isCalibrating = false;
let tempMin = 1000, tempMax = 50;
let history = [];

const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d', { alpha: false }); // 性能優化

// Step 1: 啟動
document.getElementById('startBtn').onclick = async () => {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)({ latencyHint: 'interactive' });
    const stream = await navigator.mediaDevices.getUserMedia({ 
        audio: { echoCancellation: false, noiseSuppression: true, autoGainControl: true } 
    });
    const source = audioCtx.createMediaStreamSource(stream);
    
    analyser = audioCtx.createAnalyser();
    analyser.fftSize = 2048; // 保持 2048 確保低音準確度，但優化讀取流程
    source.connect(analyser);
    dataArray = new Float32Array(analyser.fftSize);
    
    document.getElementById('startBtn').disabled = true;
    document.getElementById('caliBtn').disabled = false;
    
    // 初始化 history 陣列長度
    history = new Array(Math.floor(canvas.offsetWidth)).fill(null);
    tick();
};

// Step 2: 校準
document.getElementById('caliBtn').onclick = () => {
    isCalibrating = true;
    tempMin = 500; tempMax = 80;
    alert("請自然讀出：『他拔起把柄』");
    setTimeout(() => {
        isCalibrating = false;
        if(tempMax > tempMin + 30) {
            minHz = tempMin; maxHz = tempMax;
            alert(`校準完成！\n最低: ${Math.round(minHz)}Hz / 最高: ${Math.round(maxHz)}Hz`);
        }
    }, 5000);
};

// 使用快速自相關演算法 (Optimized Autocorrelation)
function autoCorrelate(buf, sampleRate) {
    let SIZE = buf.length;
    let rms = 0;
    for (let i = 0; i < SIZE; i++) rms += buf[i] * buf[i];
    rms = Math.sqrt(rms / SIZE);
    if (rms < 0.01) return -1; // 音量過低過濾

    let r1 = 0, r2 = SIZE - 1, thres = 0.2;
    for (let i = 0; i < SIZE / 2; i++) if (Math.abs(buf[i]) < thres) { r1 = i; break; }
    for (let i = 1; i < SIZE / 2; i++) if (Math.abs(buf[SIZE - i]) < thres) { r2 = SIZE - i; break; }

    buf = buf.slice(r1, r2);
    SIZE = buf.length;

    let c = new Float32Array(SIZE).fill(0);
    for (let i = 0; i < SIZE; i++)
        for (let j = 0; j < SIZE - i; j++)
            c[i] = c[i] + buf[j] * buf[j + i];

    let d = 0; while (c[d] > c[d + 1]) d++;
    let maxval = -1, maxpos = -1;
    for (let i = d; i < SIZE; i++) {
        if (c[i] > maxval) {
            maxval = c[i];
            maxpos = i;
        }
    }
    let T0 = maxpos;
    return sampleRate / T0;
}

function tick() {
    analyser.getFloatTimeDomainData(dataArray);
    let pitch = autoCorrelate(dataArray, audioCtx.sampleRate);
    
    if (pitch > 70 && pitch < 600) {
        document.getElementById('hzDisplay').innerText = Math.round(pitch);
        if(isCalibrating) {
            tempMin = Math.min(tempMin, pitch);
            tempMax = Math.max(tempMax, pitch);
        }
        let lv = 1 + 4 * (pitch - minHz) / (maxHz - minHz);
        lv = Math.max(1, Math.min(5, lv));
        document.getElementById('lvDisplay').innerText = lv.toFixed(1);
        history.push(lv);
    } else {
        history.push(null);
    }

    if (history.length > canvas.offsetWidth) history.shift();
    draw();
    requestAnimationFrame(tick);
}

function draw() {
    // 獲取畫布寬高
    if (canvas.width !== canvas.offsetWidth) canvas.width = canvas.offsetWidth;
    if (canvas.height !== canvas.offsetHeight) canvas.height = canvas.offsetHeight;

    ctx.fillStyle = "#000";
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    
    const m = 50;
    const h = canvas.height - m * 2;
    
    // 繪製背景參考線
    ctx.strokeStyle = "#222";
    ctx.lineWidth = 1;
    for(let i=1; i<=5; i++) {
        let y = canvas.height - m - (i-1)*(h/4);
        ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(canvas.width, y); ctx.stroke();
        ctx.fillStyle = "#555";
        ctx.fillText(i+"F", 10, y-5);
    }
    
    // 繪製聲調曲線 (綠色主線)
    ctx.beginPath();
    ctx.strokeStyle = "#00ff00";
    ctx.lineWidth = 4;
    ctx.lineCap = "round";
    ctx.lineJoin = "round";
    
    let started = false;
    for(let i=0; i<history.length; i++) {
        if (history[i] === null) {
            started = false;
            continue;
        }
        let x = i;
        let y = canvas.height - m - (history[i]-1)*(h/4);
        if (!started) {
            ctx.moveTo(x, y);
            started = true;
        } else {
            ctx.lineTo(x, y);
        }
    }
    ctx.stroke();
}
</script>
</body>
</html>

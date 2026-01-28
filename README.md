<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <title>注音五度標記正音儀 - 專業流暢版</title>
    <style>
        body { font-family: "PingFang TC", "Microsoft JhengHei", sans-serif; background: #0f0f0f; color: #eee; margin: 0; display: flex; height: 100vh; overflow: hidden; }
        #left { flex: 1; padding: 20px; display: flex; flex-direction: column; }
        #right { width: 300px; background: #1a1a1a; padding: 20px; border-left: 1px solid #333; overflow-y: auto; }
        canvas { background: #000; width: 100%; flex: 1; border-radius: 12px; border: 1px solid #444; }
        .stat-bar { background: #222; padding: 15px; border-radius: 8px; margin-bottom: 15px; display: flex; justify-content: space-around; border: 1px solid #333; }
        .val { color: #00ff00; font-family: 'Courier New', monospace; font-size: 24px; font-weight: bold; }
        .btn-group { display: flex; gap: 10px; margin-bottom: 15px; }
        button { flex: 1; padding: 12px; border: none; border-radius: 6px; cursor: pointer; font-weight: bold; transition: 0.3s; background: #4a90e2; color: white; }
        button:disabled { background: #444; cursor: not-allowed; }
        .box { background: #252525; padding: 15px; border-radius: 8px; margin-bottom: 20px; border-left: 4px solid #f39c12; }
        .sentence { font-size: 24px; color: #f39c12; text-align: center; font-weight: bold; margin: 10px 0; }
        .zhuyin { font-size: 14px; color: #888; text-align: center; margin-bottom: 10px; }
        h3 { color: #4a90e2; margin-top: 0; }
    </style>
</head>
<body>

<div id="left">
    <div class="stat-bar">
        <div>音高: <span id="hzDisplay" class="val">0</span> Hz</div>
        <div>樓層: <span id="lvDisplay" class="val">--</span></div>
    </div>
    <div class="btn-group">
        <button id="startBtn">1. 開始偵測 (學習雜音)</button>
        <button id="caliBtn" style="background:#f39c12" disabled>2. 唸例句校準</button>
    </div>
    <canvas id="canvas"></canvas>
</div>

<div id="right">
    <h3>校準例句</h3>
    <div class="box">
        <div class="sentence">他拔起把柄</div>
        <div class="zhuyin">ㄊㄚ ㄅㄚˊ ㄑㄧˇ ㄅㄚˇ ㄅㄧㄥˇ</div>
    </div>
    <h3>聲調指引</h3>
    <ul style="font-size: 14px; line-height: 2; padding-left: 20px;">
        <li><b>一聲:</b> 高位平直 (55)</li>
        <li><b>二聲:</b> 中向高升 (35)</li>
        <li><b>三聲:</b> 壓低勾起 (214)</li>
        <li><b>四聲:</b> 快速下墜 (51)</li>
    </ul>
    <p style="font-size: 12px; color: #666;">* 本工具採用即時 ACF 演算，反應延遲 < 50ms。</p>
</div>

<script>
let audioCtx, analyser, dataArray;
let minHz = 100, maxHz = 350;
let noiseFloor = 0.01;
let history = [];
let isCalibrating = false;
let tempMin = 1000, tempMax = 50;

const canvas = document.getElementById('canvas'), ctx = canvas.getContext('2d');

document.getElementById('startBtn').onclick = async () => {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)({ latencyHint: 'interactive' });
    const stream = await navigator.mediaDevices.getUserMedia({ 
        audio: { echoCancellation: false, noiseSuppression: true, autoGainControl: true } 
    });
    const source = audioCtx.createMediaStreamSource(stream);
    analyser = audioCtx.createAnalyser();
    analyser.fftSize = 2048; // 用於精確分析
    source.connect(analyser);
    dataArray = new Float32Array(analyser.fftSize);
    
    document.getElementById('startBtn').disabled = true;
    document.getElementById('startBtn').innerText = "系統運行中...";
    
    // 學習背景噪音
    setTimeout(() => {
        let sum = 0;
        analyser.getFloatTimeDomainData(dataArray);
        for(let i=0; i<dataArray.length; i++) sum += dataArray[i]*dataArray[i];
        noiseFloor = Math.sqrt(sum/dataArray.length) * 3.0;
        document.getElementById('caliBtn').disabled = false;
    }, 1000);

    history = new Array(Math.floor(canvas.offsetWidth)).fill(null);
    update();
};

document.getElementById('caliBtn').onclick = () => {
    isCalibrating = true;
    tempMin = 500; tempMax = 80;
    alert("請自然讀出：他拔起把柄");
    setTimeout(() => {
        isCalibrating = false;
        if(tempMax > tempMin + 20) {
            minHz = tempMin; maxHz = tempMax;
        }
    }, 5000);
};

// 專業級 Pitch Detection 演算法 (ACF)
function getPitch(buf, sampleRate) {
    let sum = 0;
    for (let i = 0; i < buf.length; i++) sum += buf[i] * buf[i];
    if (Math.sqrt(sum / buf.length) < noiseFloor) return -1;

    let c = new Float32Array(buf.length).fill(0);
    for (let i = 0; i < buf.length; i++) {
        for (let j = 0; j < buf.length - i; j++) c[i] += buf[j] * buf[j + i];
    }

    let d = 0; while (c[d] > c[d + 1]) d++;
    let maxval = -1, maxpos = -1;
    for (let i = d; i < buf.length; i++) {
        if (c[i] > maxval) { maxval = c[i]; maxpos = i; }
    }

    // 關鍵：拋物線插值優化 (Parabolic Interpolation)
    // 讓 Hz 偵測不再是跳躍的整數，而是平滑的浮點數
    let x0 = c[maxpos - 1], x1 = c[maxpos], x2 = c[maxpos + 1];
    let a = (x0 + x2 - 2 * x1) / 2;
    let b = (x2 - x0) / 2;
    let refinedPos = maxpos - b / (2 * a);

    return sampleRate / refinedPos;
}

function update() {
    analyser.getFloatTimeDomainData(dataArray);
    let pitch = getPitch(dataArray, audioCtx.sampleRate);

    if (pitch > 75 && pitch < 550) {
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
    requestAnimationFrame(update);
}

function draw() {
    if (canvas.width !== canvas.offsetWidth) canvas.width = canvas.offsetWidth;
    if (canvas.height !== canvas.offsetHeight) canvas.height = canvas.offsetHeight;
    
    ctx.fillStyle = "#000";
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    
    const m = 40, h = canvas.height - m * 2;
    ctx.strokeStyle = "#222";
    for(let i=1; i<=5; i++) {
        let y = canvas.height - m - (i-1)*(h/4);
        ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(canvas.width, y); ctx.stroke();
        ctx.fillStyle = "#555"; ctx.font = "12px sans-serif";
        ctx.fillText(i + " 樓", 10, y - 5);
    }

    ctx.beginPath();
    ctx.strokeStyle = "#00ff00";
    ctx.lineWidth = 4;
    ctx.lineCap = "round";
    ctx.lineJoin = "round";
    
    let started = false;
    for(let i = 0; i < history.length; i++) {
        if (history[i] === null) { started = false; continue; }
        let x = i;
        let y = canvas.height - m - (history[i] - 1) * (h/4);
        if (!started) { ctx.moveTo(x, y); started = true; }
        else { ctx.lineTo(x, y); }
    }
    ctx.stroke();
}
</script>
</body>
</html>

<<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>注音五度標記偵測工具 | Tone Practice</title>
    <style>
        body { font-family: "PingFang TC", "Microsoft JhengHei", sans-serif; background: #121212; color: #eee; margin: 0; display: flex; flex-direction: column; height: 100vh; }
        .container { display: flex; flex: 1; overflow: hidden; }
        #left { flex: 1; padding: 20px; display: flex; flex-direction: column; }
        #right { width: 300px; background: #1e1e1e; padding: 20px; border-left: 1px solid #333; overflow-y: auto; }
        
        h1 { color: #4a90e2; font-size: 22px; margin-top: 0; }
        .description { font-size: 14px; color: #bbb; line-height: 1.6; margin-bottom: 20px; }
        
        .status-box { background: #2a2a2a; padding: 15px; border-radius: 8px; margin-bottom: 15px; display: flex; gap: 20px; }
        .val { color: #00ff00; font-family: monospace; font-size: 20px; font-weight: bold; }
        
        canvas { background: #000; width: 100%; height: 350px; border-radius: 10px; border: 1px solid #444; }
        
        .btns { margin-bottom: 20px; display: flex; gap: 10px; }
        button { flex: 1; padding: 12px; border: none; border-radius: 5px; cursor: pointer; font-weight: bold; color: white; transition: 0.2s; }
        #startBtn { background: #4a90e2; }
        #caliBtn { background: #f39c12; }
        button:disabled { background: #555; opacity: 0.6; }
        
        h3 { color: #f39c12; font-size: 18px; border-bottom: 1px solid #444; padding-bottom: 8px; }
        .tips { font-size: 14px; line-height: 1.8; color: #ccc; }
        .lang-en { color: #888; font-size: 12px; display: block; margin-top: 2px; }
    </style>
</head>
<body>

<div class="container">
    <div id="left">
        <h1>五度標記法即時正音工具 <span style="font-size:14px; color:#666;">v3.0</span></h1>
        <div class="description">
            此工具能即時顯示您的發音高度，協助您掌握注音聲調的精準度。<br>
            <span class="lang-en">This tool visualizes your pitch in real-time to help master Chinese tones.</span>
        </div>

        <div class="btns">
            <button id="startBtn">Step 1: 開啟麥克風 & 學習雜音</button>
            <button id="caliBtn" disabled>Step 2: 校準個人音域</button>
        </div>

        <div class="status-box">
            <div>音高 Pitch: <span id="hzDisplay" class="val">0</span> Hz</div>
            <div>樓層 Level: <span id="lvDisplay" class="val">--</span></div>
        </div>

        <canvas id="canvas"></canvas>
    </div>

    <div id="right">
        <h3>練習建議 Tips</h3>
        <div class="tips">
            <b>● 一聲 (55) First Tone</b><br>
            維持在 5 樓的高平線。<br>
            <span class="lang-en">Steady high line on level 5.</span>
            <br>
            <b>● 二聲 (35) Second Tone</b><br>
            從 3 樓向上衝到 5 樓。<br>
            <span class="lang-en">Rise from level 3 to 5.</span>
            <br>
            <b>● 三聲 (214) Third Tone</b><br>
            必須先深蹲到 1 樓再勾起。<br>
            <span class="lang-en">Dip to level 1, then slightly rise.</span>
            <br>
            <b>● 四聲 (51) Fourth Tone</b><br>
            從 5 樓快速墜落到 1 樓。<br>
            <span class="lang-en">Quick drop from level 5 to 1.</span>
        </div>
    </div>
</div>

<script>
let audioCtx, analyser, dataArray;
let minHz = 100, maxHz = 350;
let noiseFloor = 0.015;
let isLearningNoise = false;
let isCalibrating = false;
let tempMin = 1000, tempMax = 50;

const canvas = document.getElementById('canvas'), ctx = canvas.getContext('2d');
let history = new Array(800).fill(null);

// 核心偵測按鈕
document.getElementById('startBtn').onclick = async () => {
    try {
        audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        const stream = await navigator.mediaDevices.getUserMedia({ 
            audio: { echoCancellation: true, noiseSuppression: true } 
        });
        const source = audioCtx.createMediaStreamSource(stream);
        analyser = audioCtx.createAnalyser();
        analyser.fftSize = 2048;
        source.connect(analyser);
        dataArray = new Float32Array(analyser.fftSize);
        
        document.getElementById('startBtn').disabled = true;
        document.getElementById('startBtn').innerText = "偵測中 Testing...";
        
        isLearningNoise = true;
        let samples = [];
        alert("【步驟 1】請保持安靜 2 秒，系統正在學習您的環境雜音...\n(Please stay quiet for 2s)");
        
        let checkNoise = setInterval(() => {
            analyser.getFloatTimeDomainData(dataArray);
            let rms = 0;
            for (let i = 0; i < dataArray.length; i++) rms += dataArray[i] * dataArray[i];
            samples.push(Math.sqrt(rms / dataArray.length));
        }, 100);

        setTimeout(() => {
            clearInterval(checkNoise);
            let avg = samples.reduce((a, b) => a + b) / samples.length;
            noiseFloor = Math.max(avg * 2.0, 0.01); 
            isLearningNoise = false;
            document.getElementById('caliBtn').disabled = false;
            alert("環境音學習完成！您可以開始說話，或點擊 Step 2 校準音域。");
        }, 2000);

        loop();
    } catch (e) {
        alert("無法讀取麥克風，請檢查瀏覽器設定！");
    }
};

// 校準按鈕
document.getElementById('caliBtn').onclick = () => {
    isCalibrating = true;
    tempMin = 1000; tempMax = 50;
    alert("【步驟 2】請在 5 秒內，發出「啊——」並從最低音滑到最高音。");
    setTimeout(() => {
        isCalibrating = false;
        if (tempMax > tempMin + 20) {
            minHz = tempMin; maxHz = tempMax;
            alert(`校準成功！您的音域：${Math.round(minHz)} - ${Math.round(maxHz)} Hz`);
        } else {
            alert("校準失敗，請再試一次，聲音起伏要大一點喔！");
        }
    }, 5000);
};

// 偵測算法
function findPitch(data, sampleRate) {
    let n = data.length;
    let rms = 0;
    for (let i = 0; i < n; i++) rms += data[i] * data[i];
    rms = Math.sqrt(rms / n);
    if (rms < noiseFloor) return null;

    let r = new Float32Array(n);
    for (let i = 0; i < n; i++) {
        for (let j = 0; j < n - i; j++) r[i] += data[j] * data[j + i];
    }
    let d = 0; while (r[d] > r[d+1]) d++;
    let maxV = -1, maxP = -1;
    for (let i = d; i < n; i++) {
        if (r[i] > maxV) { maxV = r[i]; maxP = i; }
    }
    let pitch = sampleRate / maxP;
    return (pitch > 70 && pitch < 500) ? pitch : null;
}

function loop() {
    if(!isLearningNoise && analyser) {
        analyser.getFloatTimeDomainData(dataArray);
        let pitch = findPitch(dataArray, audioCtx.sampleRate);
        if (pitch) {
            document.getElementById('hzDisplay').innerText = Math.round(pitch);
            if(isCalibrating) {
                tempMin = Math.min(tempMin, pitch);
                tempMax = Math.max(tempMax, pitch);
            }
            let lv = 1 + 4 * ((pitch - minHz) / (maxHz - minHz));
            lv = Math.max(1, Math.min(5, lv));
            document.getElementById('lvDisplay').innerText = lv.toFixed(1);
            history.push(lv);
        } else {
            history.push(null);
        }
    }
    if (history.length > canvas.width) history.shift();
    draw();
    requestAnimationFrame(loop);
}

function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.strokeStyle = "#333";
    ctx.lineWidth = 1;
    for(let i=1; i<=5; i++) {
        let y = canvas.height - (i-1) * (canvas.height/4) - 20;
        ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(canvas.width, y); ctx.stroke();
        ctx.fillStyle = "#666"; ctx.font = "12px Arial";
        ctx.fillText(i + " 樓", 10, y - 5);
    }
    ctx.beginPath(); ctx.strokeStyle = "#00ff00"; ctx.lineWidth = 4; ctx.lineCap = "round";
    let first = true;
    for(let i=0; i<history.length; i++) {
        if (history[i] === null) { first = true; continue; }
        let y = canvas.height - (history[i]-1) * (canvas.height/4) - 20;
        if (first) { ctx.moveTo(i, y); first = false; }
        else { ctx.lineTo(i, y); }
    }
    ctx.stroke();
}
</script>
</body>
</html>

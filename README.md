# tone-practice-traditional-chinese
There are five different tones in Chinese, and this is the tool to make sure that your tone is right or not.
這個工具可以幫助你知曉你在發音5個聲調(輕聲、一聲、二聲、三聲、四聲)時是否使用正確音調。

<script>
let audioCtx, analyser, dataArray;
let minHz = 100, maxHz = 350, isCalib = false;
const canvas = document.getElementById('canvas'), ctx = canvas.getContext('2d');
let history = new Array(800).fill(null);

// 初始化單詞表
const words = ["ㄅ：八方(55) - 穩住5樓","ㄉ：單獨(55-35) - 5到3樓","三聲：把(214) - 1樓下潛再勾起","四聲：爸爸(51) - 5樓摔到1樓"];
words.forEach(w => { let li = document.createElement('li'); li.innerText = w; document.getElementById('list').appendChild(li); });

document.getElementById('startBtn').onclick = async () => {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    const source = audioCtx.createMediaStreamSource(stream);
    analyser = audioCtx.createAnalyser();
    analyser.fftSize = 2048;
    source.connect(analyser);
    dataArray = new Float32Array(analyser.fftSize);
    loop();
};

document.getElementById('caliBtn').onclick = () => {
    isCalib = true; 
    tempMin = 1000; tempMax = 50; // 重設暫存範圍
    alert("開始校準！請發出「啊——」由低到高滑音...");
    setTimeout(() => { 
        isCalib = false; 
        if(tempMin < tempMax) {
            minHz = tempMin; maxHz = tempMax;
        }
        alert(`校準完成！範圍：${Math.round(minHz)} - ${Math.round(maxHz)} Hz`); 
    }, 5000);
};

let tempMin, tempMax;

function findPitch(data, sampleRate) {
    let n = data.length, r = new Float32Array(n);
    // 使用簡單的 ACF 演算法
    for (let i = 0; i < n; i++) {
        for (let j = 0; j < n - i; j++) r[i] += data[j] * data[j + i];
    }
    let d = 0; while (r[d] > r[d+1]) d++;
    let maxV = -1, maxP = -1;
    for (let i = d; i < n; i++) { if (r[i] > maxV) { maxV = r[i]; maxP = i; } }
    return sampleRate / maxP;
}

function loop() {
    requestAnimationFrame(loop);
    if(!analyser) return;
    analyser.getFloatTimeDomainData(dataArray);
    let pitch = findPitch(dataArray, audioCtx.sampleRate);

    // 過濾合理的人聲範圍 (80Hz - 500Hz)
    if (pitch && pitch > 80 && pitch < 500) {
        document.getElementById('hzDisplay').innerText = Math.round(pitch);
        if (isCalib) {
            if(pitch < tempMin) tempMin = pitch;
            if(pitch > tempMax) tempMax = pitch;
        }
        
        // 修正後的樓層公式：(目前頻率 - 最低) / (最高 - 最低)
        // 頻率越高，百分比越大，樓層越高
        let range = maxHz - minHz;
        let lv = 1 + 4 * ( (pitch - minHz) / range );
        
        lv = Math.max(1, Math.min(5, lv)); // 強制限制在 1-5 樓
        document.getElementById('lvDisplay').innerText = lv.toFixed(1);
        history.push(lv);
    } else {
        history.push(null);
    }
    if (history.length > canvas.width) history.shift();
    draw();
}

function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    // 繪製背景參考線
    ctx.strokeStyle = "#444";
    ctx.setLineDash([5, 5]);
    for(let i=1; i<=5; i++) {
        let y = canvas.height - (i-1) * (canvas.height/4) - 5;
        ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(canvas.width, y); ctx.stroke();
        ctx.fillStyle = "#888"; ctx.fillText(i+"樓", 5, y-5);
    }
    ctx.setLineDash([]);
    
    // 繪製音高曲線
    ctx.beginPath(); ctx.strokeStyle = "#00ff00"; ctx.lineWidth = 4; ctx.lineJoin = "round";
    let first = true;
    for(let i=0; i<history.length; i++) {
        if (history[i] === null) { first = true; continue; }
        let y = canvas.height - (history[i]-1) * (canvas.height/4) - 5;
        if (first) { ctx.moveTo(i, y); first = false; } 
        else { ctx.lineTo(i, y); }
    }
    ctx.stroke();
}
</script>

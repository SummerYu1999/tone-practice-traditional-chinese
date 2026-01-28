# tone-practice-traditional-chinese
There are five different tones in Chinese, and this is the tool to make sure if your tone right or not.
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <title>注音五度標記偵測器</title>
    <style>
        body { font-family: sans-serif; display: flex; height: 100vh; margin: 0; background: #1a1a1a; color: white; }
        #left { flex: 1; padding: 20px; border-right: 1px solid #444; display: flex; flex-direction: column; }
        #right { width: 350px; padding: 20px; overflow-y: auto; background: #222; }
        canvas { background: #000; width: 100%; height: 350px; border-radius: 8px; }
        .controls { margin-bottom: 20px; }
        button { padding: 12px; margin: 5px; cursor: pointer; border-radius: 5px; border: none; background: #4a90e2; color: white; font-weight: bold; }
        button:disabled { background: #555; }
        .level-info { font-size: 24px; color: #00ff00; margin: 10px 0; }
        h3 { color: #4a90e2; border-bottom: 1px solid #444; padding-bottom: 5px; }
        li { margin: 10px 0; font-size: 16px; border-bottom: 1px solid #333; padding-bottom: 5px; list-style: none; }
    </style>
</head>
<body>
    <div id="left">
        <h2>五度標記即時偵測</h2>
        <div class="controls">
            <button id="startBtn">1. 開啟麥克風</button>
            <button id="caliBtn" disabled>2. 校準音域 (5秒高低音)</button>
        </div>
        <div class="level-info">樓層：<span id="levelDisplay">--</span> | <span id="hzDisplay">--</span> Hz</div>
        <canvas id="canvas"></canvas>
    </div>
    <div id="right">
        <h3>日常詞組練習表</h3>
        <ul>
            <li><b>ㄅ：</b>八方、拔起、把持、霸道、爸爸、好吧</li>
            <li><b>ㄉ：</b>單獨、跌倒、低調、地道、弟弟、等等</li>
            <li><b>ㄍ：</b>廣告、改革、鞏固、尷尬、哥哥、姑姑</li>
            <li><b>ㄓ：</b>真正、支持、住宅、執著、知道、指教</li>
            <li><b>ㄧ：</b>意義、記憶、稀奇、阿姨、低頭、一直</li>
            <li><b>ㄦ：</b>女兒、兒歌、而且、耳環、二胡、兒童</li>
        </ul>
    </div>
    <script>
        let audioCtx, analyser, dataArray, source;
        let minHz = 100, maxHz = 350;
        let isCalibrating = false;
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        let history = [];

        document.getElementById('startBtn').onclick = async function() {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
            source = audioCtx.createMediaStreamSource(stream);
            analyser = audioCtx.createAnalyser();
            analyser.fftSize = 2048;
            source.connect(analyser);
            dataArray = new Float32Array(analyser.frequencyBinCount);
            this.disabled = true;
            document.getElementById('caliBtn').disabled = false;
            render();
        };

        document.getElementById('caliBtn').onclick = function() {
            isCalibrating = true;
            minHz = 1000; maxHz = 50;
            alert("請在5秒內由低到高發出「啊——」的聲音");
            setTimeout(() => { isCalibrating = false; alert("校準完成！"); }, 5000);
        };

        function render() {
            requestAnimationFrame(render);
            analyser.getFloatTimeDomainData(dataArray);
            let pitch = autoCorrelate(dataArray, audioCtx.sampleRate);
            if (pitch && pitch > 60 && pitch < 800) {
                document.getElementById('hzDisplay').innerText = Math.round(pitch);
                if (isCalibrating) {
                    if (pitch < minHz) minHz = pitch;
                    if (pitch > maxHz) maxHz = pitch;
                }
                let level = 1 + 4 * (Math.log(pitch) - Math.log(minHz)) / (Math.log(maxHz) - Math.log(minHz));
                level = Math.max(1, Math.min(5, level));
                document.getElementById('levelDisplay').innerText = level.toFixed(1);
                history.push(level);
            } else { history.push(null); }
            if (history.length > canvas.width) history.shift();
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.strokeStyle = "#444";
            for(let i=1; i<=5; i++) {
                let y = canvas.height - (i-1) * (canvas.height/4);
                ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(canvas.width, y); ctx.stroke();
                ctx.fillStyle = "#888"; ctx.fillText(i+"樓", 5, y-5);
            }
            ctx.beginPath(); ctx.strokeStyle = "#00ff00"; ctx.lineWidth = 3;
            for(let i=0; i<history.length; i++) {
                if (history[i] === null) continue;
                let y = canvas.height - (history[i]-1) * (canvas.height/4);
                if (i===0) ctx.moveTo(i, y); else ctx.lineTo(i, y);
            }
            ctx.stroke();
        }

        function autoCorrelate(buf, sampleRate) {
            let rms = 0;
            for (let i=0; i<buf.length; i++) rms += buf[i]*buf[i];
            if (Math.sqrt(rms/buf.length) < 0.015) return null;
            let r1 = 0, r2 = buf.length-1, thres = 0.2;
            for (let i=0; i<buf.length/2; i++) if (Math.abs(buf[i]) < thres) { r1 = i; break; }
            for (let i=1; i<buf.length/2; i++) if (Math.abs(buf[buf.length-i]) < thres) { r2 = buf.length-i; break; }
            buf = buf.slice(r1, r2);
            let c = new Array(buf.length).fill(0);
            for (let i=0; i<buf.length; i++) for (let j=0; j<buf.length-i; j++) c[i] = c[i] + buf[j]*buf[j+i];
            let d=0; while (c[d]>c[d+1]) d++;
            let maxval = -1, maxpos = -1;
            for (let i=d; i<buf.length; i++) if (c[i] > maxval) { maxval = c[i]; maxpos = i; }
            return sampleRate/maxpos;
        }
    </script>
</body>
</html>

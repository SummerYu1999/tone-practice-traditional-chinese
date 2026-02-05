<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <title>注音點位精確調教器 (修正版)</title>
    <style>
        body { font-family: sans-serif; background: #f4f7f6; display: flex; flex-direction: column; align-items: center; padding: 20px; }
        .main-layout { display: flex; gap: 20px; max-width: 1400px; width: 100%; justify-content: center; }
        
        .map-container { 
            position: relative; 
            background: white; 
            padding: 10px; 
            border-radius: 10px; 
            box-shadow: 0 5px 15px rgba(0,0,0,0.2);
            user-select: none;
            height: fit-content;
        }
        #target-img { max-width: 600px; height: auto; display: block; }

        .draggable-dot {
            position: absolute;
            width: 20px;
            height: 20px;
            background: #ff4757;
            border: 2px solid white;
            border-radius: 50%;
            transform: translate(-50%, -50%);
            cursor: move;
            z-index: 100;
            display: flex;
            justify-content: center;
            align-items: center;
            color: white;
            font-size: 11px;
            font-weight: bold;
            box-shadow: 0 2px 5px rgba(0,0,0,0.3);
        }
        .draggable-dot.active { background: #2ed573; scale: 1.2; box-shadow: 0 0 15px #2ed573; }

        .control-panel { width: 550px; display: flex; flex-direction: column; gap: 15px; }
        .editor-section { background: white; padding: 20px; border-radius: 10px; box-shadow: 0 5px 15px rgba(0,0,0,0.1); }
        
        .coord-list { max-height: 400px; overflow-y: auto; margin-bottom: 15px; border: 1px solid #eee; padding: 10px; border-radius: 5px; }
        .coord-item { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; padding: 5px; border-bottom: 1px solid #f9f9f9; }
        .coord-item label { width: 100px; font-weight: bold; font-size: 14px; }
        .coord-item input { width: 60px; padding: 4px; border: 1px solid #ccc; border-radius: 4px; }

        .code-output { 
            background: #2f3542; color: #ced6e0; padding: 15px; border-radius: 5px; 
            font-family: 'Courier New', monospace; font-size: 12px; height: 200px; overflow-y: auto; white-space: pre;
        }
        
        .btn-group { margin-top: 10px; display: flex; gap: 10px; }
        button { padding: 8px 15px; cursor: pointer; border-radius: 5px; border: none; background: #3498db; color: white; transition: 0.3s; }
        button:hover { background: #2980b9; }
    </style>
</head>
<body>

    <h2>🎨 視覺化拖移 + 數值微調工具 (已補上編號3)</h2>

    <div class="main-layout">
        <div class="map-container" id="map-box">
            <img id="target-img" src="https://upload.wikimedia.org/wikipedia/commons/thumb/7/75/Places_of_articulation.svg/500px-Places_of_articulation.svg.png">
            <div id="dots-layer"></div>
        </div>

        <div class="control-panel">
            <div class="editor-section">
                <div id="coord-list" class="coord-list"></div>
                <h3>📋 MasterDictionary 代碼</h3>
                <div id="code-box" class="code-output"></div>
                <div class="btn-group">
                    <button onclick="copyToClipboard()" style="background: #2ed573;">複製全部代碼</button>
                    <button onclick="window.location.reload()" style="background: #747d8c;">重置</button>
                </div>
            </div>
        </div>
    </div>

    <script>
        // 補上編號 3，並預設在齒齦附近 (x:20, y:42 左右)
        let points = [
            { char: "ㄅㄆㄇ", label: "1上", x: 9.1, y: 40.3 },
            { char: "ㄈ", label: "2上", x: 14.7, y: 44.6 },
            { char: "ㄗㄘㄙ", label: "3", x: 20.0, y: 42.0 }, // 新增齒齦點位
            { char: "ㄉㄊㄋㄌ", label: "4", x: 27.7, y: 40.6 },
            { char: "ㄓㄔㄕㄖ", label: "6", x: 38.3, y: 36.3 },
            { char: "ㄐㄑㄒ", label: "7", x: 51.7, y: 37.4 },
            { char: "ㄍㄎㄏ", label: "8", x: 64.9, y: 39.5 },
            { char: "ㄦ", label: "17", x: 12.3, y: 61.1 }
        ];

        const dotsLayer = document.getElementById('dots-layer');
        const coordList = document.getElementById('coord-list');
        const codeBox = document.getElementById('code-box');
        const img = document.getElementById('target-img');

        function init() { renderDots(); renderInputs(); updateCode(); }

        function renderDots() {
            dotsLayer.innerHTML = '';
            points.forEach((p, index) => {
                const dot = document.createElement('div');
                dot.className = 'draggable-dot';
                dot.id = `dot-${index}`;
                dot.style.left = p.x + '%';
                dot.style.top = p.y + '%';
                dot.innerText = p.label;
                dot.onmousedown = (e) => startDrag(e, index, dot);
                dotsLayer.appendChild(dot);
            });
        }

        function renderInputs() {
            coordList.innerHTML = '';
            points.forEach((p, index) => {
                const item = document.createElement('div');
                item.className = 'coord-item';
                item.innerHTML = `
                    <label>${p.char}(${p.label})</label>
                    X: <input type="number" step="0.1" value="${p.x}" oninput="syncInput(${index}, 'x', this.value)">
                    Y: <input type="number" step="0.1" value="${p.y}" oninput="syncInput(${index}, 'y', this.value)">
                `;
                coordList.appendChild(item);
            });
        }

        function startDrag(e, index, dotElement) {
            dotElement.classList.add('active');
            const rect = img.getBoundingClientRect();
            function move(e) {
                let x = Math.max(0, Math.min(100, ((e.clientX - rect.left) / rect.width * 100)));
                let y = Math.max(0, Math.min(100, ((e.clientY - rect.top) / rect.height * 100)));
                points[index].x = parseFloat(x.toFixed(1));
                points[index].y = parseFloat(y.toFixed(1));
                dotElement.style.left = points[index].x + '%';
                dotElement.style.top = points[index].y + '%';
                const inputs = coordList.querySelectorAll('.coord-item')[index].querySelectorAll('input');
                inputs[0].value = points[index].x;
                inputs[1].value = points[index].y;
                updateCode();
            }
            function stop() {
                document.removeEventListener('mousemove', move);
                document.removeEventListener('mouseup', stop);
                dotElement.classList.remove('active');
            }
            document.addEventListener('mousemove', move);
            document.addEventListener('mouseup', stop);
        }

        window.syncInput = function(index, axis, value) {
            const val = parseFloat(value) || 0;
            points[index][axis] = val;
            const dot = document.getElementById(`dot-${index}`);
            if (axis === 'x') dot.style.left = val + '%';
            else dot.style.top = val + '%';
            updateCode();
        };

        function updateCode() {
            let code = "const MasterDictionary = {\n";
            points.forEach(p => {
                code += `    "${p.char}": { pos: {x: ${p.x}, y: ${p.y}}, loc: "部位${p.label}" },\n`;
            });
            code += "};";
            codeBox.innerText = code;
        }

        window.copyToClipboard = function() {
            navigator.clipboard.writeText(codeBox.innerText);
            alert("代碼已複製！");
        }

        img.onload = init;
        if(img.complete) init();
    </script>
</body>
</html>

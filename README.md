<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <title>注音點位精確調教器 (含微調功能)</title>
    <style>
        body { font-family: sans-serif; background: #f4f7f6; display: flex; flex-direction: column; align-items: center; padding: 20px; }
        .main-layout { display: flex; gap: 20px; max-width: 1400px; width: 100%; justify-content: center; }
        
        /* 地圖區域 */
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

        /* 可拖移點 */
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

        /* 右側控制面板 */
        .control-panel { width: 550px; display: flex; flex-direction: column; gap: 15px; }
        .editor-section { background: white; padding: 20px; border-radius: 10px; box-shadow: 0 5px 15px rgba(0,0,0,0.1); }
        
        /* 數值微調列表 */
        .coord-list { max-height: 300px; overflow-y: auto; margin-bottom: 15px; border: 1px solid #eee; padding: 10px; border-radius: 5px; }
        .coord-item { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; padding: 5px; border-bottom: 1px solid #f9f9f9; }
        .coord-item label { width: 80px; font-weight: bold; font-size: 14px; }
        .coord-item input { width: 60px; padding: 4px; border: 1px solid #ccc; border-radius: 4px; }

        /* 代碼輸出 */
        .code-output { 
            background: #2f3542; color: #ced6e0; padding: 15px; border-radius: 5px; 
            font-family: 'Courier New', monospace; font-size: 12px; height: 250px; overflow-y: auto; white-space: pre;
        }
        
        .btn-group { margin-top: 10px; display: flex; gap: 10px; }
        button { padding: 8px 15px; cursor: pointer; border-radius: 5px; border: none; background: #3498db; color: white; transition: 0.3s; }
        button:hover { background: #2980b9; }
    </style>
</head>
<body>

    <h2>🎨 視覺化拖移 + 數值微調工具</h2>
    <p style="color: #666;">滑鼠拖移紅點，或在右側輸入框微調 0-100 的數值。</p>

    <div class="main-layout">
        <div class="map-container" id="map-box">
            <img id="target-img" src="https://upload.wikimedia.org/wikipedia/commons/thumb/7/75/Places_of_articulation.svg/500px-Places_of_articulation.svg.png">
            <div id="dots-layer"></div>
        </div>

        <div class="control-panel">
            <div class="editor-section">
                <h3>🔢 數值微調 (百分比 %)</h3>
                <div id="coord-list" class="coord-list">
                    </div>

                <h3>📋 MasterDictionary 代碼</h3>
                <div id="code-box" class="code-output"></div>
                
                <div class="btn-group">
                    <button onclick="copyToClipboard()" style="background: #2ed573;">複製全部代碼</button>
                    <button onclick="window.location.reload()" style="background: #747d8c;">重置位置</button>
                </div>
            </div>
        </div>
    </div>

    <script>
        // 初始化點位數據
        let points = [
            { char: "ㄅㄆㄇ", label: "1上", x: 9.1, y: 40.3 },
            { char: "ㄈ", label: "2上", x: 14.7, y: 44.6 },
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

        // 初始化渲染
        function init() {
            renderDots();
            renderInputs();
            updateCode();
        }

        // 渲染地圖上的點
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

        // 渲染右側輸入框
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

        // 拖動邏輯
        function startDrag(e, index, dotElement) {
            dotElement.classList.add('active');
            const rect = img.getBoundingClientRect();
            
            function move(e) {
                let x = ((e.clientX - rect.left) / rect.width * 100);
                let y = ((e.clientY - rect.top) / rect.height * 100);
                
                x = Math.max(0, Math.min(100, x.toFixed(1)));
                y = Math.max(0, Math.min(100, y.toFixed(1)));

                points[index].x = parseFloat(x);
                points[index].y = parseFloat(y);
                
                dotElement.style.left = x + '%';
                dotElement.style.top = y + '%';
                
                // 同步更新輸入框
                const inputs = coordList.querySelectorAll('.coord-item')[index].querySelectorAll('input');
                inputs[0].value = x;
                inputs[1].value = y;
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

        // 輸入框同步邏輯
        window.syncInput = function(index, axis, value) {
            const val = parseFloat(value) || 0;
            points[index][axis] = val;
            
            // 更新地圖點位置
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
            alert("MasterDictionary 已複製！可以直接貼上到專案中。");
        }

        img.onload = init;
        if(img.complete) init();
    </script>
</body>
</html>

<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <title>發音部位座標校準器</title>
    <style>
        body { font-family: sans-serif; display: flex; flex-direction: column; align-items: center; padding: 20px; background: #f0f0f0; }
        .container { position: relative; display: inline-block; background: white; padding: 10px; border-radius: 8px; box-shadow: 0 4px 10px rgba(0,0,0,0.2); }
        
        /* 這是你要測試的地圖圖片 */
        #test-map { display: block; max-width: 600px; height: auto; cursor: crosshair; }

        /* 點擊後產生的測試點 */
        .test-dot {
            position: absolute;
            width: 12px;
            height: 12px;
            background: red;
            border: 2px solid white;
            border-radius: 50%;
            transform: translate(-50%, -50%);
            pointer-events: none;
        }

        .info-panel {
            margin-top: 20px;
            padding: 15px;
            background: white;
            border-radius: 8px;
            border-left: 5px solid #e74c3c;
            width: 400px;
        }
        code { background: #eee; padding: 2px 5px; border-radius: 3px; color: #c0392b; }
    </style>
</head>
<body>

    <h2>發音部位座標校準器</h2>
    <p>請點擊圖片中的編號 (1~18)，下方會產生對應的 <code>pos</code> 程式碼</p>

    <div class="container" id="map-container">
        <img id="test-map" src="https://i.imgur.com/vH9XJ9W.png" alt="發音部位圖">
        <div id="dot-preview"></div>
    </div>

    <div class="info-panel">
        <strong>目前點擊位置：</strong>
        <div id="result">尚未點擊</div>
        <hr>
        <p><small>點擊後，請直接複製 <code>pos: {x: ..., y: ...}</code> 到你的字典中。</small></p>
    </div>

    <script>
        const map = document.getElementById('test-map');
        const container = document.getElementById('map-container');
        const result = document.getElementById('result');
        const dotPreview = document.getElementById('dot-preview');

        map.addEventListener('click', function(e) {
            // 計算點擊位置相對於圖片的百分比
            const rect = map.getBoundingClientRect();
            const x = ((e.clientX - rect.left) / rect.width * 100).toFixed(1);
            const y = ((e.clientY - rect.top) / rect.height * 100).toFixed(1);

            // 更新文字顯示
            result.innerHTML = `程式碼：<code>pos: {x: ${x}, y: ${y}}</code>`;

            // 在點擊處產生一個紅點預覽
            dotPreview.innerHTML = `<div class="test-dot" style="left: ${x}%; top: ${y}%;"></div>`;
            
            console.log(`點擊位置 - X: ${x}%, Y: ${y}%`);
        });
    </script>

</body>
</html>

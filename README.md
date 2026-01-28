<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>注音五度標記練習 - 專業引擎版</title>
    <style>
        body { font-family: "PingFang TC", sans-serif; background: #121212; color: #eee; margin: 0; display: flex; flex-direction: column; height: 100vh; }
        
        /* 上方標題區 */
        header { background: #1e1e1e; padding: 15px 25px; border-bottom: 2px solid #4a90e2; }
        h1 { margin: 0; font-size: 20px; color: #4a90e2; }
        .subtitle { font-size: 13px; color: #888; margin-top: 5px; }

        /* 中間主視窗區 */
        .main-content { display: flex; flex: 1; overflow: hidden; }

        /* 左側：鑲嵌的專業偵測器 */
        #detector-container { flex: 1; background: #000; position: relative; }
        iframe { width: 100%; height: 100%; border: none; }

        /* 右側：你的教學說明 */
        #sidebar { width: 320px; background: #181818; padding: 25px; border-left: 1px solid #333; overflow-y: auto; }
        .sentence-box { background: #333; border: 1px dashed #f39c12; padding: 15px; border-radius: 8px; color: #f39c12; text-align: center; font-weight: bold; margin: 15px 0; }
        
        h3 { border-bottom: 1px solid #444; padding-bottom: 8px; color: #f39c12; }
        .guide-list { line-height: 1.8; font-size: 14px; padding-left: 20px; }
        .guide-list b { color: #4a90e2; }

        /* 覆蓋層提示 */
        .overlay-tip { position: absolute; top: 10px; left: 10px; background: rgba(74, 144, 226, 0.8); color: white; padding: 5px 12px; border-radius: 4px; font-size: 12px; pointer-events: none; }
    </style>
</head>
<body>

<header>
    <h1>注音五度標記練習工具 <span style="font-weight:normal; font-size:12px; color:#666;">Powered by Bideyuanli Engine</span></h1>
    <div class="subtitle">結合專業頻率偵測與語音學五度座標制。</div>
</header>

<div class="main-content">
    <div id="detector-container">
        <div class="overlay-tip">請點擊下方畫面的「開始」按鈕啟動偵測</div>
        <iframe src="https://bideyuanli.com/pp" allow="microphone"></iframe>
    </div>

    <aside id="sidebar">
        <h3>1. 聲調練習句</h3>
        <p style="font-size:13px;">請對著麥克風讀出，觀察左側曲線：</p>
        <div class="sentence-box">「他拔起把柄。」<br><small>(Tā bá qǐ bà bǐng)</small></div>

        <h3>2. 曲線解讀指南</h3>
        
        <ul class="guide-list">
            <li><b>一聲 (55):</b> 曲線需維持在畫面上方的高位平滑線。</li>
            <li><b>二聲 (35):</b> 曲線應由中段平穩滑向高段。</li>
            <li><b>三聲 (214):</b> 曲線須有明顯的下探（壓低聲音）再反彈。</li>
            <li><b>四聲 (51):</b> 曲線應呈現近乎垂直的下降。</li>
        </ul>

        <div style="margin-top: 30px; font-size: 12px; color: #666; background: #222; padding: 10px; border-radius: 4px;">
            <b>💡 使用提示：</b><br>
            如果偵測不到聲音，請檢查瀏覽器地址列右側的麥克風圖示是否已允許存取 `bideyuanli.com`。
        </div>
    </aside>
</div>

</body>
</html>

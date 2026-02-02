<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>日常單詞隨機生成器 - 繁體注音版</title>
    <style>
        :root {
            --bg-color: #f7f9fc;
            --text-color: #2d3436;
            --accent-color: #0984e3;
            --ruby-color: #d63031;
        }

        body {
            background-color: var(--bg-color);
            color: var(--text-color);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            margin: 0;
            font-family: "標楷體", "DFKai-SB", serif;
        }

        .container {
            text-align: center;
            padding: 20px;
        }

        /* 台灣標準：注音在右側的關鍵 CSS */
        ruby {
            font-size: 6rem;
            ruby-position: ruby-text; /* 標準定位 */
            margin: 0 10px;
        }

        /* 針對支援直排注音的瀏覽器優化 */
        rt {
            font-size: 1.5rem;
            color: var(--ruby-color);
            writing-mode: vertical-rl; /* 強制注音直打 */
            text-orientation: upright;
            user-select: none;
        }

        .category-tag {
            font-size: 1.2rem;
            background: #dfe6e9;
            padding: 5px 15px;
            border-radius: 20px;
            margin-bottom: 20px;
            display: inline-block;
        }

        button {
            margin-top: 50px;
            padding: 15px 40px;
            font-size: 1.4rem;
            cursor: pointer;
            background-color: var(--accent-color);
            color: white;
            border: none;
            border-radius: 12px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.1);
            transition: 0.2s;
        }

        button:hover {
            background-color: #74b9ff;
            transform: translateY(-2px);
        }

        button:active {
            transform: translateY(0);
        }
    </style>
</head>
<body>

    <div class="container">
        <div id="category" class="category-tag">點擊下方按鈕</div>
        <div id="word-display">
            <ruby>歡迎<rt>ㄏㄨㄢㄧㄥˊ</rt></ruby>使用
        </div>
        <br>
        <button onclick="generate()">隨機換單詞</button>
    </div>

    <script>
        // 大量詞庫：[漢字陣列, 注音陣列, 分類]
        // 這樣分開寫，才能確保每個字對應到正確的右側注音
        const wordLib = [
            {w: ["蘋", "果"], z: ["ㄆㄧㄥˊ", "ㄍㄨㄛˇ"], c: "水果"},
            {w: ["香", "蕉"], z: ["ㄒㄧㄤ", "ㄐㄧㄠ"], c: "水果"},
            {w: ["西", "瓜"], z: ["ㄒㄧ", "ㄍㄨㄚ"], c: "水果"},
            {w: ["三", "明", "治"], z: ["ㄙㄢ", "ㄇㄧㄥˊ", "ㄓˋ"], c: "食物"},
            {w: ["漢", "堡"], z: ["ㄏㄢˋ", "ㄅㄠˇ"], c: "食物"},
            {w: ["腳", "踏", "車"], z: ["ㄐㄧㄠˇ", "ㄊㄚˋ", "ㄔㄜ"], c: "交通"},
            {w: ["捷", "運"], z: ["ㄐㄧㄝˊ", "ㄩㄣˋ"], c: "交通"},
            {w: ["鉛", "筆"], z: ["ㄑㄧㄢ", "ㄅㄧˇ"], c: "文具"},
            {w: ["橡", "皮", "擦"], z: ["ㄒㄧㄤˋ", "ㄆㄧˊ", "ㄘㄚ"], c: "文具"},
            {w: ["長", "頸", "鹿"], z: ["ㄔㄤˊ", "ㄐㄧㄥˇ", "ㄌㄨˋ"], c: "動物"},
            {w: ["老", "虎"], z: ["ㄌㄠˇ", "ㄏㄨˇ"], c: "動物"},
            {w: ["圖", "書", "館"], z: ["ㄊㄨˊ", "ㄕㄨ", "ㄍㄨㄢˇ"], c: "場所"},
            {w: ["游", "泳", "池"], z: ["ㄧㄡˊ", "ㄩㄥˇ", "ㄔˊ"], c: "場所"},
            {w: ["牙", "刷"], z: ["ㄧㄚˊ", "ㄕㄨㄚ"], c: "生活"},
            {w: ["吹", "風", "機"], z: ["ㄔㄨㄟ", "ㄈㄥ", "ㄐㄧ"], c: "生活"},
            {w: ["電視", "機"], z: ["ㄉㄧㄢˋ ㄕˋ", "ㄐㄧ"], c: "家電"}
            // ... 你可以仿照格式繼續往下增加
        ];

        function generate() {
            const display = document.getElementById('word-display');
            const cat = document.getElementById('category');
            
            const item = wordLib[Math.floor(Math.random() * wordLib.length)];
            
            let htmlContent = "";
            // 將每個字及其注音封裝在 ruby 中，實現一對一右側排列
            for (let i = 0; i < item.w.length; i++) {
                htmlContent += `<ruby>${item.w[i]}<rt>${item.z[i]}</rt></ruby>`;
            }
            
            display.innerHTML = htmlContent;
            cat.innerText = item.c;
        }
    </script>
</body>
</html>
